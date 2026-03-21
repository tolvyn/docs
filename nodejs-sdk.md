# Node.js SDK

TOLVYN's Node.js SDK is a drop-in replacement for the official `openai` and `@anthropic-ai/sdk` packages. Route all AI traffic through TOLVYN with a one-line change.

© 2026 TOLVYN

---

## 1. Installation

```bash
npm install tolvyn
```

Requires Node.js 18+. Peer dependencies: `openai`, `@anthropic-ai/sdk`, `node-fetch`.

---

## 2. Quick Start — OpenAI (ESM)

```typescript
import { OpenAI } from "tolvyn";

const client = new OpenAI({ tolvynApiKey: "tlv_live_..." });

const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "Hello!" }],
});
console.log(response.choices[0].message.content);
```

**CommonJS (CJS)**

```javascript
const { OpenAI } = require("tolvyn");

const client = new OpenAI({ tolvynApiKey: "tlv_live_..." });
```

---

## 3. Anthropic Example

**ESM**

```typescript
import { Anthropic } from "tolvyn";

const client = new Anthropic({ tolvynApiKey: "tlv_live_..." });

const response = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude." }],
});
console.log(response.content[0].text);
```

**CJS**

```javascript
const { Anthropic } = require("tolvyn");

const client = new Anthropic({ tolvynApiKey: "tlv_live_..." });
```

---

## 4. Streaming

Streaming works identically to the upstream SDKs:

```typescript
import { OpenAI } from "tolvyn";

const client = new OpenAI({ tolvynApiKey: "tlv_live_..." });

const stream = await client.chat.completions.stream({
    model: "gpt-4o",
    messages: [{ role: "user", content: "Write a haiku." }],
});

for await (const chunk of stream) {
    process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

---

## 5. Tagging Requests

Add attribution tags at client construction time. Tags appear in the dashboard, `tolvyn tail`, and usage analytics.

```typescript
import { OpenAI } from "tolvyn";

const client = new OpenAI({
    tolvynApiKey: "tlv_live_...",
    team: "platform",
    service: "chat-api",
    feature: "customer-support",
    agent: "support-bot-v2",
});
```

Tags are sent as HTTP headers on every request:

| Constructor option | HTTP Header |
|---|---|
| `team` | `X-Tolvyn-Team` |
| `service` | `X-Tolvyn-Service` |
| `feature` | `X-Tolvyn-Feature` |
| `agent` | `X-Tolvyn-Agent` |

---

## 6. Environment Variables

| Variable | Used by |
|---|---|
| `TOLVYN_API_KEY` | All classes — TOLVYN authentication key |
| `TOLVYN_PROXY_URL` | All classes — override proxy endpoint |
| `OPENAI_API_KEY` | `OpenAI` — fallback key for fail-open |
| `ANTHROPIC_API_KEY` | `Anthropic` — fallback key for fail-open |

```bash
export TOLVYN_API_KEY=tlv_live_...
export OPENAI_API_KEY=sk-...
```

```typescript
import { OpenAI } from "tolvyn";

// Reads TOLVYN_API_KEY and OPENAI_API_KEY from environment
const client = new OpenAI({});
```

---

## 7. Fail-Open Behavior

Fail-open is enabled by default (`failOpen: true`). When the TOLVYN proxy is unreachable or returns HTTP 503, the SDK automatically retries the request directly against the provider:

```
[TOLVYN] OpenAI proxy unavailable (...), attempting direct fallback...
```

Network errors that trigger fail-open (from `failopen.ts`):
- `ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`, `EHOSTUNREACH`, `ENETUNREACH`, `ENOTFOUND`
- HTTP 503 from the proxy

Conditions that do **not** trigger fail-open:
- HTTP 4xx responses (except 503) — intentional governance decisions.

To disable:

```typescript
const client = new OpenAI({
    tolvynApiKey: "tlv_live_...",
    failOpen: false,
});
```

---

## 8. Constructor Parameters

### `OpenAI`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `tolvynApiKey` | string or undefined | undefined | TOLVYN API key. Falls back to `TOLVYN_API_KEY` env var. Required. |
| `proxyUrl` | string or undefined | undefined | Override proxy URL. Falls back to `TOLVYN_PROXY_URL` env var. |
| `team` | string or undefined | undefined | Team attribution tag. Sent as `X-Tolvyn-Team`. |
| `service` | string or undefined | undefined | Service attribution tag. Sent as `X-Tolvyn-Service`. |
| `feature` | string or undefined | undefined | Feature attribution tag. Sent as `X-Tolvyn-Feature`. |
| `agent` | string or undefined | undefined | Agent attribution tag. Sent as `X-Tolvyn-Agent`. |
| `failOpen` | boolean | true | Route directly to OpenAI if proxy is unreachable. |
| `openaiApiKey` | string or undefined | undefined | OpenAI key for fail-open. Falls back to `OPENAI_API_KEY`. |

### `Anthropic`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `tolvynApiKey` | string or undefined | undefined | TOLVYN API key. Falls back to `TOLVYN_API_KEY` env var. Required. |
| `proxyUrl` | string or undefined | undefined | Override proxy URL. Falls back to `TOLVYN_PROXY_URL` env var. |
| `team` | string or undefined | undefined | Team attribution tag. Sent as `X-Tolvyn-Team`. |
| `service` | string or undefined | undefined | Service attribution tag. Sent as `X-Tolvyn-Service`. |
| `feature` | string or undefined | undefined | Feature attribution tag. Sent as `X-Tolvyn-Feature`. |
| `agent` | string or undefined | undefined | Agent attribution tag. Sent as `X-Tolvyn-Agent`. |
| `failOpen` | boolean | true | Route directly to Anthropic if proxy is unreachable. |
| `anthropicApiKey` | string or undefined | undefined | Anthropic key for fail-open. Falls back to `ANTHROPIC_API_KEY`. |

---

## 9. Proxied Methods

The TOLVYN `OpenAI` class proxies the following properties to the underlying `openai` client:

- `client.chat` — chat completions
- `client.completions` — legacy completions
- `client.embeddings` — text embeddings
- `client.models` — model listing

The TOLVYN `Anthropic` class proxies:

- `client.messages` — messages API
- `client.models` — model listing
