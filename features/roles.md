# Member Roles

Every member of a TOLVYN account has one of four roles: **read**, **write**, **admin** or **owner**. Each role can do everything the role before it can. Invite members and change their roles on the **Members** page, which needs admin or owner.

This page says what a read-only member can and cannot do, because that is the role people give to finance, auditors and anyone else who needs to see spend without changing anything.

---

## What a read-only member can do

- **See the account.** Dashboards, usage, requests, budgets, alerts, savings, agents, teams and the pricing changes that affect you.
- **View the ledger, one page at a time, and verify it.** `GET /v1/ledger` and `GET /v1/ledger/verify` work for every role. Verifying checks every hash in the chain.
- **Complete their own onboarding.** See [Onboarding is per user](#onboarding-is-per-user).
- **Export CSV, if an admin has allowed it.** See [the setting](#read-only-members-can-export-csv). It is off by default.

## What a read-only member cannot do

A refused API call answers `403`, and changes nothing. The Budgets page hides its controls from a read-only member; on other pages a control may still show, and using it is refused.

| Action | Least role |
|---|---|
| Create, edit or delete budgets and spend quotas | write |
| Acknowledge alerts, approve or deny budget approvals, dismiss savings | write |
| Upload or delete reconciliation runs | write |
| Activate the kill switch | write |
| Download an [evidence package](evidence-packages.md) | admin |
| Export CSV, while the setting is off | write |
| Change account settings, API keys, provider keys, teams, webhooks and integrations | admin |
| Invite members and change their roles | admin |
| Deactivate the kill switch | admin |
| Billing and deleting the account | owner |

---

## "Read-only members can export CSV"

This setting decides whether read-only members can export CSV from **Requests** and **Ledger Verify**.

- **Off by default.** With it off, a read-only member's export answers `403` with the error `csv_export_restricted`.
- **Who can change it:** admins and owners, on the **Account** page, under **Data export**.
- **Every change is audited.** The audit log records `tenant.read_csv_export_changed` with the old value, the new value and who changed it.
- **Your plan still applies.** CSV export needs Starter or higher, whatever this setting says.
- **Write, admin and owner are not affected.** They can export on any plan that includes CSV.

To change it through the API, as an admin or owner:

```bash
curl -X PUT https://api.tolvyn.io/v1/account/settings/read-csv-export \
  -H "Authorization: Bearer $TOLVYN_JWT" \
  -H "Content-Type: application/json" \
  -d '{"enabled": true}'
```

`GET /v1/account` returns the current value as `read_csv_export`.

---

## Evidence packages are admin-only

A signed [evidence package](evidence-packages.md) contains ledger records for a whole date range, ready to hand to someone outside your company. Downloading one needs admin or owner, and any other role gets `403`. A read-only member can still verify the ledger, with `GET /v1/ledger/verify` or the **Ledger Verify** page.

---

## Onboarding is per user

The welcome dialog, the setup checklist and the product tour belong to each member, not to the account. One member dismissing them changes nothing for anyone else, and every role, read-only included, can dismiss their own. `PATCH /v1/onboarding` only ever writes the signed-in member's own record.
