# Python SDK

TOLVYN's Python SDK is a drop-in replacement for the official `openai` and `anthropic` packages. No code changes beyond the import line and a TOLVYN API key.

© 2026 TOLVYN

---

## 1. Installation

```bash
pip install tolvyn
```

Requires Python 3.9+. Depends on `openai`, `anthropic`, and `httpx`.

---

## 2. Quick Start — OpenAI

```python
from tolvyn import OpenAI

client = OpenAI(tolvyn_api_key="tlv_live_...")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Summarize this quarter's AI spend."}],
)
print(response.choices[0].message.content)
```

The TOLVYN SDK is a subclass of `openai.OpenAI`. All methods, parameters, and return types are identical. The only difference is that requests route through `proxy.tolvyn.io` and are metered.

---

## 3. Anthropic Example

```python
from tolvyn import Anthropic

client = Anthropic(tolvyn_api_key="tlv_live_...")

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude."}],
)
print(response.content[0].text)
```

---

## 4. Async Classes

`AsyncOpenAI` and `AsyncAnthropic` are async drop-in replacements for the official async clients:

```python
import asyncio
from tolvyn import AsyncOpenAI

async def main():
    client = AsyncOpenAI(tolvyn_api_key="tlv_live_...")
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "Hello!"}],
    )
    print(response.choices[0].message.content)

asyncio.run(main())
```

```python
from tolvyn import AsyncAnthropic

client = AsyncAnthropic(tolvyn_api_key="tlv_live_...")
response = await client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=512,
    messages=[{"role": "user", "content": "Quick summary please."}],
)
```

---

## 5. Streaming

Streaming works identically to the upstream SDKs — TOLVYN does not interfere with the stream:

```python
from tolvyn import OpenAI

client = OpenAI(tolvyn_api_key="tlv_live_...")

with client.chat.completions.stream(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Write a haiku."}],
) as stream:
    for chunk in stream:
        print(chunk.choices[0].delta.content or "", end="", flush=True)
```

TOLVYN measures time-to-first-token (TTFT) and total latency for streaming requests.

---

## 6. Tagging Requests

Add attribution tags to associate requests with teams, services, features, or agents. Tags appear in the dashboard, `tolvyn tail`, and usage analytics.

```python
from tolvyn import OpenAI

client = OpenAI(
    tolvyn_api_key="tlv_live_...",
    team="platform",
    service="chat-api",
    feature="customer-support",
    agent="support-bot-v2",
)
```

These are sent as HTTP headers on every request:

| Parameter | HTTP Header |
|---|---|
| `team` | `X-Tolvyn-Team` |
| `service` | `X-Tolvyn-Service` |
| `feature` | `X-Tolvyn-Feature` |
| `agent` | `X-Tolvyn-Agent` |

---

## 7. Environment Variables

All constructor parameters can be set via environment variables:

| Variable | Used by |
|---|---|
| `TOLVYN_API_KEY` | All classes — TOLVYN authentication key |
| `TOLVYN_PROXY_URL` | All classes — override proxy endpoint |
| `OPENAI_API_KEY` | `OpenAI`, `AsyncOpenAI` — fallback key for fail-open |
| `ANTHROPIC_API_KEY` | `Anthropic`, `AsyncAnthropic` — fallback key for fail-open |

```bash
export TOLVYN_API_KEY=tlv_live_...
export OPENAI_API_KEY=sk-...
```

```python
from tolvyn import OpenAI

# No explicit keys needed — reads from environment
client = OpenAI()
```

---

## 8. Fail-Open Behavior

Fail-open is enabled by default (`fail_open=True`). When the TOLVYN proxy is unreachable or returns HTTP 503, the SDK transparently retries the request directly against the provider:

```
TOLVYN proxy unreachable — routing direct to OpenAI (fail-open)
```

Error conditions that trigger fail-open (from `_failopen.py`):
- `httpx.ConnectError`
- `httpx.ConnectTimeout`
- `httpx.ReadTimeout`
- `httpx.WriteTimeout`
- `httpx.RemoteProtocolError`
- HTTP 503 from the proxy

Conditions that do **not** trigger fail-open:
- HTTP 4xx errors (401, 429, etc.) — these are intentional governance decisions.

To disable:

```python
client = OpenAI(tolvyn_api_key="tlv_live_...", fail_open=False)
```

---

## 9. Constructor Parameters

### `OpenAI` and `AsyncOpenAI`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `tolvyn_api_key` | str or None | None | TOLVYN API key. Falls back to `TOLVYN_API_KEY` env var. Required. |
| `proxy_url` | str or None | None | Override proxy URL. Falls back to `TOLVYN_PROXY_URL` env var. |
| `team` | str or None | None | Team attribution tag. Sent as `X-Tolvyn-Team`. |
| `service` | str or None | None | Service attribution tag. Sent as `X-Tolvyn-Service`. |
| `feature` | str or None | None | Feature attribution tag. Sent as `X-Tolvyn-Feature`. |
| `agent` | str or None | None | Agent attribution tag. Sent as `X-Tolvyn-Agent`. |
| `fail_open` | bool | True | Route directly to OpenAI if proxy is unreachable. |
| `openai_api_key` | str or None | None | OpenAI key for fail-open fallback. Falls back to `OPENAI_API_KEY`. |
| `**kwargs` | any | — | Passed through to `openai.OpenAI` (e.g. `timeout`, `max_retries`). |

### `Anthropic` and `AsyncAnthropic`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `tolvyn_api_key` | str or None | None | TOLVYN API key. Falls back to `TOLVYN_API_KEY` env var. Required. |
| `proxy_url` | str or None | None | Override proxy URL. Falls back to `TOLVYN_PROXY_URL` env var. |
| `team` | str or None | None | Team attribution tag. Sent as `X-Tolvyn-Team`. |
| `service` | str or None | None | Service attribution tag. Sent as `X-Tolvyn-Service`. |
| `feature` | str or None | None | Feature attribution tag. Sent as `X-Tolvyn-Feature`. |
| `agent` | str or None | None | Agent attribution tag. Sent as `X-Tolvyn-Agent`. |
| `fail_open` | bool | True | Route directly to Anthropic if proxy is unreachable. |
| `anthropic_api_key` | str or None | None | Anthropic key for fail-open fallback. Falls back to `ANTHROPIC_API_KEY`. |
| `**kwargs` | any | — | Passed through to `anthropic.Anthropic` (e.g. `timeout`, `max_retries`). |
