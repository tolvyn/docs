# API Reference

Base URL: `https://api.tolvyn.io`

All endpoints return `Content-Type: application/json`. Costs are expressed in **microdollars** (1 USD = 1,000,000 µ$) internally and as `"$N.NNNN"` formatted strings in responses.

© 2026 TOLVYN

---

## Authentication

| Method | Header |
|---|---|
| JWT (tenant endpoints) | `Authorization: Bearer <jwt-token>` |
| Operator token | `Authorization: Bearer <TOLVYN_OPERATOR_TOKEN>` |

JWTs are issued by `POST /v1/auth/login` and expire after 24 hours.

---

## Health

### GET /health

No authentication required.

**Response 200:**

```json
{
  "status": "ok",
  "db": "ok",
  "version": "1.0.0",
  "timestamp": "2026-03-21T15:42:00Z"
}
```

**Response 503 (DB unavailable):**

```json
{
  "status": "degraded",
  "db": "error"
}
```

```bash
curl https://api.tolvyn.io/health
```

---

## Proxy

These endpoints forward requests to provider APIs. Use your TOLVYN API key as the Bearer token.

### POST /v1/proxy/openai/{path}

Proxies to OpenAI. All OpenAI path suffixes are supported (e.g. `/v1/chat/completions`, `/v1/embeddings`).

**Auth:** TOLVYN API key (Bearer)

**Request:** identical to the target OpenAI endpoint

**Response:** identical to the OpenAI response (streamed if requested)

```bash
curl -X POST https://api.tolvyn.io/v1/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello!"}]}'
```

### POST /v1/proxy/anthropic/{path}

Proxies to Anthropic. All Anthropic path suffixes are supported (e.g. `/v1/messages`).

**Auth:** TOLVYN API key (Bearer)

```bash
curl -X POST https://api.tolvyn.io/v1/proxy/anthropic/v1/messages \
  -H "Authorization: Bearer tlv_live_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":1024,"messages":[{"role":"user","content":"Hello!"}]}'
```

---

## Auth

### POST /v1/auth/signup

Create a new account. Starts a 30-day trial.

**Auth:** None

**Request body:**

```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "password": "securepassword"
}
```

Password must be at least 8 characters.

**Response 201:**

```json
{
  "tenant_id": "550e8400-...",
  "email": "alice@example.com",
  "plan_tier": "trial",
  "token": "eyJhbGci..."
}
```

```bash
curl -X POST https://api.tolvyn.io/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","password":"securepass"}'
```

---

### POST /v1/auth/login

Authenticate and receive a JWT.

**Auth:** None

**Request body:**

```json
{
  "email": "alice@example.com",
  "password": "securepassword"
}
```

**Response 200:**

```json
{
  "tenant_id": "550e8400-...",
  "email": "alice@example.com",
  "plan_tier": "starter",
  "token": "eyJhbGci...",
  "expires_at": "2026-03-22T15:42:00Z"
}
```

```bash
curl -X POST https://api.tolvyn.io/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"securepass"}'
```

---

### POST /v1/auth/refresh

Issue a fresh JWT for an authenticated tenant. Requires a valid (non-expired) token.

**Auth:** JWT

**Response 200:**

```json
{
  "token": "eyJhbGci...",
  "expires_at": "2026-03-22T15:42:00Z"
}
```

```bash
curl -X POST https://api.tolvyn.io/v1/auth/refresh \
  -H "Authorization: Bearer <token>"
```

---

## Account

### GET /v1/account

Retrieve account and subscription details.

**Auth:** JWT

**Response 200:**

```json
{
  "tenant_id": "550e8400-...",
  "name": "Alice",
  "email": "alice@example.com",
  "plan_tier": "starter",
  "status": "active",
  "trial_ends_at": null,
  "created_at": "2026-01-15T09:00:00Z",
  "updated_at": "2026-03-01T10:00:00Z",
  "subscription": {
    "plan_tier": "starter",
    "included_requests": 100000,
    "status": "active",
    "current_period_start": "2026-03-01T00:00:00Z",
    "current_period_end": "2026-03-31T23:59:59Z"
  }
}
```

