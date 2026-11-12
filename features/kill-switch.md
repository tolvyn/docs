# Kill Switch

The kill switch unconditionally blocks AI requests at the TOLVYN proxy. Unlike a budget (which tracks spend), a kill switch is a flag that immediately rejects matching requests with `HTTP 451 Unavailable For Legal Reasons` until it is explicitly deactivated.

Use it for incident response: a runaway agent, a leaked API key, a service in a bad state, or any time you need to stop the bleeding fast.

---

## Overview

| Property | Value |
|---|---|
| Block status | `HTTP 451` |
| Order in proxy | First — runs before budget check |
| Check | Held in memory: no database round-trip per request |
| Takes effect | Immediately on activate and lift (dashboard, API or CLI) |
| Persistence | Survives a server restart |
| Scope | `all`, `team`, `service`, `agent`, `api_key` |
| Auto-deactivation | None — must be deactivated manually |

---

## Scope types

| Scope | Matches when |
|---|---|
| `all` | Any request from this tenant — global kill |
| `team` | The request's team is the stored team. Arm it with the team's name (any case) or its id: the id is what is stored, listed, returned in the 451 and matched, so the kill survives a rename. A name no team has is refused with `400 team_not_found`; a name two teams share, with `409 ambiguous_team_name` (use the id from `GET /v1/teams`) |
| `service` | The service the proxy settled on for the request equals `scope_value`, exactly (case counts): the API key's bound service when it has one, or else the `X-Tolvyn-Service` header. On a bound key a different header is ignored, so it cannot step around the kill |
| `agent` | The agent the proxy settled on for the request equals `scope_value`, exactly (case counts): the API key's bound agent when it has one, or else the `X-Tolvyn-Agent` header. On a bound key a different header is ignored |
| `api_key` | Request's TOLVYN API key prefix (first 12 chars) equals `scope_value` |


### Check order

The kill check short-circuits on the first match. Scopes are checked in strict priority order: `all` → `team` → `service` → `agent` → `api_key`. A higher-priority scope match wins regardless of when each kill was activated:

```go
for i := range entries {
    e := &entries[i]
    switch e.ScopeType {
    case "all":
        return true, e
    case "team":
        if teamID != "" && e.ScopeValue == teamID { return true, e }
    case "service":
        if serviceName != "" && e.ScopeValue == serviceName { return true, e }
    case "agent":
        if agentName != "" && e.ScopeValue == agentName { return true, e }
    case "api_key":
        if apiKeyPrefix != "" && e.ScopeValue == apiKeyPrefix { return true, e }
    }
}
```

Priority is by scope type, not activation time. A team-scope kill activated at 09:00 beats an agent-scope kill activated at 08:30 if both would match the request — `team` is checked before `agent`. An `all`-scope kill always wins.

---

## Order in the proxy

The kill check is the **first** authorization gate after JWT validation and tenant-scope resolution. It runs **before** the budget check.

```
1. JWT validation
2. Tenant + scope extraction (team, service, agent, API key)
3. Kill switch check     ← rejects with 451
4. Provider key lookup
5. Budget check          ← rejects with 429
6. Forward to provider
```


### HTTP 451 response body

When the kill matches, the proxy returns:

```json
{
  "error": "kill_switch_active",
  "message": "<reason or 'Kill switch active'>",
  "kill_id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "scope_type": "team",
  "scope_value": "3f6b2c1e-8d4a-4b7f-9e21-5a0c7d9e4f13",
  "activated_at": "2026-05-17T15:55:00Z"
}
```

Status: `451 Unavailable For Legal Reasons`. For a team kill, `scope_value` is the team's id.

---

## How it holds

- **No database round-trip per request.** Active kill switches are held in memory, so the proxy decides without querying the database. No figure is published for how long the check takes; the point is the mechanism.
- **Immediate.** Activating or lifting a kill switch from the dashboard, the API or the CLI takes effect for the next request.
- **Survives a restart.** Active kill switches are stored durably and loaded again when the server starts. If that load fails, the server keeps serving (refusing to start would take every call down), logs it as an incident, and retries at once and then every 60 seconds until the load succeeds.
- **Reloaded every 60 seconds** as a safety net, so the proxy's view does not lag the stored state by more than 60 seconds.

---

## Activating

### Dashboard

**Kill switch** page (or the red banner button on any page during an incident):

1. Pick a scope
2. Enter the target (team name, service name, etc.) — or leave blank for `all`
3. Optional: enter a reason (recorded in audit log + included in 451 response body)
4. Click **Activate** — confirmation prompt
5. Matching requests start receiving 451 immediately

### CLI

```bash
tolvyn kill --scope team --target backend --reason "runaway agent"
```

Flags:

| Flag | Default | Description |
|---|---|---|
| `--scope` | — (required) | `team`, `service`, `agent`, `api_key`, or `all` |
| `--target` | — | Target value; required unless `--scope=all` |
| `--reason` | — | Recorded in audit log and shown in 451 response |

