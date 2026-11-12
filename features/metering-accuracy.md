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

## Gemini audio, speech and embeddings

- **Audio is priced at Google's audio rates.** Google's `usageMetadata` reports how many of a request's input, cached and output tokens were audio, per modality. TOLVYN prices that audio share at the model's audio rate and the rest at its text rate, so a request that sends a voice clip is charged what Google charges for audio, not the text rate.
- **Transcription and text-to-speech models** (for example `gemini-3.5-transcribe` and `gemini-3.8-flash-lite-tts`) are priced the same way: their input and output are audio on one side, at the rates Google lists for them.
- **A model with a single listed rate uses it for every modality.** Where Google lists no separate audio rate for a model, its audio tokens are priced at its text rate.
- **Embeddings are priced on input only.** An embedding produces no output tokens, so none are estimated. Google reports no usage for embedding calls, so the input is counted from the text you send (never from the JSON around it or from encoded media).
- **Images or video in a `gemini-embedding-2` request are flagged, not guessed.** Their tokens have no published per-token rate, so the request's cost covers its text and is marked **partly priced** (the tag `partly_priced: embedding_media`), and the media is charged to your budgets at the highest rate any Google model can bill, so a budget is never under-protected by it.
- **Inline audio is estimated by its duration before the call.** A budget reserves money before the request is sent. For audio sent inline, TOLVYN reads the clip's length and counts 25 tokens a second, as Google does, rather than counting the encoded bytes. This estimate only sizes the reservation: the recorded cost always comes from Google's own counts. An audio file passed by reference (`file_data`) has no length TOLVYN can read before the call, so its reservation is smaller; its recorded cost is still exact.
- **What each request records.** `modality` is what Google reported the request carried: `text`, or the non-text modalities joined by `+` in the order audio, image, video, document (for example `audio` or `audio+image`); it is part of the [ledger record](ledger.md). Alongside the input, cached and output token counts, each request also keeps their audio share, in `tokens_audio_input`, `tokens_audio_cached` and `tokens_audio_output`; each is a part of its total, never more.

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
- [Tamper-Evident Ledger](ledger.md)
