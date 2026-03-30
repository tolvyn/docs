# Frequently Asked Questions

© 2026 TOLVYN

---

## Does TOLVYN read my prompts or responses?

TOLVYN does not log, store, or inspect the content of your prompts or AI responses. Only metadata is recorded: model used, token counts, cost, team attribution, latency, and status code. The request and response bodies pass through the proxy without being written to disk.

---

## What happens if TOLVYN goes down?

TOLVYN SDKs support **fail-open mode** (enabled by default). If the proxy is unreachable, the SDK automatically routes the request directly to the AI provider using your original API key — so your application keeps working.

To use fail-open mode, provide your fallback provider key:

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_YOUR_KEY",
    openai_api_key="sk-...",   # fallback — used if proxy is unreachable
    fail_open=True,            # default
)
```

In proxy mode (without the SDK), requests will fail if TOLVYN is unreachable unless you implement your own fallback logic.

---

## How is my provider API key stored?

Provider API keys are encrypted at rest using AES-256-GCM with a server-side master key. The original key is never returned after it is saved — not in the dashboard, not via the API. Only the TOLVYN proxy can decrypt and use it when forwarding requests.

---

## What latency does TOLVYN add?

Metering and ledger writes happen asynchronously after the provider response is delivered to your application. The proxy adds only the network hop to and from the TOLVYN server. In practice this is typically under 10ms of added latency for requests to the same region.

---

## Can I use TOLVYN with any AI provider?

TOLVYN currently supports **OpenAI**, **Anthropic**, and **Google** (Gemini). Any model available through these providers works automatically — you don't need to configure individual models.

For providers not yet supported, use the standard SDK with those providers directly and contact founder@tolvyn.io to request support.

---

## How do budgets work — what happens when I hit the limit?

Budgets have two enforcement modes:

- **Soft mode** — alerts are sent at 50%, 75%, 90%, and 100% of the limit. Requests continue. No requests are blocked.
- **Hard mode** — at 100%, the proxy returns HTTP 429 with a `budget_exceeded` error body. Requests are rejected until the budget period resets or the limit is raised.

Budget periods reset automatically at the start of each month, week, or day depending on the period you configured.

---

## What is the immutable ledger?

Every request is appended to a hash-chained ledger. Each record includes the SHA-256 hash of the previous record, making any retroactive modification detectable. The ledger provides a tamper-evident audit trail for all AI spend.

See [Immutable Ledger](ledger.md) for full details and verification instructions.

---

## How is TOLVYN different from just looking at my OpenAI invoice?

Your provider invoice tells you total spend. TOLVYN tells you:

- Which team or service caused each dollar of spend
- Which model and feature drove the cost
- What was the cost trend over time (anomaly detection alerts you to spikes)
- Whether spend was within agreed budgets at the time of each request
- An immutable, verifiable record — not just a monthly statement

TOLVYN also supports multiple providers in one dashboard, so OpenAI and Anthropic spend appear together.

---

## Is there a free trial?

Yes. New accounts start on a **trial plan** that includes a set of free requests. No credit card is required to sign up. Sign up at [app.tolvyn.io/signup](https://app.tolvyn.io/signup).

---

## What models are supported?

All models available through OpenAI, Anthropic, and Google are supported. You do not need to configure individual models. The model you pass in your API call is forwarded directly to the provider.

To see the current list of models with pricing data, use:

```bash
tolvyn models
```

or `GET /v1/models` via the API.