The CLI prompts for confirmation before activating. Output:

```
Kill team "backend"? This will immediately block all AI requests. [y/N]: y
✓ Kill switch activated. All matching AI requests are now blocked.
  ID:    a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
  Scope: team / 3f6b2c1e-8d4a-4b7f-9e21-5a0c7d9e4f13
  Reason: runaway agent

To undo: tolvyn kill undo a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

The scope line shows the team's id: the name you typed was looked up and the id is what was armed.

### API

```bash
curl -X POST https://api.tolvyn.io/v1/kill \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_type":  "team",
    "scope_value": "backend",
    "reason":      "runaway agent loop"
  }'
```

Request fields:

| Field | Type | Required |
|---|---|---|
| `scope_type` | string | yes — `team`, `service`, `agent`, `api_key`, `all` |
| `scope_value` | string | yes (or `*` for `all`) |
| `reason` | string | no |

Response (201 Created):

```json
{
  "id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "scope_type": "team",
  "scope_value": "3f6b2c1e-8d4a-4b7f-9e21-5a0c7d9e4f13",
  "reason": "runaway agent loop",
  "activated_by": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "activated_at": "2026-05-17T15:55:00Z"
}
```

For a team, `scope_value` in the response is the team's id, whichever form you sent.

Tenants are capped at **10 active kill switches** at any time. Attempting an 11th returns `400 limit_exceeded`.

---

## Listing active kills

### CLI

```bash
tolvyn kill list
```

Aliases: `tolvyn kill ls`.

```
ID         SCOPE    TARGET                                 REASON                 ACTIVATED
a1b2c3d4…  team     3f6b2c1e-8d4a-4b7f-9e21-5a0c7d9e4f13   runaway agent loop     2026-05-17 15:55

1 active kill switch(es)
```

The table shortens each id to its first 8 characters. `tolvyn kill undo` needs the full id: get it with `tolvyn kill list --json`.

### API

```bash
curl https://api.tolvyn.io/v1/kill -H "Authorization: Bearer <jwt>"
```

Returns the active kill switches only, newest first, as `{"kills": [...], "total": n}`. A lifted one is kept in the database for the audit trail but is not listed.

---

## Deactivating

### CLI

```bash
tolvyn kill undo a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

Aliases: `tolvyn kill deactivate`, `tolvyn kill rm`. It takes the full id, not the 8 characters the `list` table shows: `tolvyn kill list --json` prints it.

```
✓ Kill switch a1b2c3d4-... deactivated. AI requests are flowing again.
```

### API

```bash
curl -X DELETE https://api.tolvyn.io/v1/kill/<kill-id> \
  -H "Authorization: Bearer <jwt>"
```

Returns `204 No Content`. An id that is not a full kill switch id is refused with `400 invalid_kill_id` and nothing is lifted; a full id that is unknown or already lifted is `404 not_found`. The kill switch is marked as lifted, with when and by whom, for the audit trail; it is not deleted — kill history is permanent.

The lift takes effect for the next request: it stops receiving 451.

---

## Kill vs budget — comparison

| | Kill switch | Budget (hard mode) |
|---|---|---|
| Purpose | Unconditional block, fast | Spend cap with running counter |
| Trigger | Active state | `current_spend >= amount` |
| HTTP status | `451 Unavailable For Legal Reasons` | `429 Too Many Requests` |
| Error code | `kill_switch_active` | `budget_exceeded` |
| Order in proxy | First | After the kill-switch check |
| Check | In memory, no database round-trip | Reads the running total per request |
| Auto-reset | Never | On period rollover |
| Use case | Incident response | Long-term cost control |
| Limit per tenant | 10 active | No limit |
| Survives server restart | Yes | Yes |

Both can be active simultaneously and they answer different questions: "should this request happen *at all*?" (kill) vs. "have we spent too much?" (budget).

---

## Operational pattern

Recommended kill-switch playbook during an incident:

1. **Scope as narrowly as possible.** Start with `--scope agent --target <agent name>` or `--scope api_key --target <prefix>` — these are surgical. Avoid `--scope all` unless you genuinely cannot identify the cause.
2. **Always record a reason.** It surfaces in the 451 response, gives downstream consumers (and on-call engineers) context, and lands in the audit log.
3. **Deactivate as soon as the cause is fixed.** Kill switches are not auto-deactivating. A forgotten `--scope all` will leave the entire org's AI traffic blocked indefinitely.
4. **Verify before deactivating.** Once you `kill undo`, the next request will hit the provider. Make sure the underlying issue (bug, leaked key, broken loop) is resolved first.

---

## See also

- [Budgets](budgets.md) — spend-based limits
- [Alerts](alerts.md) — incident detection
- [API Reference: Kill switches](../reference/api.md#kill-switches)
- [CLI Reference: `tolvyn kill`](../reference/cli.md#tolvyn-kill)
