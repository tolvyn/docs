# How TOLVYN Works

A technical overview of TOLVYN's architecture, request pipeline, and guarantees.

---

## 1. Architecture

```
Your Application
      │
      │  Bearer tlv_live_...
      ▼
┌─────────────────────────────────────────────┐
│              TOLVYN Proxy                    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Auth &   │  │  Budget  │  │ Metering │   │
│  │ Key Mgmt │→ │ & Kill   │→ │ & Ledger │   │
│  └──────────┘  └──────────┘  └──────────┘   │
│                                    │         │
│              Tail Hub ◄────────────┘         │
└─────────────────────────────────────────────┘
      │
      │  Bearer <your-provider-key>   (swapped by proxy)
      ▼
Provider API (OpenAI / Anthropic / Google / DeepSeek)
      │
      ▼
Response returned to your application unchanged
```

The proxy runs at `proxy.tolvyn.io`. The management API runs at `api.tolvyn.io`. The dashboard is at `app.tolvyn.io`.

---

## 2. Request pipeline

Every request through the TOLVYN proxy follows these steps:

1. **Extract bearer token** — the `Authorization: Bearer tlv_live_...` header is read (DeepSeek/Google clients may send it as `x-api-key` / `x-goog-api-key`; all are accepted).
2. **Key lookup** — the token is verified and the resolved identity (tenant, team, service) is briefly cached.
3. **Kill-switch gate** — if a kill switch matches the request's scope, the proxy returns **HTTP 451** immediately, before anything else.
4. **Provider credential resolution** — the tenant's encrypted provider key for the target provider is fetched and decrypted in memory.
5. **Budget enforcement** — applicable budgets are resolved (most-specific first: agent → service → team → organization), expired periods auto-reset, and `hard`/`approval` budgets are checked via a concurrency-safe reservation (see [Budgets](../features/budgets.md)). A breach returns **HTTP 429** before the provider is called; `soft` budgets only alert.
6. **Build outbound request** — the `Authorization` header is replaced with your provider key; TOLVYN attribution headers are consumed (not forwarded to the provider).
7. **Forward to provider** — the request is sent upstream; time-to-first-token and total latency are measured.
8. **Meter and record** (background, fail-open) — token counts come from the provider's usage report, cost is computed, the request + ledger + budget spend + threshold alerts are written in one transaction, anomaly detection runs after, and a live event is published for `tolvyn tail`.
9. **Return response** — the provider's response reaches your application unchanged.

**Fail-open guarantee:** once the request is dispatched (step 7), any internal TOLVYN error is logged only — the provider response always reaches the caller. See [§6](#6-fail-open-behavior).

---

## 3. Provider keys vs TOLVYN keys

| Key type | Format | Purpose |
|---|---|---|
| TOLVYN API key | `tlv_live_...` | Authenticates your app to the proxy |
| Provider key | OpenAI / Anthropic / Google / DeepSeek format | Stored encrypted; used by the proxy to call the provider |

Your provider key is encrypted at rest (AES-256-GCM, per-tenant data-encryption key) and is **never returned by the API after creation**. The proxy decrypts it only in memory, at request time. Your application code only ever holds the TOLVYN key.

---

## 4. The immutable ledger

Every metered request is appended to a tamper-evident, hash-chained ledger:

- **Monotonic sequence numbers** per tenant — gaps indicate tampering.
- **SHA-256 hash chain** — each record commits to the prior record's hash.
- **Record hash** over the canonical record payload (tenant, sequence, request id, cost, provider, model, tokens, budget/enforcement status, …).
- **HMAC signature** over each record hash, using a server-side secret.

Verify the chain any time via `GET /v1/ledger/verify`. Costs are stored in **microdollars** (1 USD = 1,000,000 µ$) to avoid floating-point error — see [Metering Accuracy](../features/metering-accuracy.md). Full details: [Immutable Ledger](../features/ledger.md).

---

## 5. Attribution hierarchy

Requests can be tagged with attribution headers, used for budget scoping, usage analytics, and anomaly detection:

| Header | Purpose |
|---|---|
| `X-Tolvyn-Team` | Engineering team or cost center |
| `X-Tolvyn-Service` | Microservice or application name |
| `X-Tolvyn-Feature` | Feature or product area |
| `X-Tolvyn-Agent` | AI agent identifier |
| `X-Tolvyn-User` | End user within your app |
| `X-Tolvyn-End-Customer` | Your customer (for SaaS cost pass-through) |

---

## 6. Fail-open behavior

Both SDKs ship with fail-open enabled by default (`fail_open=True` / `failOpen: true`).

When fail-open is active and the TOLVYN proxy is unreachable (connection/DNS error or HTTP 503), the SDK retries the request **directly against the provider** using your provider key. The request succeeds but is **not** metered, governed, or logged by TOLVYN for that call.

- HTTP **4xx** responses from the proxy (e.g. 401, 429, 451) are **not** retried — they are intentional governance decisions, not outages.
- **Proxy mode** (no SDK) has no automatic fallback — if the proxy is unreachable, the call fails. Add your own retry or use SDK mode for critical paths.

Budget/enforcement checks are themselves fail-open: a TOLVYN-side error never blocks your AI call (see [Budgets → Fail-open](../features/budgets.md#fail-open)).

---

## 7. Tenant isolation & security

- **Database-level isolation.** Tenant data is isolated at the database layer using PostgreSQL row-level security (enforced in production), on top of application-level tenant scoping — so a query for one tenant cannot read another tenant's rows.
- **Provider keys encrypted at rest** (AES-256-GCM, per-tenant key); never returned after creation.
- **No prompt or response storage** (see §8).

---

## 8. What TOLVYN does NOT do

- **No prompt storage** — request bodies are never stored or logged.
- **No response storage** — provider responses stream through and are not retained.
- **No model hosting** — TOLVYN is a proxy and control plane, not a model provider.
- **No content filtering** — TOLVYN does not inspect or modify prompt/completion content.
- **No PII processing** — attribution tags are metadata only.

---

## 9. Plans

| Plan | Included requests / month | Overage |
|---|---|---|
| Free | 10,000 | None (blocks at the limit) |
| Starter | 250,000 | None (blocks at the limit) |
| Growth | 2,000,000 | Charged per request beyond the included amount |
| Scale | 10,000,000 | Charged per request beyond the included amount |
| Enterprise | Effectively unlimited | Custom |

The Free plan needs no credit card. See [app.tolvyn.io](https://app.tolvyn.io) for current pricing and to upgrade.
