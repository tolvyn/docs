# Quickstart

Get a request flowing through TOLVYN and visible in the dashboard in under 5 minutes.

---

## Prerequisites

- An API key from OpenAI, Anthropic, Google, or DeepSeek
- One of: Python 3.9+, Node.js 18+, Go 1.21+, or any HTTP client (e.g. curl)
- A few minutes

---

## Step 1 — Sign up

Go to [app.tolvyn.io/signup](https://app.tolvyn.io/signup).

You will need:
- Your name
- A work email you can open right now
- A password of **at least 12 characters**

You start on the **Free** plan: 10,000 included requests per month, no card required, no time limit.

**Verify your email before you try to log in.** Signing up sends a verification
email; until you open the link, login answers `403 email_unverified`. The link is
**single-use and valid for 24 hours** — open it once, from any device.

---

## Step 2 — Add your provider key

Navigate to **Account → Provider Connections** in the dashboard.

Pick a provider and paste the corresponding key:

| Provider | Key format | Where to get one |
|---|---|---|
| OpenAI | `sk-...` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| Anthropic | `sk-ant-...` | [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys) |
| Google | API key for Generative Language API | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) |
| DeepSeek | `sk-...` | [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) |

The provider key is encrypted with AES-256-GCM under a server-side master key, and your tenant id and the provider name are bound into the ciphertext as additional authenticated data — so a stored key cannot be decrypted for a different tenant or a different provider, even by the server itself. Your application code never holds it again. TOLVYN uses it to authenticate to the provider when proxying your requests.

You can add keys for any of these providers from the same dashboard. DeepSeek is OpenAI-compatible — see [Integration Modes → DeepSeek](../integration-modes.md#deepseek-openai-compatible) for the client recipe.

---

## Step 3 — Get your TOLVYN API key

Navigate to **API Keys → Create**.

Give the key a name (e.g. `local-dev` or `backend-prod`) and click **Create**.

The full key is shown **once** and looks like:

```
tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ
```

- Production keys are prefixed `tlv_live_`
- Test keys (for sandboxed runs) are prefixed `tlv_test_`

Save it now. The dashboard stores only the prefix and a hash — there is no recovery if you lose the full key. Create a new one if that happens.

---

## Step 4 — Make your first request

Pick the integration style that fits your stack. All four options below produce the same metered request.

### Python SDK

```bash
pip install tolvyn
```

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ",
    openai_api_key="sk-...",        # REQUIRED for fail-open; without it there is no fallback
    team="engineering",
    service="my-app",
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.choices[0].message.content)
```

### Node.js SDK

```bash
npm install tolvyn
```

```javascript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
  tolvynApiKey: 'tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ',
  openAIApiKey: 'sk-...',           // REQUIRED for fail-open; without it there is no fallback
  team: 'engineering',
  service: 'my-app',
});

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
});
console.log(response.choices[0].message.content);
```

### Go SDK

```bash
go get github.com/tolvyn/tolvyn-go@latest
```

```go
package main

import (
    "context"
    "fmt"

    oai "github.com/openai/openai-go"
    tolvyn "github.com/tolvyn/tolvyn-go"
    tolvynopenai "github.com/tolvyn/tolvyn-go/openai"
)

func main() {
    client := tolvynopenai.NewClient(tolvyn.ClientOptions{
        TolvynAPIKey:   "tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ",
        ProviderAPIKey: "sk-...",        // REQUIRED for fail-open; without it there is no fallback
        Team:           "engineering",
        Service:        "my-app",
    })

    resp, err := client.Chat.Completions.New(context.Background(), oai.ChatCompletionNewParams{
        Model: oai.F(oai.ChatModelGPT4o),
        Messages: oai.F([]oai.ChatCompletionMessageParamUnion{
            oai.ChatCompletionUserMessageParam{
                Role: oai.F(oai.ChatCompletionUserMessageParamRoleUser),
                Content: oai.F([]oai.ChatCompletionContentPartUnionParam{
                    oai.ChatCompletionContentPartTextParam{
                        Text: oai.F("Hello"),
                        Type: oai.F(oai.ChatCompletionContentPartTextTypeText),
                    },
                }),
            },
        }),
    })
    if err != nil {
        panic(err)
    }
    fmt.Println(resp.Choices[0].Message.Content)
}
```

### curl (proxy mode)

```bash
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_aB3xK9mP2vQ8nF4hR7sT1uW5yE6dC0gJ" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

