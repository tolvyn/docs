# Budgets

Budgets cap AI spend by team, service, agent, or organization. A budget has a scope, an amount in USD, a period, and one of three modes:

- **`soft`** — alert only; never blocks.
- **`hard`** — block at the proxy (HTTP 429) before the provider is called once the cap would be exceeded.
- **`approval`** — block at the cap like `hard`, but additionally record a pending approval request so an administrator can grant a bounded, time-boxed extension.

---

## Overview

A budget is a row in the `budgets` table with these key fields:

| Field | Description |
|---|---|
| `scope_type` | `organization`, `team`, `service`, or `agent` |
| `scope_id` | A UUID, and what it holds depends on the scope: the **team's id** for `team`; a UUID **derived from the name** for `service` and `agent` (you pass the name, TOLVYN derives it — see [Service and agent names](#service-and-agent-names)); NULL for `organization` |
| `amount_microdollars` | Limit in microdollars (int64) |
| `period` | `daily`, `weekly`, or `monthly` |
| `mode` | `soft`, `hard`, or `approval` |
| `current_spend_microdollars` | Running spend in this period |
| `period_start` / `period_end` | Inclusive bounds of the current period |
| `settings` | JSONB; stores `last_alerted_threshold` for dedup |

Multiple budgets can apply to a single request (e.g. an org budget plus a team budget plus an agent budget). All are tracked simultaneously; hard limits across *any* applicable budget block the request.

---

## Scope hierarchy

When the proxy receives a request, it resolves all budgets matching the request's tenant, team, service, and agent. The resolver returns them sorted **most-specific first**:

```
agent  >  service  >  team  >  organization
```

The resolver sorts applicable budgets by the explicit ordering agent (most specific) → service → team → organization.

A request with team `backend`, service `summarizer`, and agent `claude-code` is checked against:

- Any `agent` budget created for the agent name `claude-code`
- Any `service` budget created for the service name `summarizer`
- Any `team` budget created for the `backend` team
- Any `organization` budget (no scope)

