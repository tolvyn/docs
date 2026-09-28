# Savings Dashboard

The savings dashboard surfaces cost-reduction opportunities derived from your last 30 days of TOLVYN-metered traffic. Analysis runs nightly. **TOLVYN never auto-routes or modifies requests** — every finding is a suggestion you act on yourself.

---

## Overview

| Property | Value |
|---|---|
| Analysis schedule | Daily at **02:00 UTC** |
| Window | Rolling **30 days** ending at run time |
| Output | Findings inserted into `savings_findings`, surfaced on the dashboard |
| Action taken | None — TOLVYN does not change your routing |
| Atomic replace | Yes — non-dismissed findings deleted before each rerun |
| Dismissed findings | Preserved forever — survive subsequent runs |


---

## When analysis runs

A background goroutine started at server boot:

```go
now := time.Now().UTC()
next := time.Date(now.Year(), now.Month(), now.Day(), 2, 0, 0, 0, time.UTC)
if !next.After(now) {
    next = next.Add(24 * time.Hour)
}
log.Printf("savings: next analysis scheduled for %s", next.Format(time.RFC3339))
time.Sleep(time.Until(next))
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Minute)
RunAllTenants(ctx, db)
```

The job runs at **02:00 UTC** every day with a **30-minute timeout**. Each active tenant is analyzed sequentially.

To trigger an immediate recompute outside the schedule, operators can call `POST /v1/operator/savings/recompute`.

---

## Four savings rules

All four run in order: `analyzeSmallTokenRequests`, `analyzeDuplicatePrompts`, `analyzeUnderutilizedCache`, `analyzeIdleModels`. Findings from all rules are accumulated and inserted atomically.

### 1. `small_token_requests` — downgrade expensive models on short prompts


**Trigger criteria:**

| Criterion | Value |
|---|---|
| Token count (input + output) | < 500 |
| Token count | > 0 (excludes failed requests) |
| Request count for `(model_family, service_name)` pair | > 100 |
| Estimated monthly savings | ≥ $1.00 |
| Model must have a substitute | see below |
| Substitute must be currently priced | see below |
| Period must contain no cached tokens | see below |

#### How the figure is derived

**The dollar figure is computed from the substitute's current rates, not from a stored percentage.** When a finding is written, TOLVYN looks up the substitute's input and output rates in the same pricing table your own requests are billed from, and computes:

```
optimized cost = (input tokens × input rate) + (output tokens × output rate)
savings        = what you actually paid − optimized cost
```

The token counts are the real ones from the period. "What you actually paid" is the recorded cost of those requests, not a re-derivation — so the comparison is against your invoice rather than against a model of it.

**The percentage you see is a consequence of those two numbers, not an input to them.** If a provider changes a price, the next night's finding changes with it. Nothing is carried in TOLVYN's source code that could go stale.

#### When you will NOT see a downgrade finding

The rule is deliberately silent rather than approximate. **No finding is produced when:**

| Condition | Why |
|---|---|
| The substitute has no row in the pricing table | Any figure would be invented. This is also what stops TOLVYN recommending a model that has been withdrawn upstream. |
| The substitute is marked deprecated | A model TOLVYN has marked retired is never recommended, from the moment it is marked. |
| The substitute is missing an input or output rate | A missing rate is not a zero rate. Pricing the gap at zero would overstate the saving. |
| The substitute would cost **more** than what you are paying | A negative saving falls below the $1 floor. TOLVYN will not recommend a migration that loses you money. |
| The period contains cached tokens | The estimate prices input and output only. If you are already getting a caching discount, a figure computed without it would be wrong in your disfavour. |

#### Which substitutions exist

The map holds **which model is an acceptable substitute for which** — a judgement about capability. It holds no prices and no percentages; those come from the pricing table at the moment the finding is written.

| Model in use | Suggested substitute |
|---|---|
| `gpt-4o` | `gpt-4o-mini` |
| `gpt-4-turbo` | `gpt-4o-mini` |
| `gpt-4` | `gpt-4o-mini` |
| `claude-3-5-sonnet` | `claude-haiku-4-5` |
| `claude-sonnet-4-6` | `claude-haiku-4-5` |
| `claude-3-opus` | `claude-haiku-4-5` |
| `claude-3-sonnet` | `claude-haiku-4-5` |
| `claude-opus-4-6` | `claude-sonnet-4-6` |
| `claude-opus-4-7` | `claude-sonnet-4-6` |
| `o1` | `o3-mini` |
| `o1-pro` | `o1-mini` |

**There is no Gemini substitution, and that is deliberate.** `gemini-2.5-flash` and `gemini-2.0-flash` have both been retired by Google and now return 404, and the Gemini models still being served are not cheaper than `gemini-2.5-pro` on *both* input and output — so there is no move TOLVYN could recommend that would reliably save you anything. We would rather say nothing than suggest a migration worth nothing. (Checked against the live Gemini API on 2026-09-27; a substitution will be added when one is worth making.)

If the model you use most has no substitute listed, it produces no downgrade finding.

Example finding text:

```
100% of backend calls to claude-3-opus in this period used fewer than 500
tokens (avg 184 tokens). claude-haiku-4-5 would cost ~$12.40 instead of
~$49.60 — saving ~$37.20, at claude-haiku-4-5's current rate of $1/$5 per
million input/output tokens.
```

The rates are quoted in the finding so you can check the arithmetic against the provider's own price list.

### 2. `duplicate_prompts` — cache redundant requests


**Trigger criteria:**

| Criterion | Value |
|---|---|
| Detection method | `request_hash` column equality |
| Duplicate percentage | ≥ 5.0% of total requests |
| `request_hash` must be non-NULL | Required |

