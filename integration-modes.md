# Integration Modes

TOLVYN offers two integration modes. Pick one based on your reliability requirements.

## TL;DR

| Capability | SDK mode | Proxy mode |
|------------|----------|------------|
| Setup | Install `tolvyn` package | Set `OPENAI_BASE_URL` env var |
| Languages | Python, Node.js, Go | Any |
| Fail-open (auto-fallback to provider direct) | ✅ Yes — **requires a provider key**; only when TOLVYN was not reached (connection failure, or TOLVYN's own `503`) | ❌ No — request fails if TOLVYN is unreachable |
| Latency overhead | <50ms | <50ms |
| TOLVYN in critical path | No (fails open) | Yes |
| Recommended for | Production workloads | Prototyping, internal tools, non-critical batch |

**Rule of thumb:** Use SDK mode for production. Use proxy mode when you want the simplest possible integration and can tolerate rare TOLVYN downtime.

---

## SDK Mode

SDK mode wraps the OpenAI or Anthropic SDK. Install the `tolvyn` package, change the import, and use a TOLVYN key — and register your provider key with TOLVYN first, or the proxy has no credential to call the provider with. Your call sites are unchanged.

The key advantage: **if TOLVYN is unreachable, the SDK retries the request directly to OpenAI or Anthropic — provided you configured a provider key.** The fallback is narrower than it sounds and the limits are listed below: it covers only the cases where TOLVYN was not reached, so the request cannot have been forwarded — the connection was refused, DNS or the TLS handshake failed, connecting timed out, or TOLVYN answered `503` with the header `X-Tolvyn-Unavailable: 1`. It does **not** cover a provider's own `503`, any other `5xx`, or a timeout after the request was sent. A request that falls back never reaches TOLVYN, so it is not metered, not budget-checked and not in the ledger.

### Python

```python
# Before (standard OpenAI)
from openai import OpenAI
client = OpenAI(api_key="sk-...")

# After (TOLVYN SDK mode)
from tolvyn import OpenAI
client = OpenAI(
    tolvyn_api_key="tlv_live_...",   # your TOLVYN key
    openai_api_key="sk-...",          # REQUIRED for fail-open; no key, no fallback
    team="engineering",
    service="chatbot-api"
)

# Everything else is identical
response = client.chat.completions.create(
    model="gpt-6-luna",
    messages=[{"role": "user", "content": "Hello"}]
)
```

### Node.js

```javascript
// Before (standard OpenAI)
import OpenAI from 'openai'
const client = new OpenAI({ apiKey: 'sk-...' })

// After (TOLVYN SDK mode)
import { OpenAI } from 'tolvyn'
const client = new OpenAI({
  tolvynApiKey: 'tlv_live_...',
  openAIApiKey: 'sk-...',        // REQUIRED for fail-open; no key, no fallback
  team: 'engineering',
  service: 'chatbot-api'
})

// Everything else is identical
const response = await client.chat.completions.create({
  model: 'gpt-6-luna',
  messages: [{ role: 'user', content: 'Hello' }]
})
```

### What happens when TOLVYN is down?

When the SDK cannot reach `proxy.tolvyn.io` (connection refused, DNS or TLS failure, connect timeout), or TOLVYN answers `503` with `X-Tolvyn-Unavailable: 1`:

1. SDK detects the failure automatically — no error is thrown to your code
2. SDK retries the request directly to `api.openai.com` using your `openai_api_key`
3. Your AI call succeeds as normal
4. The request is **not** metered in TOLVYN for that call (it bypassed the proxy)
5. You may see a slightly higher latency on that one request (one retry)

This means that in SDK mode a TOLVYN outage does not have to take your application down with it — within the limits above. It does not take TOLVYN out of your critical path entirely: the fallback needs a provider key you configured, it does not cover every failure mode, and while it is in effect you are running without metering, budgets or the ledger.

---

## DeepSeek (OpenAI-compatible)

DeepSeek's API is OpenAI-compatible: the SDKs' `DeepSeek` client is used exactly like the OpenAI one and already points at TOLVYN's DeepSeek proxy path.

**SDK mode** — use the `DeepSeek` client, whose fail-open fallback goes to DeepSeek (an `OpenAI` client pointed at this path would fall back to OpenAI):

```python
# Python
from tolvyn import DeepSeek

client = DeepSeek(tolvyn_api_key="tlv_live_...")
resp = client.chat.completions.create(
    model="deepseek-flash",
    messages=[{"role": "user", "content": "Hello"}],
)
```

```javascript
// Node.js
import { DeepSeek } from 'tolvyn'

const client = new DeepSeek({ tolvynApiKey: 'tlv_live_...' })
const resp = await client.chat.completions.create({
  model: 'deepseek-flash',
  messages: [{ role: 'user', content: 'Hello' }],
})
```

**Proxy mode** — point any OpenAI client at `https://proxy.tolvyn.io/v1/proxy/deepseek` with your TOLVYN key as the bearer token.

**Add your DeepSeek key in the dashboard** (**Providers** → DeepSeek, at `/providers`). TOLVYN swaps it in when forwarding; your SDK only ever holds the TOLVYN key.

> **Served-model note:** DeepSeek may answer as a different model name than the one you sent (for example `deepseek-v4-flash`); usage shows the name DeepSeek reports, at that model's rate. The older names `deepseek-chat` and `deepseek-reasoner` still work.

See the per-SDK pages for the full DeepSeek recipe, including Go: [Python](./sdks/python.md) · [Node.js](./sdks/nodejs.md) · [Go](./sdks/go.md).

---

## Proxy Mode

Proxy mode requires no SDK installation. Point your existing OpenAI client at TOLVYN's proxy URL.

```bash
# Set these environment variables
export OPENAI_BASE_URL=https://proxy.tolvyn.io/v1/proxy/openai/v1
export OPENAI_API_KEY=tlv_live_...   # your TOLVYN key, not your OpenAI key
```

### Python (proxy mode)

```python
from openai import OpenAI

client = OpenAI(
    api_key="tlv_live_...",                               # TOLVYN key
    base_url="https://proxy.tolvyn.io/v1/proxy/openai/v1"   # TOLVYN proxy — keep the trailing /v1
)

response = client.chat.completions.create(
    model="gpt-6-luna",
    messages=[{"role": "user", "content": "Hello"}],
    extra_headers={
        "X-Tolvyn-Team": "engineering",
        "X-Tolvyn-Service": "chatbot-api"
    }
)
```

### curl (proxy mode)

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -H "X-Tolvyn-Team: engineering" \
  -H "X-Tolvyn-Service: chatbot-api" \
  -d '{
    "model": "gpt-6-luna",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### Any language (raw HTTP)

```
POST https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions
Authorization: Bearer tlv_live_...
Content-Type: application/json
X-Tolvyn-Team: engineering
X-Tolvyn-Service: chatbot-api

{"model":"gpt-6-luna","messages":[{"role":"user","content":"Hello"}]}
```

### Important: TOLVYN is in your critical path

In proxy mode, if `proxy.tolvyn.io` is unreachable, your AI call fails. There is no automatic fallback.

To mitigate this:
- Set up uptime monitoring on `https://proxy.tolvyn.io/health`
- Configure your HTTP client with a reasonable timeout (recommended: 30s)
- Use SDK mode for any workload where AI downtime is unacceptable

---

### Google Gemini (proxy mode)

Use Gemini's native API through TOLVYN: point the Google GenAI SDK, or your HTTP client, at `https://proxy.tolvyn.io/v1/proxy/google` with your TOLVYN key. Gemini's OpenAI-compatible endpoint (`/v1beta/openai/...`) is not supported through TOLVYN and is answered `501`. So is the Files API upload (`client.files.upload()`): upload the file with your Google key directly, then pass its URI (`file_data`) in a request through TOLVYN, which is metered. See [the proxy reference](reference/api.md#proxy).

With Google's current Python SDK, `google-genai`, set the base URL and API version once, on the client:

```python
from google import genai
from google.genai import types

client = genai.Client(
    api_key="tlv_live_...",  # your TOLVYN key, never your Gemini key
    http_options=types.HttpOptions(
        base_url="https://proxy.tolvyn.io/v1/proxy/google",
        api_version="v1beta",
        headers={  # tags for every call this client makes
            "X-Tolvyn-Team": "engineering",
            "X-Tolvyn-Service": "support-bot",
        },
    ),
)

response = client.models.generate_content(
    model="gemini-3.1-flash-lite",
    contents="Hello",
    config=types.GenerateContentConfig(
        # tags for this call only, added to the client's
        http_options=types.HttpOptions(headers={"X-Tolvyn-Feature": "greeting"}),
    ),
)
print(response.text)
```

- `api_key` is your **TOLVYN key**. The SDK sends it in `x-goog-api-key`, which TOLVYN reads as your TOLVYN key; your Gemini key stays in TOLVYN's Connections and never goes in your code.
- Headers set on the client go with every call; headers set on a call are added to them.

### Adding your own fallback

Proxy mode has no automatic fallback (see above). If a Gemini call must still be answered while TOLVYN is unavailable, wrap the call yourself and fall back to Google directly:

```python
import os

from google import genai
from google.genai import errors, types

client = genai.Client(  # through TOLVYN, as above
    api_key="tlv_live_...",
    http_options=types.HttpOptions(base_url="https://proxy.tolvyn.io/v1/proxy/google", api_version="v1beta"),
)
google_direct = genai.Client(api_key=os.environ["GEMINI_API_KEY"])  # your Gemini key, used only for the fallback


def generate(**kwargs):
    try:
        return client.models.generate_content(**kwargs)
    except errors.APIError as e:
        if e.code != 503 or e.response is None or e.response.headers.get("x-tolvyn-unavailable") != "1":
            raise  # Google's own errors, and TOLVYN's 429 and 451
        return google_direct.models.generate_content(**kwargs)  # TOLVYN itself was unavailable
    except Exception:
        return google_direct.models.generate_content(**kwargs)  # the proxy could not be reached


response = generate(model="gemini-3.1-flash-lite", contents="Hello")
print(response.text)
```

- **It falls back only when TOLVYN itself is unavailable:** the proxy cannot be reached at all, or it answers `503` with the header `X-Tolvyn-Unavailable: 1`. TOLVYN sends that header only when its own service could not take the call, so the call never reached Google.
- **It raises every other error, including Google's own `503`.** When Google is overloaded, TOLVYN passes Google's `503` through unchanged, without the header. Calling Google directly would hit the same overloaded service, so the error is raised to your code as before.
- **It never falls back on `429` (a budget or quota limit) or `451` (a kill switch).** Those are TOLVYN doing its job; falling back would spend the money the limit was set to stop.
- **Fallback calls are not metered.** A call answered by `google_direct` does not pass through TOLVYN: it is not budget-checked, not attributed and not in your ledger, and the dashboard shows no row for it.
- A call that times out also falls back. If the slow call had already reached Google, it may run twice.

---

## Switching from proxy mode to SDK mode

It is two lines:

**Python:**
```python
# Remove this:
base_url="https://proxy.tolvyn.io/v1/proxy/openai/v1"

# Add this:
from tolvyn import OpenAI   # change import
openai_api_key="sk-..."     # add fallback key
```

**Node.js:**
```javascript
// Remove this:
baseURL: 'https://proxy.tolvyn.io/v1/proxy/openai/v1'

// Add this:
import { OpenAI } from 'tolvyn'   // change import
openAIApiKey: 'sk-...'            // add fallback key
```

---

## Reliability: design targets and behaviour

**Design targets and how TOLVYN behaves:**
- Latency added by the proxy: a design target of under 50 ms at p99. In our own benchmark (2,000 requests against a
  local stub, 1 Nov 2026) it added about 5 ms at p50. No production figure is published.
- SDK fail-open: fallback only when TOLVYN was not reached — a connection failure (refused, DNS, TLS, connect timeout) or TOLVYN's own `503` with `X-Tolvyn-Unavailable: 1`. **Not** on a provider's `503`, any other `5xx`, `429`, `451`, or a timeout after the request was sent (TOLVYN may already have forwarded it)
- No prompt modification: TOLVYN does not modify your prompts or messages
- No response modification: TOLVYN does not modify provider responses
- No content storage: TOLVYN stores metadata (model, tokens, cost, latency) only, not prompt text or response content

**Outside what TOLVYN covers:**
- Provider availability (OpenAI, Anthropic, Google, DeepSeek outages are outside TOLVYN's control)
- Metering covers every request that reaches TOLVYN. **Fail-open requests do not reach TOLVYN and are therefore not metered** — they are not counted, not budget-checked, and not in the ledger. If the proxy is down for an hour, that hour of spend is invisible

**Health endpoint:**
```
GET https://proxy.tolvyn.io/health
→ {"status":"ok","db":"ok","version":"1.0.0"}
```

Use this for uptime monitoring in proxy mode.

---

## Production Checklist

Before going to production with TOLVYN:

- [ ] **SDK mode**: Install `tolvyn` package, set both `tolvyn_api_key` and provider `api_key` — the provider key is **required** for fail-open, not optional
- [ ] **Proxy mode**: Set up uptime monitoring on `proxy.tolvyn.io/health`
- [ ] Set `X-Tolvyn-Team` and `X-Tolvyn-Service` headers — required for cost attribution
- [ ] Create a budget with hard mode for each team or service
- [ ] Test the fail-open path: temporarily set `proxy_url` to an unreachable host and verify your code still works (SDK mode only)
- [ ] Add `X-Tolvyn-User` for per-user tracking (optional but recommended)
- [ ] Add `X-Tolvyn-End-Customer` if you are a SaaS company billing AI costs to customers

---

## See also

- [Migrate from Helicone](./migration/from-helicone.md)
- [Migrate from direct OpenAI](./migration/from-openai-direct.md)
- [Python SDK reference](./sdks/python.md)
- [Node.js SDK reference](./sdks/nodejs.md)
- [Go SDK reference](./sdks/go.md)