For `service` and `agent`, TOLVYN matches on a UUID it derives from the name you
gave when you created the budget — you never handle that UUID yourself. See
[Service and agent names](#service-and-agent-names).

All four can exist simultaneously and all four spend counters tick on every request.

---

## Hard mode

When `mode = "hard"`, requests are rejected at the proxy **before the provider is called** once the budget would be exceeded.

### How the check works (reservation-based pre-pay)

TOLVYN **estimates the cost of each request before sending it to the provider**. Input tokens are estimated from the request body size; output tokens use the `max_tokens` field from the request body, or `1000` if not set.

Rather than comparing against the (lagging) recorded `current_spend`, the proxy **reserves** the estimate against the budget for the duration of the provider call, under a per-budget row lock. Concretely, for each hard budget it atomically:

1. locks the budget row (`SELECT … FOR UPDATE`) — this serializes concurrent requests against the same budget;
2. sums the in-flight reservations already held against that budget;
3. blocks if **`current_spend + reserved + this_estimate ≥ amount`**; otherwise inserts a reservation row and proceeds.

The reservation is **released when the request finishes** (success, error, or panic); a background sweeper reclaims any reservation orphaned by a process crash. This closes the concurrency window where many simultaneous requests could each pass against the same stale spend and collectively overshoot the cap.

If the budget check itself errors (a TOLVYN-side database failure), the request is **allowed through — TOLVYN fails open and never blocks a customer's AI call because of an internal error** (see [Fail-open](#fail-open)).

When a request would breach the cap, the proxy returns **HTTP 429 Too Many Requests** before contacting the provider:

```json
{
  "error": "budget_exceeded",
  "budget_id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope": "team",
  "limit_usd": "1000.00",
  "spent_usd": "990.20",
  "estimated_usd": "12.40",
  "message": "Budget limit would be exceeded by this request. Contact your administrator."
}
```

The `estimated_usd` field shows the pre-flight estimate that triggered the block. Because the estimate is a heuristic, actual post-request spend may differ slightly — exact spend is recorded by `meterAndRecord` after the response completes.

### What does NOT count

- The blocked request **never reaches the provider** — no upstream cost is incurred
- The blocked request **is not metered** — no row appears in the request log
- No tokens, no provider response time, no cost — the request exists only as a 429 to the client

---

## Soft mode

When `mode = "soft"`, the budget tracks spend and fires alerts at threshold crossings but **never blocks requests**. The proxy continues to forward requests to the provider after a soft budget is exceeded.

### Threshold levels

Threshold levels:

```go
var thresholds = []int{50, 75, 90, 100}
```

| Threshold | Severity |
|---|---|
| 50% | `info` |
| 75% | `warning` |
| 90% | `warning` |
| 100% | `critical` |

When a request pushes utilization past a threshold, an alert is inserted into the `alerts` table, an email is sent to the tenant's address (if SMTP is configured), and a webhook is dispatched (event type `alert.budget_threshold`).

### Dedup

Each budget tracks the last threshold it alerted for in its `settings.last_alerted_threshold` JSONB field. Once `90%` has fired, `50%`, `75%`, and `90%` are skipped for the rest of the period; only `100%` can still fire. The field resets to empty when the period rolls over.

---

## Approval mode (approve-and-wait)

When `mode = "approval"`, the budget enforces **exactly like `hard` at its cap** — the request is rejected with HTTP 429 and the provider is never called — but with one addition: the proxy records a **pending approval request** so an administrator can grant a bounded, time-boxed cap extension instead of editing the budget.

This is *not* a request queue. The blocked request is not stored or replayed; the caller receives a 429 and should retry later. An approval simply raises the effective cap for a while.

### What the proxy returns

Status: **HTTP 429 Too Many Requests**.

```json
{
  "error": "budget_approval_required",
  "status": "pending_approval",
  "budget_id": "e5f8a1b2-3c4d-5e6f-7a8b-9c0d1e2f3a4b",
  "scope": "team",
  "limit_usd": "1000.00",
  "estimated_usd": "12.40",
  "message": "Budget cap reached. An approval request has been created — an administrator must approve to continue."
}
```

### The flow

1. A request hits the cap on an `approval`-mode budget → **429** (provider not called).
2. The proxy records **one pending approval** for that budget and fires a single notification through the alerts channel. This is **deduplicated**: a burst of over-cap requests produces exactly one pending approval row and one alert, not a storm. (Recording the approval is best-effort — it never blocks or changes the 429.)
3. An administrator reviews pending approvals and either approves (with a bounded amount and time) or denies.
4. While an approval grant is active, the budget's **effective limit = `amount` + active grants**, computed atomically with the same reservation check, so requests flow again until the grant's amount or time runs out.

### Managing approvals (API)

| Endpoint | Purpose |
|---|---|
| `GET /v1/budget-approvals` | List approval requests (filter by status) |
| `POST /v1/budget-approvals/{id}/approve` | Grant a **bounded** extension |
| `POST /v1/budget-approvals/{id}/deny` | Deny the request |

Approve body: `{ "extra_usd": <float, required, > 0>, "duration_hours": <int, optional> }`.

- **`extra_usd`** is required and must be > 0 — an approval grants a bounded amount, never an open door.
- **`duration_hours`** defaults to **24** and is capped at **720 (30 days)** — a grant cannot be unbounded in time. The grant expires automatically at `now() + duration_hours`.

> **Setting `approval` mode:** create or edit an `approval`-mode budget via the **dashboard** or the API (`POST /v1/budgets` with `"mode": "approval"`). The CLI `tolvyn budgets create --mode` currently accepts `soft` and `hard` only — use the dashboard/API for approval mode until CLI support ships.

---

## Fail-open

Budget enforcement is **fail-open**: a TOLVYN-side error must never stop a customer's AI call.

- If the **plan/feature lookup** fails (so TOLVYN can't tell whether hard enforcement is enabled), the request is **allowed through** — the proxy does not hard-block on an unknown plan.
- If a **reservation/budget check** errors mid-flight, the proxy **logs and allows** the request rather than rejecting it.
- Only a genuine, successfully-evaluated breach produces a 429.

The same principle runs throughout the proxy hot path: when in doubt due to an internal failure, TOLVYN allows the call and records what it can — it is never a hard dependency that can take your AI offline. (Accounts with no subscription/legacy accounts still enforce budgets normally; only *errors* fail open.)

---

## Period types

| Period | Bounds (UTC) |
|---|---|
| `daily` | 00:00:00 → 23:59:59 of the same calendar day |
| `weekly` | Monday 00:00:00 → Sunday 23:59:59 |
| `monthly` | 1st of month 00:00:00 → last day of month 23:59:59 |

Period bounds are computed in UTC.

### Auto-reset

When the proxy resolves budgets for a request and finds one whose `period_end` is in the past, it resets that budget inline:

```sql
UPDATE budgets
SET    current_spend_microdollars = 0,
       period_start = <new start>,
       period_end   = <new end>,
       settings     = '{}'
WHERE  id = ?
  AND  period_end < now()
```

This means:

- A monthly budget rolls over at the start of each calendar month
- `current_spend_microdollars` resets to zero
- The `last_alerted_threshold` setting resets — so 50%/75%/90%/100% will fire again in the new period
- The reset is idempotent: if two requests arrive simultaneously, only the first wins the `UPDATE`; the second re-reads the updated row

The dashboard's budget list endpoint also resets expired periods so the UI reflects the current period even if no proxy request has triggered the reset yet.

---

## Creating a budget

### Dashboard

**Budgets → Create Budget**:

1. Pick a scope (Organization / Team / Service / Agent)
2. Enter the amount in USD
3. Pick a period (daily / weekly / monthly)
4. Pick a mode (soft / hard / approval)
5. Save

For a team scope, the dashboard provides a team picker. For a service or agent scope, you **type the name** — the same value your requests carry in `X-Tolvyn-Service` or `X-Tolvyn-Agent` (or that is bound to the API key). **It must match exactly**: the budget is matched on that name, so `checkout` and `Checkout` are two different services.

### API

`POST /v1/budgets` (see [API Reference](../reference/api.md#budgets) for the full schema):

```bash
curl -X POST https://api.tolvyn.io/v1/budgets \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_type":  "team",
    "scope_id":    "b3e2c8a4-1f6d-4a8c-9e0f-7d1c4b2a3e5f",
    "amount_usd":  1000,
    "period":      "monthly",
    "mode":        "hard"
  }'
```

### CLI

```bash
tolvyn budgets create \
  --scope team \
  --team backend \
  --amount 1000 \
  --period monthly \
  --mode hard
```

CLI flags:

| Flag | Default | Notes |
|---|---|---|
| `--scope` | `org` | `org` → `organization` server-side |
| `--team` | — | Required when `--scope=team` |
| `--service` | — | Required when `--scope=service` |
| `--agent` | — | Required when `--scope=agent` |
| `--amount` | — | Required, > 0 |
| `--period` | `monthly` | `monthly`, `weekly`, `daily` |
| `--mode` | `soft` | `soft` or `hard` |

---

## Service and agent names

Services and agents are identified by **a name you choose**, carried on each
request in `X-Tolvyn-Service` / `X-Tolvyn-Agent` (or the `service` / `agent` SDK
option, or bound to the API key). Neither is a record in TOLVYN — there is no
"create a service" step — so the name is the whole of the identity.

When you create a service or agent budget you pass **the name**: `service_name`
or `agent_name`. TOLVYN derives a deterministic UUID from it and stores that in
the `scope_id` column, which is UUID-typed. At resolution time the proxy reads
the header, derives the same UUID, and matches. **You never see or supply that
UUID**, and `scope_id` in a create request is ignored for these two scopes.

**THE NAME MUST MATCH EXACTLY.** The derivation is over the literal string, so
`checkout` and `Checkout` are two different services and a budget on one will
never match traffic tagged with the other. If a budget shows no spend while the
corresponding page under Services or Features does, a name mismatch is the first
thing to check.

The derivation, for anyone who wants to reproduce it:

```go
// The same derivation is used for a service name and an agent name.
// ScopeNameToUUID converts the name to a deterministic UUID v5
// (SHA-1, DNS namespace) so it can be stored in the UUID-typed scope_id column.
func ScopeNameToUUID(name string) string {
    // DNS namespace UUID: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
    ns := []byte{0x6b, 0xa7, 0xb8, 0x10, ...}
    h := sha1.New()
    h.Write(ns)
    h.Write([]byte(agentName))
    // ... format as UUIDv5
}
```

Properties:

- Same agent name + same namespace always yields the same UUID
- No collision-resolution table needed
- Two agents with the same name across different tenants share the same UUID, but they are separated by `tenant_id` in every query
- Agent names are case-sensitive — `Claude-Code` and `claude-code` are different agents

At budget-resolution time the proxy reads the header, runs the same derivation, and matches on the result. No reverse lookup is needed, and no name is stored in the matching column — the budget's display name is kept separately so the dashboard can show it back to you.

---

## Kill switch vs budget

| | Budget | Kill switch |
|---|---|---|
| Purpose | Spend cap with running counter | Unconditional block |
| What it tracks | `current_spend_microdollars` per period | Nothing — just a flag |
| When it blocks | `mode=hard` AND `spend >= amount` | Always (while active) |
| Order in proxy | Checked after the kill switch | First check |
| HTTP response | `429 budget_exceeded` | `451 kill_switch_active` |
| Auto-reset | Yes, on period rollover | No — must be deactivated manually |
| Use for | Long-term cost control | Incident response, runaway agents |

Both can be active simultaneously. Kill switches are checked first.

---

## See also

- [Alerts](alerts.md) — budget threshold alert delivery
- [Kill Switch](kill-switch.md) — emergency unconditional block
- [API Reference: Budgets](../reference/api.md#budgets)
- [CLI Reference: `tolvyn budgets`](../reference/cli.md#tolvyn-budgets)