---

### PUT /v1/account

Update account name.

**Auth:** JWT

**Request body:**

```json
{
  "name": "Alice Smith"
}
```

**Response 200:**

```json
{
  "name": "Alice Smith"
}
```

---

## Provider Keys

### POST /v1/provider-keys

Store or rotate a provider API key. Keys are encrypted server-side and never returned.

**Auth:** JWT

**Request body:**

```json
{
  "provider": "openai",
  "key": "sk-..."
}
```

Supported providers: `openai`, `anthropic`, `google`. Upserting replaces the existing key for the provider and increments `key_version`.

**Response 201:**

```json
{
  "id": "b2c3d4e5-...",
  "provider": "openai",
  "message": "Provider key stored successfully"
}
```

---

### GET /v1/provider-keys

List connected provider keys. Key values are not returned.

**Auth:** JWT

**Response 200:**

```json
[
  {
    "id": "b2c3d4e5-...",
    "provider": "openai",
    "key_version": 2,
    "last_rotated_at": "2026-03-15T14:30:00Z",
    "created_at": "2026-01-15T09:00:00Z"
  }
]
```

---

### DELETE /v1/provider-keys/{id}

Remove a provider key.

**Auth:** JWT

**Response 200:**

```json
{ "status": "deleted" }
```

---

## Teams

### POST /v1/teams

Create a team.

**Auth:** JWT

**Request body:**

```json
{
  "name": "platform",
  "cost_center": "ENG-001",
  "parent_team_id": null
}
```

**Response 201:**

```json
{
  "id": "a1b2c3d4-...",
  "tenant_id": "550e8400-...",
  "name": "platform",
  "cost_center": "ENG-001",
  "parent_team_id": null,
  "created_at": "2026-03-01T09:00:00Z"
}
```

---

### GET /v1/teams

List all teams.

**Auth:** JWT

**Response 200:** array of team objects (same schema as POST response)

---

### GET /v1/teams/{id}

Get a single team by ID.

**Auth:** JWT

**Response 200:** team object

**Response 404:**

```json
{ "error": "not_found", "message": "Team not found" }
```

---

### PUT /v1/teams/{id}

Update team name or cost center.

**Auth:** JWT

**Request body:** same fields as POST (all optional)

**Response 200:** updated team object

---

### DELETE /v1/teams/{id}

Delete a team.

**Auth:** JWT

**Response 200:**

```json
{ "status": "deleted" }
```

---

## API Keys

### POST /v1/api-keys

Create a new TOLVYN API key. The full key value is returned once and cannot be retrieved again.

**Auth:** JWT

**Request body:**

```json
{
  "name": "prod-api",
  "environment": "production",
  "team_id": "a1b2c3d4-..."
}
```

`team_id` is optional. `environment` defaults to `"production"`.

**Response 201:**

```json
{
  "id": "c3d4e5f6-...",
  "key": "tlv_live_xK3mQ9pR...",
  "prefix": "tlv_live_xK",
  "name": "prod-api",
  "environment": "production"
}
```

---

### GET /v1/api-keys

List API keys. Key values are not returned.

**Auth:** JWT

**Response 200:**

```json
[
  {
    "id": "c3d4e5f6-...",
    "prefix": "tlv_live_xK...",
    "name": "prod-api",
    "environment": "production",
    "last_used_at": "2026-03-21T15:42:00Z",
    "revoked_at": null,
    "created_at": "2026-01-20T10:00:00Z"
  }
]
```

---

### DELETE /v1/api-keys/{id}

Revoke an API key. The key stops working within the current request cycle (proxy cache is invalidated).

**Auth:** JWT

**Response 200:**

```json
{ "status": "deleted" }
```

---

## Budgets

### POST /v1/budgets

Create a budget.

**Auth:** JWT

**Request body:**

```json
{
  "scope_type": "team",
  "scope_id": "a1b2c3d4-...",
  "amount_usd": 200.00,
  "period": "monthly",
  "mode": "hard"
}
```

