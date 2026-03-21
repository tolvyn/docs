# How TOLVYN Works

A technical overview of TOLVYN's architecture, request pipeline, and guarantees.

© 2026 TOLVYN

---

## 1. Architecture

```
Your Application
      │
      │  Bearer tlv_live_...
      ▼
┌─────────────────────────────────────────────┐
│              TOLVYN Proxy                   │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Auth &   │  │  Budget  │  │ Metering │  │
│  │ Key Mgmt │→ │ Resolver │→ │ & Ledger │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                    │        │
│              Tail Hub ◄────────────┘        │
└─────────────────────────────────────────────┘
      │
      │  Bearer <your-provider-key>   (swapped by proxy)
      ▼
Provider API (OpenAI / Anthropic / Google)
      │
      ▼
Response returned to your application unchanged
```

The proxy runs at `proxy.tolvyn.io`. The management API runs at `api.tolvyn.io`. The dashboard is at `app.tolvyn.io`.

---

## 2. Request Pipeline

Every request through the TOLVYN proxy follows these steps (from `proxy.go`):

1. **Extract Bearer token** — the `Authorization: Bearer tlv_live_...` header is read from the incoming request.
2. **Key lookup** — the token prefix is used to find the matching row in `api_keys`, then the full token is verified with bcrypt. Resolved identity (tenant, team, service) is cached for 60 seconds.
3. **Provider credential resolution** — the tenant's encrypted provider key is fetched and decrypted for the target provider (OpenAI or Anthropic).
4. **Budget resolution** — `ResolveBudgets` returns all applicable budgets ordered most-specific first (service → team → organization). Expired periods are auto-reset before evaluation.
5. **Hard-budget enforcement** — if the request would exceed any hard-mode budget's current spend, the proxy returns HTTP 429 immediately. Soft-mode budgets are noted but do not block.
6. **Build outbound request** — the `Authorization` header is replaced with the provider key. Attribution headers (`X-Tolvyn-Team`, `X-Tolvyn-Service`, `X-Tolvyn-Feature`, `X-Tolvyn-Agent`) are propagated.
7. **Forward to provider** — the request is sent to the provider API. Time-to-first-token (TTFT) and total latency are measured.
8. **Meter and record** (background goroutine, fail-open):
   - Token count is extracted from the provider response; cost is calculated from the pricing table.
   - A single DB transaction: `INSERT` request row → `AppendRecord` ledger → `UPDATE` budget spends → `CheckAndFireThresholdAlerts`.
   - After commit: `CheckCostAnomaly` runs asynchronously.
   - Live event is published to the Tail Hub for `tolvyn tail` subscribers.
9. **Return provider response** — the provider's response body is returned to the caller unchanged.

**Fail-open guarantee:** once the provider request is dispatched (step 7), any internal TOLVYN error is logged only. The provider response always reaches the caller.

---

## 3. Provider Keys vs TOLVYN Keys

| Key type | Format | Purpose |
|---|---|---|
| TOLVYN API key | `tlv_live_...` (production) | Authenticates your app to the proxy |
| Provider key | OpenAI / Anthropic / Google format | Stored encrypted; used by the proxy to call the provider |

Your provider key is stored encrypted (AES-GCM) in the database. It is never returned by the API after creation. The proxy decrypts it at request time using a server-side key encryption key (DEK).

Key prefixes are stored in `api_keys.key_prefix` (12 chars). The full token is verified with bcrypt on each request (or from the 60-second in-memory cache).

---

## 4. The Immutable Ledger

Every metered request produces a ledger entry in `ledger_records`. The chain is designed to be tamper-evident:

