# End Customers

Know your AI cost-of-goods-sold per customer. If you sell AI features to other companies, attaching an end-customer identifier to every request gives you per-customer revenue/cost analysis, free-tier abuse detection, and audit-grade billing data.

---

## How it works

Set the `X-Tolvyn-End-Customer` header on every request. The value is whatever identifier you use for your customers internally — typically a tenant ID, account ID, or short slug.

**No registration step.** A new customer appears in the dashboard, CLI, and API automatically on its first tagged request — there is nothing to pre-register, in code or in the UI.

How you supply the value depends on whether one client serves one customer or many:

- **One client, many customers** (a typical web app or request handler) → set the header **per request**. See [Per-request attribution](#per-request-attribution-multi-tenant-apps) below — this is what most apps want.
- **One client, one customer** (a per-tenant worker or batch job) → set it once on the constructor, as shown here.

### Single-tenant clients (constructor field)

If a client instance serves exactly one customer for its entire lifetime, set `end_customer` once on the constructor and every request it makes is tagged automatically:

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    end_customer="acme-corp",
)
```

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  endCustomer: 'acme-corp',
});
```

```go
client := tolvynopenai.NewClient(tolvyn.ClientOptions{
    TolvynAPIKey: "tlv_live_...",
    EndCustomer:  "acme-corp",
})
```

This sets **one fixed value for the client's lifetime**. It is the wrong default for a request handler that serves many customers — every customer's calls would be tagged as the single value hardcoded here. For that case, use [per-request attribution](#per-request-attribution-multi-tenant-apps).

### Proxy mode / curl

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "X-Tolvyn-End-Customer: acme-corp" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
```

Combine with other attribution headers as needed — see [combining](#combining-with-team-insights) below.

---

## Per-request attribution (multi-tenant apps)

Most applications serve many customers through a single shared client. In that case, do **not** set `end_customer` on the constructor — pass the identifier on each request, read from your own user/session/tenant context. A new customer is attributed automatically on its first call, so one shared client can cover thousands of distinct customers with no per-customer setup.

**Python** — pass `extra_headers` on the call:

```python
from tolvyn import OpenAI

# One shared client — note: no end_customer here.
client = OpenAI(tolvyn_api_key="tlv_live_...", service="ticket-summarizer")

# Per request, inside your handler — the id comes from your own context:
resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}],
    extra_headers={"X-Tolvyn-End-Customer": current_user.account_id},
)
```

**Node** — pass per-request headers as the second argument:

```javascript
import { OpenAI } from 'tolvyn';

// One shared client — note: no endCustomer here.
const client = new OpenAI({ tolvynApiKey: 'tlv_live_...', service: 'ticket-summarizer' });

// Per request, inside your handler:
const resp = await client.chat.completions.create(
  { model: 'gpt-4o', messages: [{ role: 'user', content: prompt }] },
  { headers: { 'X-Tolvyn-End-Customer': req.user.accountId } },
);
```

### Don't set it in two places

Set the end-customer **either** on the constructor **or** per request — not both on the same client. A per-request header overrides the constructor default, but to stay safe use the exact name `X-Tolvyn-End-Customer`: a mismatched-case header (e.g. `x-tolvyn-end-customer`) can in some setups be sent *alongside* the constructor default as a duplicate, and the proxy reads only the first one — quietly attributing to the wrong value. The clean rule that sidesteps all of this: pick one place to set it. For multi-tenant apps, that's per request — leave the constructor field unset.

---

## What is tracked

Per-end-customer aggregation from `HandleUsageByEndCustomer` (`internal/api/client.go:2158`):

| Field | Type | Meaning |
|---|---|---|
| `customer` | string | The exact value sent in `X-Tolvyn-End-Customer` |
| `request_count` | int64 | Total proxied requests for this customer in the period |
| `total_cost_usd` | string | Sum of cost as USD |
| `avg_cost_usd` | string | Average cost per request |
| `model_count` | int64 | Distinct `model_id`s the customer triggered |
| `top_model` | string | Most-used model family |
| `first_seen` | RFC 3339 | Timestamp of the customer's first request in the window |
| `last_seen` | RFC 3339 | Timestamp of the customer's most recent request |

Default window is 30 days. Override with `?from=` / `?to=`. Top 100 customers by cost are returned. The list response also includes a grand total:

```json
{
  "customers": [ ... ],
  "total": 84,
  "total_cost_usd": "8412.30",
  "period_start": "...",
  "period_end": "..."
}
```

Filter customers above a cost threshold with `?min_cost_usd=10.00` (server-side `HAVING SUM(cost) >= ...`).

---

## Last-active indicator

The dashboard's **Last Active** column color-codes based on days since `last_seen` (from `dashboard-client/src/pages/CustomersDashboard.jsx:23-53`):

| Days since last request | Color | Label |
|---|---|---|
| 0 | Green | `today` |
| 1 – 7 | Green | `Nd ago` |
| 8 – 30 | Yellow | `Nd ago` |
| > 30 | Red | `Nd ago` |

Customers in the **red** zone (> 30 days idle) but still listed are either churned, on the verge of churn, or on a free tier with usage gaps. Worth flagging to your success team.

---

## Use cases

### 1. SaaS billing — usage-based pricing

Every customer that makes a request shows up in `GET /v1/usage/by-end-customer/{customer}` with exact token counts and microdollar cost. Compute customer-level margin:

```
Customer margin = MRR(customer) − (AI cost × markup) − fixed overhead
```

Pull the AI cost monthly via the API, multiply by your overhead multiplier, subtract from MRR. If margin goes negative on a customer, you have a costed-out free tier or an unprofitable usage pattern to investigate.

### 2. Free-tier abuse detection

Sort customers descending by `total_cost_usd` and filter to those on your free plan. Customers in the top 10% of AI consumption who pay nothing are candidates for either:

- A polite "you'd benefit from the paid plan" outreach
- Rate-limiting at the application level
- A per-customer hard budget (see [Budgets](budgets.md))

You can wire a hard budget against an end-customer scope to block AI for an abusive free user. The proxy will return `429 budget_exceeded` for that customer until the budget resets.

### 3. Enterprise customer reporting

Some enterprise customers contractually require monthly AI usage reports. Export `GET /v1/usage/by-end-customer/{customer}?from=...&to=...` as JSON, attach to the invoice or send as a stand-alone report. The detail endpoint includes per-model breakdown which often satisfies compliance or audit requirements directly.

---

## CLI

```bash
tolvyn cost --by customer
tolvyn cost --by customer --from 2026-05-01 --to 2026-05-17
```

Output:

```
Customer                       Requests    Total Cost    Avg Cost    Top Model               Last Active
acme-corp                      18,432      $1,240.50     $0.0673     gpt-4o                  today
beta-industries                12,184      $812.30       $0.0666     gpt-4o                  2d ago
gamma-llc                      4,201       $241.10       $0.0574     gpt-4o-mini             18d ago
delta-startup                  102         $4.20         $0.0411     gpt-4o-mini             42d ago