`scope_type`: `"organization"`, `"team"`, or `"service"`. `scope_id` is required for `team` and `service` scopes. `period`: `"monthly"`, `"weekly"`, or `"daily"`. `mode`: `"soft"` or `"hard"`.

**Response 201:**

```json
{
  "id": "d4e5f6a7-...",
  "scope_type": "team",
  "scope_id": "a1b2c3d4-...",
  "amount_usd": "$200.0000",
  "period": "monthly",
  "mode": "hard"
}
```

---

### GET /v1/budgets

List all budgets.

**Auth:** JWT

**Response 200:**

```json
[
  {
    "id": "d4e5f6a7-...",
    "scope_type": "organization",
    "scope_id": null,
    "amount_usd": "$500.0000",
    "current_spend_usd": "$142.8300",
    "utilization_pct": 28.6,
    "period": "monthly",
    "mode": "soft"
  }
]
```

---

### GET /v1/budgets/{id}

Get a single budget.

**Auth:** JWT

**Response 200:** budget object (same schema as list)

---

### PUT /v1/budgets/{id}

Update a budget's amount, mode, or period.

**Auth:** JWT

**Request body:** same fields as POST (all optional)

**Response 200:** updated budget object

---

### DELETE /v1/budgets/{id}

Delete a budget.

**Auth:** JWT

**Response 200:**

```json
{ "status": "deleted" }
```

---

## Usage

### GET /v1/usage/summary

Spend summary with top models and teams.

**Auth:** JWT

**Query parameters:**

| Parameter | Description |
|---|---|
| `from` | Start date (RFC3339 or YYYY-MM-DD) |
| `to` | End date (RFC3339 or YYYY-MM-DD) |
| `team_id` | Filter by team UUID |
| `model` | Filter by model ID |

**Response 200:**

```json
{
  "total_cost_usd": "$142.8300",
  "total_requests": 48231,
  "total_tokens_input": 12400000,
  "total_tokens_output": 3800000,
  "top_models": [
    {
      "model_id": "gpt-4o",
      "requests": 32100,
      "cost_usd": "$98.2100",
      "cost_microdollars": 98210000
    }
  ],
  "top_teams": [
    {
      "team_id": "a1b2c3d4-...",
      "requests": 31050,
      "cost_usd": "$89.3200",
      "cost_microdollars": 89320000
    }
  ]
}
```

---

### GET /v1/usage/requests

Paginated request history.

**Auth:** JWT

**Query parameters:** `from`, `to`, `team_id`, `model`, `limit` (default 50, max 1000), `offset`

**Response 200:**

```json
{
  "data": [
    {
      "id": "e5f6a7b8-...",
      "provider": "openai",
      "model_id": "gpt-4o",
      "tokens_input": 823,
      "tokens_output": 411,
      "cost_usd": "$0.0048",
      "latency_total_ms": 842,
      "team_id": "a1b2c3d4-...",
      "service_name": "chat-api",
      "status_code": 200,
      "created_at": "2026-03-21T15:42:01Z"
    }
  ],
  "total": 48231,
  "limit": 50,
  "offset": 0
}
```

---

### GET /v1/usage/by-model

Usage grouped by model.

**Auth:** JWT

**Query parameters:** `from`, `to`

**Response 200:** array of `{ model_id, provider, requests, cost_usd, cost_microdollars, tokens_input, tokens_output }`

---

### GET /v1/usage/by-team

Usage grouped by team.

**Auth:** JWT

**Query parameters:** `from`, `to`

**Response 200:** array of `{ team_id, team_name, requests, cost_usd, cost_microdollars }`

---

### GET /v1/usage/by-service

Usage grouped by service name.

**Auth:** JWT

**Query parameters:** `from`, `to`

**Response 200:** array of `{ service_name, requests, cost_usd, cost_microdollars }`

---

## Ledger

### GET /v1/ledger

List ledger records (paginated).

**Auth:** JWT

**Query parameters:** `limit` (default 50, max 1000), `offset`

