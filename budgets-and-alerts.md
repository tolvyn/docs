# Budgets & Alerts

Control and monitor AI spending with budgets, threshold alerts, and anomaly detection.

© 2026 TOLVYN

---

## 1. How Budgets Work

Every request through the TOLVYN proxy is checked against all applicable budgets before being forwarded to the provider. Budget resolution (from `resolver.go`) proceeds as follows:

1. All budgets for the tenant are fetched — including `organization` scope and `team` scope budgets matching the request's team.
2. Any budget whose period has expired is automatically reset (spend zeroed, new period calculated) before evaluation.
3. Budgets are ordered most-specific first: `service > team > organization`.
4. **Hard-mode** budgets are enforced: if `current_spend >= amount`, the request is rejected with HTTP 429 before reaching the provider.
5. **Soft-mode** budgets track spend and fire alerts but never block requests.

After a request completes, the cost is deducted from all applicable budgets in a single database transaction, atomically with the ledger record and alert checks.

---

## 2. Budget Scopes

| Scope | Description | Matches |
|---|---|---|
| `organization` | Covers all requests for your account | All requests |
| `team` | Covers requests tagged with a specific team | Requests with matching `X-Tolvyn-Team` header |
| `service` | Reserved for future use <!-- verify --> | — |

Multiple budgets can be active simultaneously. A single request is checked against and charged to all applicable budgets.

---

## 3. Budget Modes

### Hard Mode

When a hard-mode budget is exceeded, the proxy returns **HTTP 429** immediately without forwarding the request to the provider:

```json
{
  "error": "budget_exceeded",
  "message": "Hard budget exceeded for scope team/..."
}
```

The provider is never called. No tokens are consumed. No cost is incurred.

### Soft Mode

When a soft-mode budget is exceeded, requests continue to succeed. Threshold alerts are fired (see section 5). The proxy logs the soft limit but does not block.

---

## 4. Creating Budgets

### CLI

```bash
# Organization monthly soft budget
tolvyn budgets create --scope org --amount 500 --period monthly --mode soft

# Team monthly hard budget
tolvyn budgets create --scope team --team platform --amount 200 --period monthly --mode hard

# Daily hard budget
tolvyn budgets create --scope org --amount 50 --period daily --mode hard
```

### API

```bash
curl -X POST https://api.tolvyn.io/v1/budgets \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_type": "team",
    "scope_id": "<team-uuid>",
    "amount_usd": 200.00,
    "period": "monthly",
    "mode": "hard"
  }'
```

### Dashboard

Create and manage budgets from [app.tolvyn.io](https://app.tolvyn.io) → Budgets.

---

## 5. Threshold Alerts

TOLVYN automatically fires alerts when budget utilization crosses these thresholds (from `threshold.go`):

| Threshold | Severity |
|---|---|
| 50% | info |
| 75% | warning |
| 90% | warning |
| 100% | critical |

Each threshold is fired **once per budget period**. If the budget is reset (period rolls over), thresholds reset and can fire again.

Alert records are stored in the `alerts` table with `alert_type = 'budget_threshold'` and can be retrieved via `GET /v1/alerts`. An email notification is also sent to the account's registered email address.

Example alert title: `Budget organization at 75% utilization`

---

## 6. Anomaly Detection

TOLVYN detects unusually expensive individual requests using a rolling baseline (from `anomaly.go`):

- The **last 100 requests** for the tenant (filtered by service if a service tag is present) are used to compute an average cost.
- If the **current request costs 10× or more** the average, and there are **at least 10 prior requests** in the baseline, a `cost_anomaly` alert is fired.
- The alert includes: current cost, average cost, multiplier, model ID, service name, sample count, and request ID.

```json
{
  "alert_type": "cost_anomaly",
  "severity": "critical",
  "title": "Cost anomaly: chat-api — request cost 14x above average",
  "metadata": {
    "current_cost": 14200000,
    "avg_cost": 985000,
    "multiplier": 14,
    "model_id": "gpt-4o",
    "service_name": "chat-api",
    "sample_count": 87,
    "request_id": "..."
  }
}
```

Anomaly detection runs **after** the request is committed, in a background goroutine, so it never affects request latency.

---

## 7. Alert Types

| `alert_type` | Triggered by |
|---|---|
| `budget_threshold` | Budget utilization crossing 50%, 75%, 90%, or 100% |
| `cost_anomaly` | Single request costing 10x above the rolling average |

Retrieve alerts via:

```bash
curl https://api.tolvyn.io/v1/alerts \
  -H "Authorization: Bearer <token>"
```

Acknowledge an alert:

```bash
curl -X POST https://api.tolvyn.io/v1/alerts/<id>/acknowledge \
  -H "Authorization: Bearer <token>"
```

---

## 8. Budget Period Reset

Periods reset automatically at the start of each new period:

| Period | Reset schedule |
|---|---|
| `daily` | 00:00:00 UTC each day |
| `weekly` | Monday 00:00:00 UTC; period ends Sunday 23:59:59 UTC |
| `monthly` | 1st of each month 00:00:00 UTC; period ends last day 23:59:59 UTC |

Reset happens lazily — the period is recalculated the first time the budget is evaluated after expiry (during a request or a `ResolveBudgets` call). On reset: `current_spend` is zeroed and alert state (`last_alerted_threshold`) is cleared, allowing thresholds to fire again in the new period.

---

## 9. Kill Switch

Block all AI requests for a team immediately:

```bash
tolvyn kill --team <team-name>
```

This creates a `$0.000001` hard monthly budget for the team. Since any real request will immediately exceed this limit, all requests from that team receive HTTP 429.

The budget ID is printed; delete it to restore access:

```bash
# Via API
curl -X DELETE https://api.tolvyn.io/v1/budgets/<budget-id> \
  -H "Authorization: Bearer <token>"
```

See [CLI Reference — tolvyn kill](cli-reference.md#tolvyn-kill) for full details.

---

## 10. Viewing Budgets

```bash
tolvyn budgets list
```

```
SCOPE                  MODE    LIMIT         SPENT         UTIL       PERIOD
organization           soft    $500.00       $142.83       28.6%      monthly
team (platform-uuid)   hard    $200.00       $89.32        44.7%      monthly
```

Utilization is color-coded: green below 75%, yellow 75–90%, red 90%+.

Via API:

```bash
curl https://api.tolvyn.io/v1/budgets \
  -H "Authorization: Bearer <token>"
```

Individual budget:

```bash
curl https://api.tolvyn.io/v1/budgets/<id> \
  -H "Authorization: Bearer <token>"
```
