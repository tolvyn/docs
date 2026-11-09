# Events

The **Events** page shows what happened to your account, when, why, for how long, and whose doing it was: the
provider's, TOLVYN's, or your own limits. Every event comes from a record TOLVYN already keeps, so the page and
the rest of the dashboard always agree.

---

## What appears

| Event | What it is |
|---|---|
| Scheduled price change | A price change a provider has announced, shown **before** it takes effect, with the date and the old and new rates |
| Provider model change | A provider serving one model as another from an announced date (for example, Google serving `gemini-2.5-pro` as Gemini 3.8 Flash from 1 December 2026), shown before it happens |
| Price change | A catalogue price that changed |
| Served model changed | One of your services started being served a different model |
| Substitution | Requests served by a different model than they asked for, per day |
| True-up | A request held at the highest price because its model was not yet priced, trued up to its real cost |
| Release | Held charges released because the connection was set to unpriced by design |
| Unknown-model hold | Requests held at the highest price while their model had no price, per day |
| Kill switch / lifted | A kill switch on an agent, service, team, key or the whole account, with who turned it on or off and when |
| Notice | Every alert, email and webhook TOLVYN sent you: how, to whom, when, and whether it went |
| Provider incident | An incident at a provider: recorded by TOLVYN, or the provider's own statement from its status page |
| TOLVYN incident | An incident at TOLVYN, with its cause and post-mortem |

Filter by type, provider, model, agent or service. **Export CSV** downloads the same list with what each event meant
for you; every export is recorded in your audit trail.

---

## What it meant for you

Where an event has a time window, it shows what happened to your traffic in it:

- **Requests affected**: requests sent to that provider (and model) in the window. **Failed**: of those, a provider
  error (5xx), a rate limit (429), a timeout, a connection failure, or a stream that broke.
- **Refused before the model was read**: requests a kill switch or your plan's request quota refused before TOLVYN read
  which model they asked for. They never reached the provider and cost nothing.
- **Spend**: what your requests in the window cost.
- **Billed although it failed**: a call the provider charged for even though it failed, because the error reported
  token usage or a stream broke after tokens were sent. A failure with no usage costs nothing and is not listed.
- **Your client disconnected**: requests your own client stopped waiting for. These are not counted as failures.

Request counts are written once a minute. While TOLVYN's database is unavailable they can lag by more than a minute,
and a restart can lose up to the last minute's counts. Your requests, costs and ledger are never affected.

---

## Provider status feeds

TOLVYN reads **Anthropic's** status page and shows its incidents as the provider's own statement, beside anything TOLVYN
recorded. It is shown on this page only; TOLVYN never sends it to you as a notification.

**Google's Gemini API and DeepSeek publish no status feed TOLVYN can read.** Their incidents appear here only when
TOLVYN records one, or, once TOLVYN detects incidents from traffic, when it detects one.

---

## Kill and Lift

The Agents and Services pages, and the team table on the Overview, have a **Kill** button on each row. It stops every
request from that agent, service or team at once: each is refused with `HTTP 451` before it reaches the provider. The
page asks you to confirm and says what will stop.

- **Kill** needs the write role or above.
- **Lift** needs an admin or the owner.

A button your role cannot use is not shown, and TOLVYN refuses the action anyway. Each kill and lift appears on the
Events page with who did it and when. See [Kill Switch](kill-switch.md).
