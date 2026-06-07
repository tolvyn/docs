# Metering Accuracy

How TOLVYN measures cost — where it's exact, where it's approximate, and which direction the error goes. The rule: **TOLVYN may under-count, but it never over-charges**, and this page tells you where.

## Precision

- Costs are tracked in **microdollars** (1 USD = 1,000,000 µ$) using integer/decimal math — no floating-point drift.
- Per request, each token bucket (input, output, cached read, cache write) is priced at the model's published per-million-token rate and summed.

## Where metering is exact

For standard requests on a priced model — streaming or non-streaming — token counts come directly from the provider's own usage report, so the recorded cost is exact to the microdollar.

## Where it currently under-counts (never over-charges)

- **1-hour cache writes** and **>200k long-context** requests are billed at the **base rate** today. Providers charge a premium for these tiers; TOLVYN has the premium rates staged but **not yet switched on in production**, so these specific usages are slightly under-counted until they are. (When enabled, they only ever *raise* the recorded cost toward the true rate.)
- **Batch API** submissions are asynchronous: a submission returns a job id with no per-request usage, so it is recorded at **$0 and explicitly tagged as unmetered** — identifiable in your data, not a silent gap. The (discounted) batch usage itself is not metered.

In every case the error is one-directional — toward under-counting — so TOLVYN never bills you for more than you used.

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
