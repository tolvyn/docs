# Spend Quotas

Spend quotas are an **alert-tier** control: they track spend along a dimension and **fire alerts** as it approaches a limit — they **never block requests**. They are the non-enforcing sibling of [budgets](budgets.md).

## Quotas vs budgets

Use a **budget** when you want to *cap* spend (it can reject requests). Use a **spend quota** when you want *visibility* — to be notified as spend climbs, without ever risking a blocked request.

| | Spend quota | Budget |
|---|---|---|
| Blocks requests? | **Never** — alert only | Yes, in `hard`/`approval` mode (HTTP 429) |
| What it does at the limit | Fires an alert | Rejects the request before the provider is called |
| Dimensions | model, team, service, end_customer, tenant_total | organization, team, service, agent |
| Extra feature | **Burn-rate forecast** | Approve-and-wait grants |
| Best for | Cost awareness, early warning, forecasting | Hard cost control |

Quotas and budgets are independent — you can run both on the same spend.

---

## Dimensions

A quota measures spend along exactly one dimension:

| `dimension_type` | Measures | `dimension_value` |
|---|---|---|
| `model` | Spend on a specific model | the model id (e.g. `claude-sonnet-4-6`) |
| `team` | Spend by a team | the team id |
| `service` | Spend by a service | the service name |
| `end_customer` | Spend attributed to one of your end customers | the end-customer id you send in `X-Tolvyn-End-Customer` |
| `tenant_total` | All spend for your account | *(empty — no value)* |

---

## Periods

Quotas reset on a fixed period (UTC):

| `period` | Window |
|---|---|
| `day` | calendar day |
| `week` | Monday → Sunday |
| `month` | 1st → last day of the month |

---

## Alert thresholds

`alert_thresholds` is a list of utilization percentages. An alert fires as spend crosses each one within the period.

- Values are integers in the range **1–1000** — so over-100% thresholds are allowed (e.g. `150` to alert at 1.5× the limit).
- A typical set is `[50, 80, 100]`.
- Alerts are delivered through the usual [alerts](alerts.md) channel (in-app, email, webhook).

---

## Burn-rate forecast

Each quota has a forecast endpoint that projects whether you are on track to hit the limit before the period resets, based on recent spend.

`GET /v1/spend-quotas/{id}/forecast`

> **This is an estimate, not billing truth.** The forecast is a **linear extrapolation** of spend over a trailing window, and the spend it is built from comes from metering that can under-report in some cases (long-context, cached, and batch usage). Treat it as an early-warning signal, not an exact prediction. See the metering-accuracy notes for the caveats.

### Forecast states

| `state` | Meaning |
|---|---|
| `on_track` | At the current burn rate, the cap is not projected to be hit before the period resets |
| `projected_exhaustion` | The cap is projected to be hit before the period ends — includes `projected_exhaustion` (timestamp) and `days_remaining` |
| `over_limit` | Already over the limit this period |
| `insufficient_data` | Not enough history (or no positive limit) to forecast yet |

### Response shape

```json
{
  "quota": { "id": "…", "dimension_type": "tenant_total", "period": "month", "limit_usd": "5000.0000", "...": "…" },
  "consumption": {
    "period_start": "2026-06-01T00:00:00Z",
    "period_end":   "2026-06-30T23:59:59Z",
    "spent_microdollars": 1320000000,
    "spent_usd": "1320.0000",
    "utilization_pct": 26.4
  },
  "forecast": {
    "state": "projected_exhaustion",
    "burn_rate_per_day_microdollars": 88000000,
    "burn_rate_per_day_usd": "88.0000",
    "window_days": 7,
    "is_estimate": true,
    "method": "linear_extrapolation",
    "note": "at ~$88.0000/day, projected to hit the $5000.0000 cap on 2026-06-26 14:00 UTC (~14.8 days) — estimate, linear extrapolation",
    "disclaimer": "estimate from linear extrapolation; spend derived from metering that may under-report",
    "projected_exhaustion": "2026-06-26T14:00:00Z",
    "days_remaining": 14.8
  }
}
```

`projected_exhaustion` and `days_remaining` appear only when `state = "projected_exhaustion"`.

---

## Managing spend quotas

### Dashboard

The **Spend Quotas** page lists your quotas with current utilization; the burn-rate forecast for each row is fetched lazily when you expand it.

### API

| Endpoint | Purpose |
|---|---|
| `POST /v1/spend-quotas` | Create a quota |
| `GET /v1/spend-quotas` | List quotas |
| `GET /v1/spend-quotas/{id}` | Get one quota |
| `PUT /v1/spend-quotas/{id}` | Update a quota |
| `DELETE /v1/spend-quotas/{id}` | Delete a quota |
| `GET /v1/spend-quotas/{id}/forecast` | Burn-rate forecast |

See the [API Reference](../reference/api.md#spend-quotas) for request/response details.

---

## See also

- [Budgets](budgets.md) — enforce a hard spend cap (can block)
- [Alerts](alerts.md) — how threshold alerts are delivered
- [API Reference: Spend Quotas](../reference/api.md#spend-quotas)