Total AI Spend:      $2,298.10
```

---

## API

### `GET /v1/usage/by-end-customer`

| Param | Default | Description |
|---|---|---|
| `from` | `now - 30 days` | RFC 3339 or `YYYY-MM-DD` |
| `to` | `now` | RFC 3339 or `YYYY-MM-DD` |
| `min_cost_usd` | — | Server-side cost floor (decimal USD) |

Returns top 100 customers by cost, with the response shape above.

### `GET /v1/usage/by-end-customer/{customer}`

`{customer}` is URL-encoded. Returns detail similar to the per-user detail endpoint: total spend, retry rate, model breakdown, daily spend (last 7 days), top services.

---

## Combining with Team Insights

`X-Tolvyn-User` and `X-Tolvyn-End-Customer` answer different questions:

| Header | Question it answers |
|---|---|
| `X-Tolvyn-User` | *Which of my employees made this request?* |
| `X-Tolvyn-End-Customer` | *Which of my customers is paying for the underlying AI?* |

Both can be set on the same request — common in SaaS support tooling or AI-powered customer ops, where an internal user works on behalf of an external customer.

```python
# Static attribution on the client; the variable parts per request.
client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    team="support",
    service="ticket-summarizer",
)

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}],
    extra_headers={
        "X-Tolvyn-User": agent.email,                 # your employee handling the request
        "X-Tolvyn-End-Customer": ticket.account_id,   # your customer they're serving
    },
)
```

The dashboard's request log shows both fields. Per-user, per-customer, and per-team aggregations are independent.

---

## Empty state

If the customers page or `GET /v1/usage/by-end-customer` is empty:

1. **Add the header.** Update your code so every customer-attributed request includes `X-Tolvyn-End-Customer`. Existing requests from before you added the header will not be retroactively attributed.
2. **Check the value is non-empty.** The query filters `tolvyn_end_customer IS NOT NULL AND tolvyn_end_customer != ''`.
3. **Check date range.** Default lookback is 30 days. Customers with no traffic in that window won't appear even if you sent attribution headers earlier.
4. **Verify in raw requests.** `tolvyn requests --limit 5` shows recent metadata — if the `tolvyn_end_customer` column is blank, the header isn't reaching the proxy.

---

## Choosing the identifier value

The value is stored and displayed verbatim (see [Privacy](#privacy)) — it's the key you see in the dashboard, the CLI, and `GET /v1/usage/by-end-customer`, so choose it deliberately:

- **Opaque id** (a UUID or numeric account ID from your auth system) — stable across customer renames and reveals nothing about who the customer is, but is unreadable at a glance in the dashboard.
- **Readable slug or name** (e.g. `acme-corp`) — friendly in reports and the dashboard, but can change (a rename starts a new, separate record) and exposes your customer list to anyone with dashboard access.

A common middle ground: send the stable opaque id and map it to a display name inside your own tooling. Whichever you pick, send it **consistently** — the id *is* the customer key, so changing it for an existing customer splits their history into two records.

## Privacy

The `X-Tolvyn-End-Customer` value is stored as you sent it. If you send `acme-corp`, that's what TOLVYN stores. If your customers' identifiers are PII (e.g. their company email), prefer an opaque internal ID (a UUID or numeric ID from your auth system) that maps back to the customer only inside your own systems.

TOLVYN never reads prompt content. Customer identifiers are the only customer data persisted.

---

## See also

- [Team Insights](team-insights.md) — for per-employee attribution
- [Budgets](budgets.md) — set hard caps per end-customer
- [Agents](agents.md) — for per-agent budgets and attribution
- [API Reference: Usage](../reference/api.md#usage-and-analytics)
- [CLI Reference: `tolvyn cost`](../reference/cli.md#tolvyn-cost)
