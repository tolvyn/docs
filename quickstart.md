# Quick Start

Get your first governed AI request running in 5 minutes.

© 2026 TOLVYN

---

## 1. Prerequisites

- Python 3.9+ **or** Node.js 18+
- An OpenAI or Anthropic API key
- A TOLVYN account (sign up at [app.tolvyn.io/signup](https://app.tolvyn.io/signup))

---

## 2. Create an Account

Sign up at [app.tolvyn.io/signup](https://app.tolvyn.io/signup). You'll start on a **trial** plan (1,000 included requests).

Then install the CLI and authenticate:

```bash
# macOS / Linux (download binary)
curl -fsSL https://releases.tolvyn.io/install.sh | sh

# Configure and authenticate
tolvyn init
```

`tolvyn init` prompts you for your API URL, email, and password, then saves credentials to `~/.tolvyn/config.json`.

Already have credentials? Use `tolvyn login` instead.

---

## 3. Install the SDK

**Python**

```bash
pip install tolvyn
```

**Node.js**

```bash
npm install tolvyn
```

---

## 4. Add Your Provider Key

Tell TOLVYN which provider key to use when forwarding your requests:

```bash
tolvyn providers add openai
```

You'll be prompted to paste your API key (input is hidden). Supported providers: `openai`, `anthropic`, `google`.

---

## 5. Change One Line

Replace your existing SDK import with the TOLVYN drop-in. Everything else stays identical.

**Python — OpenAI**

```python
# Before
# from openai import OpenAI

# After
from tolvyn import OpenAI

client = OpenAI(tolvyn_api_key="tlv_live_...")   # or set TOLVYN_API_KEY env var

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.choices[0].message.content)
```

**Python — Anthropic**

```python
from tolvyn import Anthropic

client = Anthropic(tolvyn_api_key="tlv_live_...")

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.content[0].text)
```

**Node.js — OpenAI**

```typescript
import { OpenAI } from "tolvyn";

const client = new OpenAI({ tolvynApiKey: "tlv_live_..." });

const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: "Hello!" }],
});
console.log(response.choices[0].message.content);
```

The TOLVYN proxy URL and your stored provider key are resolved automatically.

---

## 6. Watch Live Requests

In a separate terminal, start the live tail:

```bash
tolvyn tail
```

Output format:

```
TIME     | TEAM/SERVICE          | MODEL            |   TOKENS |     COST |  LATENCY
─────────┼───────────────────────┼──────────────────┼──────────┼──────────┼─────────
15:42:01 | backend/chat          | gpt-4o           |    1,234 |  $0.0048 |    842ms
15:42:09 | backend/chat          | gpt-4o           |      892 |  $0.0031 |    601ms
```

Filter by team, service, or model:

```bash
tolvyn tail --team backend --model gpt-4o
```

Press `Ctrl+C` to stop.

---

## 7. Set Your First Budget

Create a monthly spending limit for your organization:

```bash
tolvyn budgets create --scope org --amount 100 --period monthly --mode soft
```

- `--mode soft` sends an alert when the limit is hit but does not block requests.
- `--mode hard` returns HTTP 429 to callers when the limit is exceeded.

You will receive email alerts at 50%, 75%, 90%, and 100% utilization.

---

## 8. What's Next

- [How TOLVYN Works](concepts.md) — architecture, proxy pipeline, and the ledger
- [Python SDK Reference](python-sdk.md) — all constructor parameters and async support
- [Node.js SDK Reference](nodejs-sdk.md) — TypeScript types and ESM/CJS imports
- [Proxy Mode](proxy-mode.md) — use any HTTP client without installing the SDK
- [CLI Reference](cli-reference.md) — complete command and flag reference
- [API Reference](api-reference.md) — REST API endpoints for automation
- [Budgets & Alerts](budgets-and-alerts.md) — enforcement modes, anomaly detection
