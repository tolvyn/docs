# Proxy Mode

Use TOLVYN with any HTTP client — no SDK required. Point your existing OpenAI or Anthropic client at `proxy.tolvyn.io` and set your TOLVYN API key as the Bearer token.

© 2026 TOLVYN

---

## 1. How It Works

In proxy mode:

1. Set your client's base URL to the TOLVYN proxy endpoint.
2. Use your TOLVYN API key (`tlv_live_...`) as the Bearer token instead of your provider key.
3. Send requests exactly as you normally would — the proxy swaps the authorization header and forwards to the provider.

TOLVYN meters the request, records it in the ledger, and enforces budgets. Your provider key never leaves the server.

---

## 2. Provider Proxy URLs

| Provider | Proxy URL |
|---|---|
| OpenAI | `https://proxy.tolvyn.io/v1/proxy/openai` |
| Anthropic | `https://proxy.tolvyn.io/v1/proxy/anthropic` |

Path suffixes are preserved. For example, `POST /v1/chat/completions` becomes `POST https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions`.

---

## 3. Language Examples

### Python (using `openai` directly)

```python
from openai import OpenAI

client = OpenAI(
    api_key="tlv_live_...",
    base_url="https://proxy.tolvyn.io/v1/proxy/openai",
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

### Node.js (using `openai` directly)

```typescript
import OpenAI from "openai";

const client = new OpenAI({
    apiKey: "tlv_live_...",
    baseURL: "https://proxy.tolvyn.io/v1/proxy/openai",
});

const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "Hello!" }],
});
```

### curl

```bash
curl -X POST https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

Anthropic:

```bash
curl -X POST https://proxy.tolvyn.io/v1/proxy/anthropic/v1/messages \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### Go

```go
package main

import (
    "context"
    openai "github.com/sashabaranov/go-openai"
)

func main() {
    cfg := openai.DefaultConfig("tlv_live_...")
    cfg.BaseURL = "https://proxy.tolvyn.io/v1/proxy/openai/v1"
    client := openai.NewClientWithConfig(cfg)

    resp, _ := client.CreateChatCompletion(
        context.Background(),
        openai.ChatCompletionRequest{
            Model: "gpt-4o",
            Messages: []openai.ChatCompletionMessage{
                {Role: "user", Content: "Hello!"},
            },
        },
    )
    _ = resp
}
```

### Ruby

```ruby
require "openai"

client = OpenAI::Client.new(
  access_token: "tlv_live_...",
  uri_base: "https://proxy.tolvyn.io/v1/proxy/openai"
)

response = client.chat(
  parameters: {
    model: "gpt-4o",
    messages: [{ role: "user", content: "Hello!" }]
  }
)
```

---

## 4. Attribution Headers

Add attribution headers to tag requests with team, service, feature, or agent metadata:

```bash
curl -X POST https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "X-Tolvyn-Team: platform" \
  -H "X-Tolvyn-Service: chat-api" \
  -H "X-Tolvyn-Feature: customer-support" \
  -H "X-Tolvyn-Agent: support-bot-v2" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello!"}]}'
```

| Header | Purpose |
|---|---|
| `X-Tolvyn-Team` | Team or cost center name |
| `X-Tolvyn-Service` | Microservice or application name |
| `X-Tolvyn-Feature` | Feature or product area |
| `X-Tolvyn-Agent` | AI agent identifier |

---

## 5. Proxy Mode vs SDK Mode

| Feature | Proxy Mode | SDK Mode |
|---|---|---|
| SDK installation required | No | Yes |
| Language support | Any HTTP client | Python, Node.js |
| Attribution headers | Manual | Set once in constructor |
| Fail-open (auto-fallback) | Not available | Built-in |
| Streaming | Yes | Yes |
| Metering and ledger | Yes | Yes |
| Budget enforcement | Yes | Yes |

---

## 6. Limitations of Proxy Mode

- **No automatic fail-open.** If the proxy is unavailable, requests fail. Use the SDK for built-in fallback behavior.
- **No automatic key resolution.** You must manage and pass your TOLVYN API key explicitly in each client configuration.
- **Manual header management.** Attribution headers must be added to each request or configured in your HTTP client's default headers.
