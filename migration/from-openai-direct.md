# Migrate from OpenAI direct to TOLVYN

Two lines of your code change. There is a step before them that is not code.

---

## What setup is

1. **Register your OpenAI key with TOLVYN**, once, on the **Providers** page (`/providers`). TOLVYN calls OpenAI with *your* key, so until it has one every proxied request returns `409 provider_credentials_missing`. This is a dashboard step, not a code change, and it is the one people miss.
2. **Point the base URL at the proxy** — `https://api.openai.com/v1` → `https://proxy.tolvyn.io/v1/proxy/openai/v1`. **The trailing `/v1` matters on both sides**: the OpenAI SDKs append `/chat/completions` to whatever base you give them.
3. **Use a TOLVYN key in place of your provider key** — `sk-...` → `tlv_live_...`. Your OpenAI key stays with TOLVYN and never goes in your application again.

Steps 2 and 3 are the two lines. Step 1 is why the first request after a five-minute migration sometimes returns a 409 instead of a completion.

Your prompts, models, parameters, and response format are completely unchanged. TOLVYN speaks OpenAI's API natively.

---

## Before / after

### Python

**Before:**
```python
from openai import OpenAI

client = OpenAI(api_key="sk-...")
```

**After:**
```python
from openai import OpenAI

client = OpenAI(
    api_key="tlv_live_...",
    base_url="https://proxy.tolvyn.io/v1/proxy/openai/v1",
)
```

Add your real OpenAI key once in the TOLVYN dashboard, on the **Providers** page (`/providers`). Your app code never holds a provider key again.

---

### Node.js

**Before:**
```js
import OpenAI from "openai";

const client = new OpenAI({ apiKey: "sk-..." });
```

**After:**
```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "tlv_live_...",
  baseURL: "https://proxy.tolvyn.io/v1/proxy/openai/v1",
});
```

---

### curl

**Before:**
```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer sk-..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-6-luna","messages":[{"role":"user","content":"Hello"}]}'
```

**After:**
```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-6-luna","messages":[{"role":"user","content":"Hello"}]}'
```

---

### Raw HTTP

**Before:**
```
POST /v1/chat/completions HTTP/1.1
Host: api.openai.com
Authorization: Bearer sk-...
Content-Type: application/json
```

**After:**
```
POST /v1/proxy/openai/v1/chat/completions HTTP/1.1
Host: proxy.tolvyn.io
Authorization: Bearer tlv_live_...
Content-Type: application/json
```

---

## What you get immediately

Once the two lines are changed, TOLVYN automatically:

- Logs every request with model, tokens, cost, and latency
- Calculates cost in real-time using live pricing data
- Stores a tamper-evident audit ledger of all spend
- Lets you set budgets and get alerts when limits are approached

No configuration required. The dashboard is live the moment the first request goes through.

---

## If TOLVYN is unavailable

The setup above is proxy mode: every call goes through TOLVYN, and there is no automatic fallback. If TOLVYN cannot be reached, the call fails. For an automatic fallback to OpenAI direct, use the `tolvyn` SDK with your OpenAI key (SDK mode); it falls back only when TOLVYN was not reached, and a call that falls back is not metered. See [Integration Modes](../integration-modes.md).

---

## Optional: add attribution headers

To see cost broken down by team, feature, or user in the dashboard:

```
X-Tolvyn-Team: backend-team
X-Tolvyn-Service: summarization
X-Tolvyn-Feature: report-gen
X-Tolvyn-User: user-123
X-Tolvyn-End-Customer: acme-corp
```

None of these are required. Add them incrementally as you need the visibility.