> **Why add your provider key?** It is what makes fail-open possible. If the
> SDK cannot reach TOLVYN, it retries the request directly against the provider
> using that key. **Without the provider key there is no fallback at all** — the
> key is required for this behaviour, not merely recommended.
>
> **What fail-open covers, precisely** (verified in the Go SDK; see
> [Fail-open behavior](../sdks/go.md#fail-open-behavior)): connection refused,
> DNS failure, timeout, EOF, connection reset, and **HTTP 503**. It does **not**
> cover `500`, `502` or `504` — those propagate to your code as errors. It does
> not cover 4xx, which are real API errors and should not be retried.
>
> **A fallen-back request is not metered.** It goes straight to the provider,
> its `X-Tolvyn-*` attribution headers are stripped, and no row appears in your
> dashboard. Your AI keeps working; that spend is invisible to TOLVYN.

### Using Anthropic or Google instead?

Replace the import and constructor.

**Anthropic (Python):**

```python
from tolvyn import Anthropic

client = Anthropic(
    tolvyn_api_key="tlv_live_...",
    anthropic_api_key="sk-ant-...",   # REQUIRED for fail-open; without it there is no fallback
    team="engineering",
    service="my-app",
)
```

**Google (Python, requires the `[google]` extra):**

```python
from tolvyn import Google

goog = Google(tolvyn_api_key="tlv_live_...")
model = goog.GenerativeModel("gemini-2.5-flash")
```

**Proxy mode (any language, any provider):**

```bash
# OpenAI
curl https://proxy.tolvyn.io/v1/proxy/openai/v1/chat/completions ...

# Anthropic
curl https://proxy.tolvyn.io/v1/proxy/anthropic/v1/messages ...

# Google
curl https://proxy.tolvyn.io/v1/proxy/google/v1beta/models/...
```

---

## Step 5 — See it in the dashboard

Open [app.tolvyn.io/requests](https://app.tolvyn.io/requests).

Within a few seconds of the request completing, you will see a row with:

- Timestamp
- Model (`gpt-4o`)
- Tokens in / out
- Exact cost in USD
- Latency
- HTTP status

Click the row to see the full metadata: attribution headers, provider response time, ledger sequence number, and any tags captured from the request headers.

On the **Dashboard** page you will see the request reflected in the cost-over-time chart and the model breakdown within a minute.

---

## Next steps

You now have a working metered request. The next things to set up depend on your goal:

| Goal | Read this |
|---|---|
| Pick the right integration for production reliability | [Integration Modes](../integration-modes.md) |
| Block runaway spend before the bill arrives | [Budgets & Enforcement](../features/budgets.md) |
| Break costs down by team, service, user, or customer | [Team Insights](../features/team-insights.md) · [End Customers](../features/end-customers.md) |
| Migrate from an existing AI gateway | [Migration guides](../migration/index.md) |

---

## Troubleshooting

**Login returns 403 with `email_unverified`** — you have not opened the verification link yet. Check your spam folder. The link is single-use and valid for 24 hours; if it has expired or been used, request a new one from the login page.

**Signup returns 400 with `password_too_short`** — passwords must be at least 12 characters. (`password_too_long` means over 72 bytes, which is the bcrypt limit.)

**Request returns 401 with `invalid_key`** — your TOLVYN key is wrong, expired, or revoked. Check the prefix in **API Keys** matches the key you're using.

**Request returns 409 with `provider_credentials_missing`** — you have not added a provider key for the provider you're calling. Go back to Step 2.

**Request returns 400 with `unknown_provider`** — the provider segment in the URL is not one of `openai`, `anthropic`, `google`, `deepseek` or `custom`.

**Request returns 503** — TOLVYN proxy is unreachable. In SDK mode this triggers the fail-open path, if a provider key is configured: the request continues directly to the provider but is **not metered**. In proxy mode the request fails.

**Request returns 500, 502 or 504** — these do **not** trigger fail-open. The SDK surfaces them to your code. Only connection-level failures and `503` fall back.

**Request succeeds but does not appear in the dashboard** — the SDK fell back direct to the provider. Check your network can reach `proxy.tolvyn.io` and that you set `tolvyn_api_key` (not `openai_api_key`) as the TOLVYN key.

Email [founder@tolvyn.io](mailto:founder@tolvyn.io) if anything else looks wrong.