**Response 200:**

```json
{
  "data": [
    {
      "id": "f6a7b8c9-...",
      "tenant_id": "550e8400-...",
      "sequence_number": 1024,
      "previous_hash": "a3f8c2...",
      "record_hash": "b4e9d1...",
      "hmac_signature": "c5f0e2...",
      "request_id": "e5f6a7b8-...",
      "cost_microdollars": 4820,
      "hierarchy_path": "team:platform/service:chat-api",
      "provider": "openai",
      "model_id": "gpt-4o",
      "model_family": "gpt-4",
      "modality": "text",
      "tokens_input": 823,
      "tokens_output": 411,
      "budget_status": "ok",
      "enforcement_action": "allow",
      "created_at": "2026-03-21T15:42:01Z"
    }
  ],
  "total": 48231,
  "limit": 50,
  "offset": 0
}
```

---

### GET /v1/ledger/verify

Verify the integrity of the ledger chain for a sequence range.

**Auth:** JWT

**Query parameters:**

| Parameter | Description |
|---|---|
| `from_seq` | Start sequence number (default: 1) |
| `to_seq` | End sequence number (default: latest) |

**Response 200:**

```json
{
  "valid": true,
  "records_checked": 1024,
  "first_invalid_sequence": null,
  "reason": null
}
```

On tampering detected:

```json
{
  "valid": false,
  "records_checked": 512,
  "first_invalid_sequence": 513,
  "reason": "seq 513: previous_hash mismatch (stored abc..., expected def...)"
}
```

---

## Alerts

### GET /v1/alerts

List alerts. Returns unacknowledged alerts by default.

**Auth:** JWT

**Query parameters:** `limit`, `offset`

**Response 200:**

```json
[
  {
    "id": "g7b8c9d0-...",
    "alert_type": "budget_threshold",
    "severity": "warning",
    "title": "Budget organization at 75% utilization",
    "metadata": {
      "budget_id": "d4e5f6a7-...",
      "scope_type": "organization",
      "threshold": 75,
      "utilization": "75.3%",
      "spend": 376500000,
      "amount": 500000000
    },
    "acknowledged_at": null,
    "created_at": "2026-03-21T14:30:00Z"
  }
]
```

---

### POST /v1/alerts/{id}/acknowledge

Mark an alert as acknowledged.

**Auth:** JWT

**Response 200:**

```json
{ "status": "acknowledged" }
```

---

## Stream / SSE

### GET /v1/stream/tail

Live tail of proxy requests over Server-Sent Events (SSE). Stays open until the client disconnects. Heartbeat events are sent every 15 seconds.

**Auth:** JWT

**Query parameters:**

| Parameter | Description |
|---|---|
| `team` | Filter by team name |
| `service` | Filter by service name |
| `model` | Filter by model (substring) |
| `min_cost` | Minimum cost in microdollars |

**Response:** `Content-Type: text/event-stream`

**Event types:**

Request event:

```
data: {"type":"request","timestamp":"15:42:01","team":"platform","service":"chat-api","model":"gpt-4o","tokens_input":823,"tokens_output":411,"cost_usd":"$0.0048","latency_ms":842}
```

Alert event:

```
data: {"type":"alert","severity":"warning","message":"Budget organization at 75% utilization"}
```

Heartbeat (every 15s):

```
data: {"type":"heartbeat"}
```

```bash
curl -N https://api.tolvyn.io/v1/stream/tail \
  -H "Authorization: Bearer <token>" \
  -H "Accept: text/event-stream"
```

---

## Operator API

The Operator API is for platform administration. Requires a static `TOLVYN_OPERATOR_TOKEN` configured on the server.

### POST /v1/operator/auth/verify

Verify an operator token. Accepts token from `Authorization: Bearer` header or `{"token": "..."}` body.

**Auth:** None (this is the auth endpoint)

**Response 200:**

```json
{ "valid": true }
```

---

### GET /v1/operator/tenants

List all tenants with usage summaries.

**Auth:** Operator token

