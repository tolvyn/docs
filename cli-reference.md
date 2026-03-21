# CLI Reference

Complete reference for the `tolvyn` command-line tool.

© 2026 TOLVYN

---

## 1. Installation

Download the binary for your platform from [docs.tolvyn.io/install](https://docs.tolvyn.io/install) <!-- verify --> or use the install script:

```bash
curl -fsSL https://releases.tolvyn.io/install.sh | sh
```

Verify:

```bash
tolvyn --help
```

---

## 2. Configuration

The CLI stores configuration in `~/.tolvyn/config.json`:

```json
{
  "api_url": "https://api.tolvyn.io",
  "token": "eyJhbGci...",
  "default_environment": "production"
}
```

| Field | Description |
|---|---|
| `api_url` | API base URL (default: `https://api.tolvyn.io`) |
| `token` | JWT authentication token (set by `tolvyn login` or `tolvyn init`) |
| `default_environment` | Default environment for new API keys |

The file is created with mode `0600` (owner-readable only).

---

## 3. Global Flags

These flags work with every command:

| Flag | Description |
|---|---|
| `--api-url <url>` | Override the API URL for this invocation |
| `--json` | Output raw JSON instead of formatted tables |
| `--no-color` | Disable colored output |

---

## 4. Commands

### `tolvyn init`

Configure the TOLVYN CLI interactively. Prompts for API URL, email, and password, then saves credentials to `~/.tolvyn/config.json`.

```bash
tolvyn init
```

Use this for first-time setup. For subsequent logins, use `tolvyn login`.

---

### `tolvyn login`

Authenticate with your TOLVYN account. Reads email and password interactively (password input is hidden).

```bash
tolvyn login
```

Saves the JWT token to `~/.tolvyn/config.json`. Tokens expire after 24 hours; run `tolvyn login` again to refresh.

---

### `tolvyn logout`

Clear stored credentials (removes the token from `~/.tolvyn/config.json`).

```bash
tolvyn logout
```

```
Logged out.
```

---

### `tolvyn status`

Check API connectivity and authentication status.

```bash
tolvyn status
```

Example output:

```
API:      https://api.tolvyn.io  [OK]
Database: ok
Version:  1.0.0
Latency:  23ms
Auth:     Authenticated as: alice@example.com
```

With `--json`:

```bash
tolvyn status --json
```

Returns the raw `/health` JSON response.

---

### `tolvyn tail`

Stream live AI requests in real-time over Server-Sent Events. Press `Ctrl+C` to stop.

```bash
tolvyn tail [flags]
```

**Flags:**

| Flag | Description |
|---|---|
| `--team <name>` | Filter by team name |
| `--service <name>` | Filter by service name |
| `--model <substring>` | Filter by model (substring match) |
| `--min-cost <usd>` | Minimum cost in USD to display |
| `--no-alerts` | Suppress alert events |

**Example:**

```bash
tolvyn tail --team platform --model gpt-4o
```

**Output format:**

```
TIME     | TEAM/SERVICE          | MODEL            |   TOKENS |     COST |  LATENCY
─────────┼───────────────────────┼──────────────────┼──────────┼──────────┼─────────
15:42:01 | platform/chat-api     | gpt-4o           |    1,234 |  $0.0048 |    842ms
15:42:09 | platform/chat-api     | gpt-4o-mini      |      521 |  $0.0003 |    312ms
[ALERT] Budget organization at 75% utilization
```

Columns:
- `TIME` — 8-character timestamp (HH:MM:SS)
- `TEAM/SERVICE` — `team/service` combined, truncated to 22 chars
- `MODEL` — model ID, truncated to 16 chars
- `TOKENS` — total tokens (input + output), comma-formatted
- `COST` — cost in USD
- `LATENCY` — total latency in milliseconds

**Reconnection behavior:** the CLI retries up to 3 times on connection loss, with a 5-second delay between attempts. After 3 failures it exits with an error.

---

### `tolvyn cost`

Show a spend summary. Calls `GET /v1/usage/summary`.

```bash
tolvyn cost [flags]
```

**Flags:**

| Flag | Description |
|---|---|
| `--from <YYYY-MM-DD>` | Start date |
| `--to <YYYY-MM-DD>` | End date |
| `--team <id>` | Filter by team ID |
| `--model <id>` | Filter by model |

**Example:**

```bash
tolvyn cost --from 2026-03-01 --to 2026-03-31
```

Example output:

```
Period:              2026-03-01 to 2026-03-31
Total Spend:         $142.8300
Total Requests:      48,231
Avg Cost/Req:        $0.0030

Top Models:
  gpt-4o               $98.2100      68.8%  32,100 reqs
  claude-sonnet-4-6    $31.4200      22.0%  10,240 reqs
  gpt-4o-mini          $13.1900       9.2%   5,891 reqs

Top Teams:
  platform             $89.3200      62.5%
  ml-research          $53.5100      37.5%
```

---

### `tolvyn requests`

Show request history. Calls `GET /v1/usage/requests`.

```bash
tolvyn requests [flags]
```

**Flags:**

| Flag | Description |
|---|---|
| `--team <id>` | Filter by team ID |
| `--model <id>` | Filter by model |
| `--limit <n>` | Number of rows (default: 20, max: 100) |
| `--from <YYYY-MM-DD>` | Start date |
| `--to <YYYY-MM-DD>` | End date |

**Example:**

```bash
tolvyn requests --limit 50 --team platform
```

Example output:

```
TIME              | TEAM/SERVICE     | MODEL          |  TOKENS |     COST | LATENCY
2026-03-21 15:42 | platform/chat    | gpt-4o         |   1,234 |  $0.0048 |   842ms
2026-03-21 15:41 | platform/chat    | gpt-4o-mini    |     521 |  $0.0003 |   312ms
```

---

### `tolvyn keys`

Manage TOLVYN API keys.

#### `tolvyn keys list`

List all API keys for your account.

```bash
tolvyn keys list
```

Example output:

```
NAME             PREFIX               ENV            LAST USED
prod-api         tlv_live_xK3m...     production     2026-03-21 15:42
dev-testing      tlv_live_aB9p...     production     never
```

#### `tolvyn keys create`

Create a new API key. The full key value is shown once and cannot be retrieved again.

```bash
tolvyn keys create --name <name> [--env <env>] [--team <id>]
```

**Flags:**

| Flag | Description |
|---|---|
| `--name <name>` | Key name (required) |
| `--env <env>` | Environment (default: `production`) |
| `--team <id>` | Associate key with a team ID |

**Example:**

```bash
tolvyn keys create --name prod-api --env production
```

```
New API key created: prod-api

tlv_live_xK3mQ9pR2nL7vB4w...

Save this key — it will not be shown again.
```

#### `tolvyn keys revoke`

Revoke an API key by ID. Prompts for confirmation.

```bash
tolvyn keys revoke <id>
```

```bash
tolvyn keys revoke 550e8400-e29b-41d4-a716-446655440000
Revoke key 550e8400-e29b-41d4-a716-446655440000? [y/N]: y
Key revoked.
```

---

### `tolvyn providers`

Manage provider API keys (stored encrypted server-side).

#### `tolvyn providers list`

List connected providers.

```bash
tolvyn providers list
```

```
PROVIDER      ID                                    VER    ADDED
openai        b2c3d4e5-...                          1      2026-03-01 09:00
anthropic     c3d4e5f6-...                          2      2026-03-15 14:30
```

#### `tolvyn providers add`

Add or rotate a provider API key. Input is hidden.

```bash
tolvyn providers add <provider>
```

Supported providers: `openai`, `anthropic`, `google`.

```bash
tolvyn providers add openai
Enter openai API key (input hidden):
✓ openai provider key stored (id: b2c3d4e5-...)
```

---

### `tolvyn teams`

Manage teams for attribution and budget scoping.

#### `tolvyn teams list`

```bash
tolvyn teams list
```

```
NAME                 COST CENTER    CREATED
platform             ENG-001        2026-01-15
ml-research          RES-002        2026-02-01
```

#### `tolvyn teams create`

```bash
tolvyn teams create --name <name> [--cost-center <code>]
```

**Flags:**

| Flag | Description |
|---|---|
| `--name <name>` | Team name (required) |
| `--cost-center <code>` | Cost center code |

**Example:**

```bash
tolvyn teams create --name platform --cost-center ENG-001
Team created: platform (id: a1b2c3d4-...)
```

---

### `tolvyn budgets`

Manage spending budgets.

#### `tolvyn budgets list`

```bash
tolvyn budgets list
```

```
SCOPE                  MODE    LIMIT         SPENT         UTIL       PERIOD
organization           soft    $500.00       $142.83       28.6%      monthly
team (platform-uuid)   hard    $200.00       $89.32        44.7%      monthly
```

Utilization is color-coded: green below 75%, yellow 75–90%, red 90%+.

#### `tolvyn budgets create`

```bash
tolvyn budgets create --scope <org|team|service> --amount <usd> [flags]
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--scope <org\|team\|service>` | `org` | Budget scope |
| `--team <name>` | — | Team name (if `--scope team`) |
| `--service <name>` | — | Service name (if `--scope service`) |
| `--amount <usd>` | required | Budget limit in USD |
| `--period <period>` | `monthly` | Period: `monthly`, `weekly`, or `daily` |
| `--mode <mode>` | `soft` | Mode: `soft` (alert only) or `hard` (block) |

**Examples:**

```bash
# Organization-wide monthly soft budget
tolvyn budgets create --scope org --amount 500 --period monthly --mode soft

# Team hard budget
tolvyn budgets create --scope team --team platform --amount 200 --period monthly --mode hard

# Daily hard budget
tolvyn budgets create --scope org --amount 50 --period daily --mode hard
```

---

### `tolvyn kill`

Emergency spend kill switch. Blocks all AI requests for a team immediately by creating a $0.000001 hard monthly budget.

```bash
tolvyn kill --team <name>
```

**Flag:**

| Flag | Description |
|---|---|
| `--team <name>` | Team name to block (required) |

**Example:**

```bash
tolvyn kill --team ml-research
Block all AI requests for team "ml-research"? This takes effect immediately. [y/N]: y
KILL SWITCH ACTIVATED — all requests for team "ml-research" are now blocked.
Budget ID: f4a1b2c3-...
To undo: DELETE /v1/budgets/f4a1b2c3-...
```

**How it works:** resolves the team name to a UUID, then creates a `scope_type=team`, `mode=hard`, `amount_usd=0.000001`, `period=monthly` budget. Any subsequent request from that team exceeds the budget and receives HTTP 429.

**To undo:** delete the budget via the API:

```bash
curl -X DELETE https://api.tolvyn.io/v1/budgets/<budget-id> \
  -H "Authorization: Bearer <jwt-token>"
```

Or from the dashboard at [app.tolvyn.io](https://app.tolvyn.io).
