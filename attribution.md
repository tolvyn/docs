# Attribution Headers

TOLVYN uses request headers to attribute AI spend to teams, services, and features.

© 2026 TOLVYN

---

## Headers

| Header | Description | Required |
|---|---|---|
| `X-Tolvyn-Team` | Team name for cost attribution and budget enforcement | No |
| `X-Tolvyn-Service` | Service or application name | No |
| `X-Tolvyn-Feature` | Feature tag (e.g. "search", "summarize") | No |
| `X-Tolvyn-Agent` | Agent or pipeline name | No |

Without attribution headers, requests are recorded against the organization scope and show as "—" in the Team and Service columns of the dashboard.

With attribution headers, per-team budgets take effect and spend is broken down by team and service in analytics.

---

## How Team Resolution Works

`X-Tolvyn-Team` takes a **team name** (not an ID). TOLVYN looks up the team by name within your organization. If the name doesn't match an existing team, the request falls back to the team associated with the API key (if any), then to the organization scope.

Create teams in the dashboard under **Account → Teams**, or via the CLI:

```bash
tolvyn teams create --name engineering
```

---

## Adding Headers — Each Integration Method

### curl / any HTTP client

```bash
curl -X POST https://proxy.tolvyn.io/v1/proxy/anthropic/v1/messages \
  -H "Authorization: Bearer tlv_live_YOUR_KEY" \
  -H "X-Tolvyn-Team: engineering" \
  -H "X-Tolvyn-Service: chatbot-api" \
  -H "X-Tolvyn-Feature: search" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-haiku-4-5","max_tokens":100,"messages":[{"role":"user","content":"Hello"}]}'
```

### Python SDK

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_YOUR_KEY",
    team="engineering",
    service="chatbot-api",
    feature="search",
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
```

Headers are sent on every request made by this client instance.

### Python — Anthropic SDK (proxy mode)

```python
import anthropic

client = anthropic.Anthropic(
    base_url="https://proxy.tolvyn.io/v1/proxy/anthropic",
    api_key="tlv_live_YOUR_KEY",
    default_headers={
        "X-Tolvyn-Team": "engineering",
        "X-Tolvyn-Service": "chatbot-api",
    },
)
```

### Node.js SDK

```typescript
import { OpenAI } from 'tolvyn';

const client = new OpenAI({
    tolvynApiKey: "tlv_live_YOUR_KEY",
    team: "engineering",
    service: "chatbot-api",
    feature: "search",
});
```

### Node.js — OpenAI SDK (proxy mode)

```javascript
import OpenAI from 'openai';

const client = new OpenAI({
    apiKey: "tlv_live_YOUR_KEY",
    baseURL: "https://proxy.tolvyn.io/v1/proxy/openai",
    defaultHeaders: {
        "X-Tolvyn-Team": "engineering",
        "X-Tolvyn-Service": "chatbot-api",
    },
});
```

### Environment variables (proxy mode, no SDK change)

```bash
export OPENAI_API_KEY="tlv_live_YOUR_KEY"
export OPENAI_BASE_URL="https://proxy.tolvyn.io/v1/proxy/openai"
```

For attribution headers in proxy mode without the TOLVYN SDK, you must set headers explicitly on your HTTP client. The environment variable approach does not carry attribution headers.

---

## Budget Enforcement

Budgets can be scoped to a team. When `X-Tolvyn-Team` is present and matches a team with a hard-mode budget, the budget is checked before forwarding the request. If exceeded, the proxy returns HTTP 429.

See [Budgets & Alerts](budgets-and-alerts.md) for budget configuration.

---

## Analytics

Attribution data appears in:

- **Dashboard → Requests** — Team and Service columns per request
- **Dashboard → Usage → By Team** — aggregated spend per team
- **Dashboard → Usage → By Service** — aggregated spend per service
- `GET /v1/usage/by-team` and `GET /v1/usage/by-service` API endpoints
