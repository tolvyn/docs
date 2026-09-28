# Migrate to TOLVYN

TOLVYN sits in front of any AI proxy or direct OpenAI integration: the request and response shapes are unchanged, so your call sites stay as they are. What changes is the base URL, the key you send, and one piece of setup — registering your provider key with TOLVYN.
Pick your migration path:

| From | Time | Difficulty |
|------|------|------------|
| [Helicone](./from-helicone.md) | 15 minutes | Easy |
| [Portkey](./from-portkey.md) | 15 minutes | Easy |
| [OpenAI direct](./from-openai-direct.md) | 5 minutes | Trivial |

All migrations preserve your existing API calls. No prompt changes. No model changes.
TOLVYN is fail-open: if the proxy is unreachable and you configured a provider key, the SDK retries directly against the provider. Fail-open covers connection failures, timeouts and `503` — not `500`, `502` or `504` — and a fallen-back request is not metered.