**Response 200:** array of tenant objects including `total_requests`, `total_cost_usd`, `mrr_usd`, and `subscription` detail.

---

### GET /v1/operator/tenants/{id}

Get a single tenant with recent requests and API keys.

**Auth:** Operator token

**Response 200:**

```json
{
  "id": "550e8400-...",
  "name": "Alice",
  "email": "alice@example.com",
  "plan_tier": "starter",
  "status": "active",
  "total_requests": 48231,
  "total_cost_microdollars": 142830000,
  "total_cost_usd": "$142.8300",
  "subscription": { ... },
  "recent_requests": [ ... ],
  "api_keys": [ ... ]
}
```

---

### PUT /v1/operator/tenants/{id}

Update tenant plan tier or status. Automatically syncs subscription limits from the canonical plan configuration.

**Auth:** Operator token

**Request body:**

```json
{
  "plan_tier": "team",
  "status": "active"
}
```

**Response 200:**

```json
{ "status": "updated" }
```

---

### GET /v1/operator/stats

Platform-wide statistics for today.

**Auth:** Operator token

**Response 200:**

```json
{
  "total_tenants": 1240,
  "active_tenants": 987,
  "total_requests_today": 284120,
  "total_cost_today_microdollars": 8723400,
  "total_cost_today_usd": "$8.7234",
  "mrr_microdollars": 48000000000,
  "mrr_usd": "$48000.0000",
  "top_models_by_request_count": [
    {
      "model_id": "gpt-4o",
      "provider": "openai",
      "request_count": 142000,
      "cost_today_usd": "$5.1200"
    }
  ]
}
```

---

### GET /v1/operator/models

List all models with pricing.

**Auth:** Operator token

**Response 200:**

```json
[
  {
    "id": "...",
    "provider": "openai",
    "model_id": "gpt-4o",
    "display_name": "GPT-4o",
    "modality": "text",
    "pricing_input_per_mtok": "2.500000",
    "pricing_output_per_mtok": "10.000000",
    "pricing_cached_per_mtok": "1.250000",
    "context_window": null
  }
]
```

---

### POST /v1/operator/models

Add a new model with pricing.

**Auth:** Operator token

**Request body:**

```json
{
  "provider": "openai",
  "model_id": "gpt-5",
  "display_name": "GPT-5",
  "modality": "text",
  "pricing_input_per_mtok": 10.0,
  "pricing_output_per_mtok": 40.0,
  "pricing_cached_per_mtok": 2.5,
  "context_window": 128000
}
```

**Response 201:**

```json
{ "id": "...", "model_id": "gpt-5" }
```

---

### PUT /v1/operator/models/{id}

Update model pricing. All pricing fields are optional.

**Auth:** Operator token

**Request body:**

```json
{
  "pricing_input_per_mtok": 10.0,
  "pricing_output_per_mtok": 40.0,
  "pricing_cached_per_mtok": 2.5
}
```

**Response 200:**

```json
{ "status": "updated" }
```

Changes are recorded in the audit log.

---

## Errors

All errors follow this schema:

```json
{
  "error": "error_code",
  "message": "Human-readable description"
}
```

| HTTP Status | `error` code | Meaning |
|---|---|---|
| 400 | `invalid_body` | Malformed JSON request body |
| 400 | `name_required` | Required name field missing |
| 400 | `invalid_email` | Email format invalid |
| 400 | `password_too_short` | Password must be at least 8 characters |
| 400 | `email_taken` | Email already registered |
| 400 | `provider_required` | Provider field missing |
| 400 | `key_required` | Key field missing |
| 400 | `required_fields` | Required fields missing |
| 401 | `missing_auth` | Authorization header missing |
| 401 | `invalid_credentials` | Invalid email or password |
| 401 | `invalid_token` | Token invalid or expired |
| 404 | `not_found` | Resource not found |
| 429 | `budget_exceeded` | Hard budget limit exceeded |
| 500 | `internal_error` | Internal server error |
| 503 | `operator_disabled` | Operator API not configured |
| 503 | `streaming_unsupported` | Server does not support streaming |
