# Metering Accuracy

How TOLVYN measures cost — where it's exact, where it's approximate, and which direction the error goes. **There is one place where a figure can be too high — DeepSeek traffic outside peak hours — and it is the first section below.** Everywhere else, TOLVYN may under-count but never over-states.

## Precision

- Costs are tracked in **microdollars** (1 USD = 1,000,000 µ$) using integer/decimal math — no floating-point drift.
- Per request, each token bucket (input, output, cached read, cache write) is priced at the model's published per-million-token rate and summed.

## Where metering is exact

For standard requests on a priced model — streaming or non-streaming — token counts come directly from the provider's own usage report, so the recorded cost is exact to the microdollar.

## Where metering is currently inexact

### DeepSeek spend is overstated outside peak hours

**DeepSeek charges half price outside 01:00–04:00 and 06:00–10:00 UTC on
weekdays. TOLVYN stores one rate per model and applies the peak rate to every
request, so an individual DeepSeek request made outside those windows is recorded
at exactly double its true cost.**

Your DeepSeek figures are correct for traffic inside the two weekday windows and
double outside them — which includes **all weekend traffic**. What that means for
a total depends on when your workload runs:

| when your DeepSeek traffic runs | your DeepSeek total is overstated by |
|---|---|
| mostly 06:00–10:00 UTC on weekdays | about **40%** |
| spread evenly across the week | about **65%** |
| US business hours, or weekends, or batch overnight | **100% — exactly double** |

**The error is only ever in this direction. TOLVYN never understates DeepSeek
spend**, so your real DeepSeek bill is always lower than the figure shown, never
higher. **No other provider is affected** — this is specific to DeepSeek's
time-of-day pricing.

**Until this is corrected, reconcile DeepSeek against your provider invoice
rather than against this dashboard.** Every other provider's figures are
unaffected and can be read as usual.

### Where it under-counts

- **1-hour cache writes** and **>200k long-context** requests are billed at the **base rate** today. Providers charge a premium for these tiers; TOLVYN has the premium rates staged but **not yet switched on in production**, so these specific usages are slightly under-counted until they are. (When enabled, they only ever *raise* the recorded cost toward the true rate.)
- **Batch API** submissions are asynchronous: a submission returns a job id with no per-request usage, so it is recorded at **$0 and explicitly tagged as unmetered** — identifiable in your data, not a silent gap. The (discounted) batch usage itself is not metered.

**These two are one-directional toward under-counting.** The DeepSeek off-peak
case above is the one exception in the other direction, and it is the only place
where a TOLVYN figure can exceed what you were actually charged.

## Reconciliation is the backstop

Upload your provider invoices and TOLVYN reconciles its metered spend against them, surfacing any drift. See [Reconciliation](reconciliation.md).

## "Why does my call show $0.0000?"

The dashboard shows costs to four decimal places. A cheap model or a single small call can cost only a few microdollars — below $0.0001 — so it **rounds to `$0.0000` in the UI even though it was metered**. The stored value is exact (microdollar precision), and usage analytics sum the exact figures, not the rounded display.

## Served-model names

The dashboard shows the model the **provider actually served**, which may be a concrete snapshot behind an alias:

| You request | May appear as | Billed at |
|---|---|---|
| `deepseek-chat` | `deepseek-v4-flash` | the `deepseek-chat` rate |
| `claude-haiku-4-5` | `claude-haiku-4-5-20251001` | the `claude-haiku-4-5` rate |

This is expected and correctly priced — the cost uses the canonical family rate regardless of the served snapshot name.

**Gemini "thinking" tokens** are billed as **output** tokens. A Gemini 2.5+ request can return few or no visible text tokens while spending its budget on internal reasoning; those reasoning tokens are counted as output (which is how the provider bills them).

## See also

- [Spend Quotas → Burn-rate forecast](spend-quotas.md#burn-rate-forecast) — forecasts are estimates built on this metering
- [Reconciliation](reconciliation.md)
- [Immutable Ledger](ledger.md)
