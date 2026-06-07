# Frequently Asked Questions

## Does TOLVYN read my prompts or responses?

No. TOLVYN does not log, store, or inspect the content of your prompts or AI responses. Only metadata is recorded: model, token counts, cost, attribution tags, latency, and status code. Request and response bodies pass through the proxy without being written to disk.

---

## What happens if TOLVYN goes down?

The SDKs support **fail-open mode** (enabled by default). If the proxy is unreachable, the SDK routes the request directly to the AI provider using your provider key — so your application keeps working.

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_YOUR_KEY",
    openai_api_key="sk-...",   # fallback — used only if the proxy is unreachable
    fail_open=True,            # default
)
```

Intentional governance responses (HTTP 4xx, e.g. a hard-budget 429 or a kill-switch 451) are **not** retried — only outages trigger fail-open. In proxy mode (no SDK), there is no automatic fallback. See [How TOLVYN Works → Fail-open](how-tolvyn-works.md#6-fail-open-behavior).

---

## How is my provider API key stored?

Encrypted at rest with AES-256-GCM using a server-side master key (per-tenant data-encryption key). The original key is never returned after it is saved — not in the dashboard, not via the API. Only the proxy can decrypt it, in memory, when forwarding requests.

---

## Is my data isolated from other customers?

Yes. Tenant data is isolated at the database level using PostgreSQL row-level security (enforced in production), in addition to application-level tenant scoping. See [How TOLVYN Works → Tenant isolation](how-tolvyn-works.md#7-tenant-isolation--security).

---

## What latency does TOLVYN add?

Metering and ledger writes happen asynchronously, after your application already has the provider response. The proxy adds only the network hop to and from the TOLVYN server — typically well under 50ms.

---

## Which AI providers can I use?

**OpenAI**, **Anthropic**, **Google (Gemini)**, and **DeepSeek** — all verified end-to-end. Any model offered by these providers works automatically; you don't configure individual models. DeepSeek is OpenAI-compatible — see [Integration Modes → DeepSeek](../integration-modes.md#deepseek-openai-compatible).

For a provider not yet supported, use that provider's SDK directly and email founder@tolvyn.io to request it.

---

## How do budgets work — what happens at the limit?

Budgets have **three** modes:

- **Soft** — alerts fire at threshold crossings; requests are never blocked.
- **Hard** — at the cap, the proxy returns HTTP 429 and the provider is never called.
- **Approval** — blocks like hard, but records a pending approval an administrator can grant for a bounded amount and time.

Periods reset automatically (daily / weekly / monthly). See [Budgets](../features/budgets.md). For *alerting without blocking*, see [Spend Quotas](../features/spend-quotas.md).

---

## Is metering exact?

For standard requests on priced models, yes — costs come from the provider's own usage report and are exact to the microdollar. A few cases currently **under-count** (never over-charge), and tiny costs can display as `$0.0000` due to rounding even though they were metered. The full, honest breakdown — and reconciliation as the backstop — is in [Metering Accuracy](../features/metering-accuracy.md).

---

## What is the immutable ledger?

Every request is appended to a SHA-256 hash-chained, HMAC-signed ledger, making any retroactive modification detectable — a tamper-evident audit trail for all AI spend. See [Immutable Ledger](../features/ledger.md).

---

## How is this different from just reading my provider invoice?

Your invoice tells you total spend. TOLVYN tells you which team, service, feature, agent, user, or end-customer drove each dollar; the cost trend over time (with anomaly alerts); whether each request was within budget at the time; and a verifiable, immutable record — across OpenAI, Anthropic, Google, and DeepSeek in one place.

---

## Is there a free plan?

Yes — the **Free** plan includes 10,000 requests per month, no credit card required. Sign up at [app.tolvyn.io/signup](https://app.tolvyn.io/signup). See [How TOLVYN Works → Plans](how-tolvyn-works.md#9-plans).

---

## What models are supported?

All models offered by OpenAI, Anthropic, Google, and DeepSeek. You don't configure individual models — the model you pass is forwarded to the provider. To see the current priced model list:

```bash
tolvyn models
```

or `GET /v1/models`.