The `request_hash` column is populated by the proxy with a normalized hash of the request body (system prompt + messages + model + key parameters). A "duplicate" is two or more requests with the same hash.

**Savings estimate:** the full cost of every redundant request beyond the first per hash (`SUM(cost_microdollars)` of all but one duplicate per hash group).

Example finding text:

```
8.4% of requests this month are duplicate prompts (1247 redundant calls).
A prompt caching or request deduplication layer would save ~$42.80.
```

**Note on accuracy:** caching would only save the redundant portion if your application can tolerate cached responses (no temperature, no time-sensitive content). TOLVYN cannot tell which duplicates are intentional retries vs. avoidable cache misses.

### 3. `underutilized_cache` — enable provider prompt caching


**Trigger criteria:**

| Criterion | Value |
|---|---|
| Providers checked | `openai`, `anthropic` only |
| Minimum input tokens in window | ≥ 100,000 |
| Cache hit rate threshold | < 5% (`cached / input < 0.05`) |
| Estimated monthly savings | ≥ $0.50 |

**Savings estimate**:

```go
estimatedInputCost := int64(float64(totalCost) * 0.65)
savings := int64(float64(estimatedInputCost) * 0.30 * 0.90)
```

Assumes:

- 65% of total cost is input cost (typical ratio for chat workloads)
- 30% of input tokens are cacheable (the system prompt, few-shot examples, etc.)
- 90% discount on cached tokens (standard provider rate)

These constants are not parameterized — they are sensible defaults for typical chat applications but won't match every workload.

Example finding text:

```
You're paying full input price for repeated prompts on anthropic.
Current cache hit rate: 0.8% (target ≥5%).
Enabling prompt caching could save ~$84.20 (estimated assuming 30% of tokens cacheable).
```

Google is excluded from this rule — its prompt caching mechanism is different and TOLVYN does not yet model it.

### 4. `idle_model` — surface unused model allocations


**Trigger criteria:**

| Criterion | Value |
|---|---|
| Model had activity in the last 90 days | Required |
| Model has had 0 requests in the last 30 days | Required |
| Savings amount | `$0.00` |

Idle-model findings carry **no dollar estimate** — the previous budget-share proxy metric was removed because it presented a fabricated number. The finding is still surfaced with the model name and idle period; only the misleading savings figure is gone.

Example finding text:

```
gpt-4o-2024-05-13 has had 0 requests in the last 30 days (last used 47 days ago,
2104 total historical requests).
If you have active budgets tied to services that relied on this model,
consider removing them to reduce management overhead.
```

---

## Dismissing findings

Findings have a `dismissed` boolean. Once dismissed, a finding:

- Disappears from the active dashboard
- Is **not deleted** — it stays in the `savings_findings` table
- **Survives subsequent nightly runs** — even if the underlying condition (e.g. low cache hit rate) persists, dismissed findings are not regenerated

### Dashboard

**Savings** page → click a finding → **Dismiss**. The finding moves to the dismissed list (visible via a filter toggle).

### API

`POST /v1/savings/{id}/dismiss` — sets `dismissed = true` and `dismissed_at = now()`.

```bash
curl -X POST https://api.tolvyn.io/v1/savings/<finding-id>/dismiss \
  -H "Authorization: Bearer <jwt>"
```

To un-dismiss, currently you must wait for the next nightly rerun to regenerate a non-dismissed copy — there is no `un-dismiss` endpoint. (Dismissing is intended to be one-way.)

---

## Triggering a recompute

Tenant-facing endpoints cannot trigger a rerun on demand. Operators can:

```bash
curl -X POST https://api.tolvyn.io/v1/operator/savings/recompute \
  -H "Authorization: Bearer <operator-token>"
```

This runs the savings analyzer across all tenants immediately. Useful after a major code change or after a backfill of `request_hash`.

---

## Atomic-replace behavior

At the start of each rerun, the analyzer:

```sql
DELETE FROM savings_findings
WHERE tenant_id = $1 AND dismissed = false
```

Then it inserts fresh findings from all four rules. This means:

- A finding that no longer applies (e.g. you switched models and the small-token-request opportunity is gone) **disappears** at the next run
- A finding that **still applies** gets a **new row** with a fresh timestamp — IDs change between runs
- **Dismissed findings persist** — the `DELETE` excludes them — so dismissing a finding once dismisses it forever

If you need stable IDs for tracking outside TOLVYN, use the `(finding_type, model_id, service_name)` tuple as the natural key.

---

## Limitations

### Estimates are approximate

Every dollar figure in a savings finding is an **estimate**. Sources of inaccuracy:

- **Downgrade ratios are hardcoded** per family (not based on your actual prompt complexity)
- **Cache savings assume** 30% of input tokens are cacheable and 65% of cost is input — true for some workloads, off by 2× for others
- **Pricing drift** — savings are computed at the prices in effect during the 30-day window. A pricing change mid-window can skew estimates
- **No prompt analysis** — TOLVYN doesn't look at prompt content, so it can't tell you whether a cheaper model would actually produce acceptable output

Treat findings as a hit-list to investigate, not a guaranteed savings amount.

### Rolling 30-day window only

The window is always "last 30 days". You can't run analysis for a custom date range. To get last-quarter findings, wait until the analyzer has run for 30 days against the period of interest.

### Hardcoded model lists

The downgrade map covers 13 model families. Models released after the last update still need to be added in source — file an issue if you see expensive usage of an unmapped model.

---

## See also

- [Budgets](budgets.md) — set hard caps on the spend savings analysis surfaces
- [Pricing Changes](pricing-changes.md) — what triggers price drift
- [API Reference: Savings](../reference/api.md#savings)