- **Sequence numbers** are monotonically increasing per tenant. Gaps indicate tampering.
- **SHA-256 hash chain** — each record includes `previous_hash` (the `record_hash` of the prior record). The very first record's `previous_hash` is `SHA-256(tenant_id)` (the genesis hash).
- **Record hash** — `SHA-256(canonical JSON of record payload)`. The payload includes: `tenant_id`, `sequence_number`, `previous_hash`, `request_id`, `cost_microdollars`, `hierarchy_path`, `provider`, `model_id`, `model_family`, `modality`, `tokens_input`, `tokens_output`, `budget_status`, `enforcement_action`.
- **HMAC-SHA256 signature** — `HMAC-SHA256(record_hash, TOLVYN_HMAC_SECRET)` computed with the server-side secret.

The chain can be verified at any time via `GET /v1/ledger/verify`.

Costs are stored in **microdollars** (1 USD = 1,000,000 µ$) to avoid floating-point rounding errors.

---

## 5. Attribution Hierarchy

Every request can be tagged with up to four levels of attribution. These are sent as HTTP headers and stored in the `requests` table:

| Header | DB column | Purpose |
|---|---|---|
| `X-Tolvyn-Team` | `team_id` | Engineering team or cost center |
| `X-Tolvyn-Service` | `service_name` | Microservice or application name |
| `X-Tolvyn-Feature` | `feature_tag` | Feature or product area |
| `X-Tolvyn-Agent` | `agent_name` | AI agent identifier |

Attribution is used for budget scoping, usage analytics (`/v1/usage/by-team`, `/v1/usage/by-service`), and cost anomaly detection.

---

## 6. Fail-Open Behavior

Both SDKs ship with fail-open enabled by default (`fail_open=True` / `failOpen: true`).

When fail-open is active and the TOLVYN proxy is unreachable (connection error, DNS failure) or returns HTTP 503, the SDK automatically retries the request directly against the provider API using your `openai_api_key` / `anthropic_api_key`. A warning is printed to stderr:

```
TOLVYN proxy unreachable — routing direct to OpenAI (fail-open)
```

The request succeeds but is **not** metered, governed, or logged by TOLVYN.

HTTP 4xx errors from the proxy (e.g. 401, 429) are **not** retried — they represent intentional governance decisions.

To disable fail-open: `OpenAI(tolvyn_api_key=..., fail_open=False)`.

---

## 7. Budget Enforcement

Budget resolution (from `resolver.go`) works as follows:

- Budgets are resolved per-request for the tenant + team combination.
- Applicable budgets include: `scope_type = 'organization'` and `scope_type = 'team'` where `scope_id` matches the request's team.
- Budgets are ordered most-specific first: `service > team > organization`.
- If any **hard-mode** budget has `current_spend >= amount`, the proxy returns HTTP 429 with body `{"error":"budget_exceeded","message":"..."}` before forwarding to the provider.
- **Soft-mode** budgets trigger alerts at threshold crossings but never block requests.
- Expired budget periods are auto-reset (spend zeroed, new period calculated) at resolution time.

Budget periods:
- `daily` — resets at 00:00:00 UTC each day
- `weekly` — resets Monday 00:00:00 UTC, ends Sunday 23:59:59 UTC
- `monthly` — resets on the 1st of each month, ends on the last day

---

## 8. What TOLVYN Does NOT Do

- **No prompt storage** — request bodies are never stored or logged by TOLVYN.
- **No response storage** — provider responses are streamed through and not retained.
- **No model hosting** — TOLVYN is a proxy and control plane, not a model provider.
- **No content filtering** — TOLVYN does not inspect or modify prompt or completion content.
- **No PII processing** — attribution tags are metadata only; TOLVYN does not see message content.

---

## 9. Plans

| Plan | Included Requests | Overage |
|---|---|---|
| Trial | 1,000 | None (no overage) |
| Starter | 100,000 | None (no overage) |
| Team | 1,000,000 | $0.000100 per request |
| Pro | 5,000,000 | $0.000050 per request |
| Enterprise | Unlimited | $0.000025 per request |

Upgrade your plan from [app.tolvyn.io](https://app.tolvyn.io) or via the operator API.
