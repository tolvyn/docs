# Immutable Ledger

The ledger is TOLVYN's answer to the question: **how do I prove what my AI costs were on a specific date, to an auditor who doesn't trust my dashboard?**

Every request that flows through TOLVYN is recorded in a SHA-256 hash-chained, HMAC-signed ledger. The chain is verifiable end-to-end at any time. Tampering with any historical record breaks the chain at that point — detectably, without external consensus, in milliseconds.

This is what separates TOLVYN from "AI usage analytics." Analytics dashboards can be edited. Audit ledgers cannot.

---

## Why hash-chaining matters for AI governance

Three categories of question that traditional logs cannot answer:

- *"Did your finance team alter the spend numbers before the audit?"* — Application logs are mutable. They were designed for debugging, not financial evidence.
- *"Prove your March AI cost is exactly the value you billed to the customer."* — Aggregating from mutable logs is a sticky note, not an audit trail.
- *"Show me the exact request that was blocked when our budget hit 100%."* — The ledger records the enforcement decision (`allow` / `block`) alongside the cost.

The TOLVYN ledger is built on the premise that AI spend has crossed the threshold where standard observability stops being adequate. When your AI bill is bigger than your CI/CD bill, you need financial-grade evidence.

---

## How it works

### Genesis block

Each tenant's chain begins with a synthetic predecessor: `previous_hash` of the **very first record** is `SHA-256(tenantID)` rendered as hex. The hash is taken over the tenant ID's **text form** — the ASCII bytes of the UUID string, hyphens included — not over the 16 raw bytes a UUID decodes to. A verifier that parses the UUID first will derive a different genesis and conclude the chain is broken. The genesis hash anchors the chain — verifying from sequence 1 requires re-deriving it:

```go
func genesisHash(tenantID string) string {
    h := sha256.Sum256([]byte(tenantID))
    return hex.EncodeToString(h[:])
}
```

Two tenants that happened to start with the same first proxied request would still produce different chains, because the genesis hashes differ.

### Record structure

Every ledger row's `record_hash` is `SHA-256` of a **canonical JSON serialization** of the 14 fields below, in this exact order:

```json
{
  "tenant_id": "9f8d6a4e-2c33-4b1e-a3f0-21d8f2c4b5c1",
  "sequence_number": 1843,
  "previous_hash": "<hex of previous record_hash>",
  "request_id": "<UUID of the metered request>",
  "cost_microdollars": 8400,
  "hierarchy_path": "/backend/summarizer",
  "provider": "openai",
  "model_id": "gpt-4o-2024-08-06",
  "model_family": "gpt-4o",
  "modality": "text",
  "tokens_input": 120,
  "tokens_output": 35,
  "budget_status": "ok",
  "enforcement_action": "allow"
}
```

**The block above is pretty-printed for reading. It is NOT the byte sequence that gets hashed.**
Getting this wrong is the single most common reason an independent verifier fails to reproduce our
hashes, so the canonical form is worth stating exactly:

- **No whitespace at all.** No newlines, no indentation, no space after `:` or `,`. The hashed bytes
  are one line beginning `{"tenant_id":"…`.
- **Fields in the order shown**, which is the struct's declaration order — *not* alphabetical. Do not
  sort the keys. A JSON canonicalization library that sorts keys (RFC 8785/JCS, for one) will fail on
  every record.
- **`&`, `<` and `>` are escaped** as `\u0026`, `\u003c` and `\u003e`. This is Go's default HTML
  escaping in `encoding/json`, and it applies to string values such as `hierarchy_path`. A path like
  `/r&d/summarizer` is hashed as `/r\u0026d/summarizer`.
- **All 14 fields are always present**, including empty strings and zeros. Nothing is omitted.

Given those rules, `SHA-256` over the resulting bytes reproduces `record_hash` exactly.

### Hash chain

Each record stores the hash of the previous one. The chain looks like:

```
genesisHash(tenant_id) → record[1].record_hash → record[2].record_hash → ... → record[N].record_hash
                          ↑                       ↑
                          stored as record[2].previous_hash
                          stored as record[3].previous_hash
```

Any tampering with record `K` produces a new `record_hash[K]`. That breaks the chain because `record[K+1].previous_hash` no longer matches. Verification immediately flags the seam.

### HMAC signing

`record_hash` is then signed with `HMAC-SHA256` keyed on the signing secret:

```go
mac := hmac.New(sha256.New, secret)
mac.Write([]byte(recordHash))
hmacSignature := hex.EncodeToString(mac.Sum(nil))
```

The HMAC adds a second layer of defense: even an attacker with direct database write access cannot forge a valid record without also knowing the secret. The SHA-256 chain catches accidental or external tampering; the HMAC catches insider tampering. Different threat models, both addressed.

The signing secret must be at least 32 bytes. The server refuses to start without it set.

### Advisory lock for sequence integrity

Sequence numbers must be gap-free per tenant for verification to work. Concurrent ledger appends for the same tenant could otherwise race and produce duplicates or gaps.

The fix: a per-tenant PostgreSQL advisory lock:

```go
lockKey := advisoryLockKey(tenantID)  // first 8 bytes of SHA-256(tenantID) as int64
if _, err := tx.ExecContext(ctx, "SELECT pg_advisory_xact_lock($1)", lockKey); err != nil {
    return err
}
```

A per-tenant advisory lock serializes ledger appends for the same tenant **within each transaction**. The lock releases automatically on transaction commit or rollback. Across tenants, no contention — each tenant's chain has its own lock key.

This means a single tenant making 1,000 concurrent requests will have those 1,000 ledger appends serialized; cross-tenant traffic is unaffected.

---

## What goes in every record

From `recordPayload`:

| Field | Meaning |
|---|---|
| `tenant_id` | Owner — RLS enforces visibility |
| `sequence_number` | Gap-free integer per tenant, starts at 1 |
| `previous_hash` | Hash of the prior record (or `genesisHash` for record 1) |
| `request_id` | UUID linking to the `requests` table |
| `cost_microdollars` | Exact cost as int64 — never a float |
| `hierarchy_path` | `/team/service[/agent]` for forensic drill-down |
| `provider` | `openai`, `anthropic`, `google`, `deepseek` |
| `model_id` | Exact model used (e.g. `gpt-4o-2024-08-06`) |
| `model_family` | Normalized family (e.g. `gpt-4o`) |
| `modality` | `text` — the only value currently written to the ledger |
| `tokens_input` / `tokens_output` | Provider-reported counts |
| `budget_status` | `ok`, `warning`, `exceeded` at request time |
| `enforcement_action` | `allow` on every record — see the note below |

**A blocked request does not appear in the ledger at all.** The ledger is written only after a
request has reached the provider and been metered, and a budget block returns its `429` before that
point — so no row is created and `enforcement_action` is `allow` on every record it does contain.
The field is retained because it is part of the hashed payload and removing it would invalidate every
historical record's hash, not because it varies.

Blocked requests are audited separately from this chain. If you need to prove that spend was
*prevented*, that evidence is in the enforcement audit trail, not here.

---

## Verification

`GET /v1/ledger/verify` walks the chain and re-derives every hash and HMAC. Parameters:

| Param | Default | Description |
|---|---|---|
| `from_seq` | `1` | Lowest sequence to check |
| `to_seq` | current max | Highest sequence to check |

### Successful verification

```bash
curl https://api.tolvyn.io/v1/ledger/verify \
  -H "Authorization: Bearer <jwt>"
```

```json
{
  "valid": true,
  "records_checked": 18432
}
```

### Failed verification

```json
{
  "valid": false,
  "records_checked": 1042,
  "first_invalid_sequence": 1043,
  "reason": "seq 1043: record_hash mismatch (stored a1b2c3..., derived d4e5f6...)"
}
```

`first_invalid_sequence` is the exact sequence where the chain broke. `reason` gives one of:

- `sequence gap: expected N, got M` — a record is missing
- `seq N: previous_hash mismatch` — record `N` claims a prior hash that doesn't match record `N-1`'s `record_hash`
- `seq N: record_hash mismatch` — re-deriving from the stored payload produces a different hash (the payload was edited after insertion)
- `seq N: HMAC mismatch` — record_hash is consistent but the HMAC doesn't verify (signing key changed, or HMAC was forged with the wrong secret)

Anything other than `valid: true` means the chain has been compromised at or after `first_invalid_sequence`. Investigate immediately — at minimum, restore from a verified backup and rotate the signing secret.

---

## Exporting the ledger

`GET /v1/ledger?format=csv` streams every column to a CSV file. Includes `record_hash`, `previous_hash`, and `hmac_signature` so the export can be verified **offline** against the server's signing secret without ever talking to TOLVYN again.

```bash
curl "https://api.tolvyn.io/v1/ledger?format=csv&from=2026-05-01T00:00:00Z&to=2026-06-01T00:00:00Z" \
  -H "Authorization: Bearer <jwt>" \
  > tolvyn-ledger-2026-05.csv
```

Columns in the export:

```
sequence_number, created_at, request_id, provider, model_id, model_family,
cost_usd, hierarchy_path, budget_status, enforcement_action,
tokens_input, tokens_output, record_hash, previous_hash, hmac_signature
```

For auditors: hand them the CSV. **Do not hand over the signing secret.**

They do not need it. The `record_hash` chain is keyed on nothing — an auditor re-derives each hash
from the record's own fields using the canonical form described above, and walks `previous_hash` from
one record to the next, with any SHA-256 library and no secret of yours. That is the whole integrity
proof, and it works without trusting TOLVYN's verify endpoint.

The `hmac_signature` column is a *separate* control with a different purpose: it proves a record was
written by TOLVYN rather than forged by someone with direct database access. It is an internal
control, not auditor evidence. **There is one signing secret per deployment**, so anyone holding it
can mint a valid signature for any record of any tenant — which is exactly what an auditor must not
be able to do, and exactly why handing it over destroys the control it was meant to provide.

On the **Scale** and **Enterprise** plans a signed [evidence package](evidence-packages.md) is
available — the records, a signed manifest, and a standalone verifier the auditor runs themselves. It is the stronger version of
this flow and it also requires no secret. On Free, Starter and Growth the CSV above is the export to
use (CSV export requires Starter or higher).

---

## What ledger integrity proves — and what it does NOT prove

### What it proves

- Every cost figure in the ledger is the value TOLVYN saw when the request was metered. It was not changed retroactively.
- **Within the range you verify, the sequence is contiguous** — nothing was inserted between two records, and nothing was removed from inside the range. Sequence numbers are the check: a gap means a record that once existed is gone, and verification reports it. Note the scope — this is a statement about the range verified, not about the chain as a whole. See *What it does NOT prove*.
- The cost and attribution of every request that **reached a provider** are recorded contemporaneously with that request. Blocked requests are not in this chain (see *What goes in every record*).
- The total cost over any time range can be computed from the verified ledger and is **mathematically equivalent** to the sum of `cost_microdollars` across those records.

### What it does NOT prove

- **The content of prompts or responses.** TOLVYN does not store prompt or response text. The ledger proves what model was called and what it cost, not what was said.
- **That the provider's invoice matches.** TOLVYN's view is the proxy's view. Requests that bypassed the proxy never reach the ledger. Use [Reconciliation](reconciliation.md) to spot-check against provider invoices.
- **The fairness of provider pricing.** The ledger records what TOLVYN was told the cost was at the time of metering, using prices in effect at that moment. It does not validate whether the provider charged correctly.
- **That the signing secret was never leaked.** Rotation invalidates the HMAC of all prior records (they no longer verify with the new secret). Don't rotate the signing secret without an explicit re-signing plan, or you destroy your own audit trail.
- **That the ledger is COMPLETE.** This is the limit most easily read past. A verified chain proves
  the records you hold were not altered and that they follow one another. It cannot prove none is
  missing from before the range you verified: a record removed from the *start* of a chain leaves no
  gap between two survivors, only a chain that begins later than it once did.
- **That records are kept forever.** They are not — see [Retention and the verified
  range](#retention-and-the-verified-range).

---

## Retention and the verified range

Ledger records do not live forever. Each plan has a data retention window, and records older than
that window are removed:

| Plan | Ledger retention |
|---|---|
| Free | 7 days |
| Starter | 30 days |
| Growth | 90 days |
| Scale | 365 days |
| Enterprise | Unlimited |

Retention runs nightly and removes the **oldest** records first, so a chain past its window begins at
a higher sequence number than it originally did. This does not weaken what remains: every record
still present is internally verifiable and its sequence numbers are still contiguous. What left the
window is simply gone — and, as above, a chain truncated at the start leaves nothing behind for
verification to detect.

**Two consequences worth planning around.** If you must retain records for a fixed period — whatever
your regulator, auditor or contracts require — pick a plan whose window covers it, and check the
number above against your own obligation rather than assuming the default is enough. And upgrading a
plan does not bring back records that have already aged out: a longer window applies from the point
you have it, not retroactively. Export before records age out if you need to hold them longer; the
ledger CSV export is available from **Starter** up, and signed evidence packages on **Scale** and
**Enterprise**.

---

## Compliance use cases

### Audit evidence

For an external auditor (SOC 2, ISO 27001, financial audit): export the ledger CSV for the audit
period and give them the canonical-form rules above so they can re-derive the chain themselves. **The
signing secret is not part of what you hand over** — the hash chain verifies without it. On Scale and
Enterprise, a signed [evidence package](evidence-packages.md) bundles the records, a signed
manifest and a standalone verifier, which is the cleaner hand-off. Either way the auditor verifies independently, without
trusting your dashboard.

Check first that the audit period fits inside your plan's [retention
window](#retention-and-the-verified-range) — records older than the window are already gone, and no
export recovers them.

### CFO reporting

When the CFO asks "did we really spend $12K on AI in March?", run `GET /v1/ledger/verify` for March's sequence range. If `valid: true`, the sum of `cost_microdollars` in that range *is* the answer — verifiably, mathematically.

### Incident response

When something looks wrong on the dashboard, run a chain verification first. If the chain is valid,
the dashboard is reflecting the truth and the issue is upstream, in metering or pricing.

If verification reports invalid, check *which kind* of failure it is before concluding anything about
your data:

- **A `previous_hash` mismatch on the first record of the range, and only there.** Usually a missing
  anchor rather than tampering. Verification checks the first record against the hash of the record
  immediately before it; if that predecessor is unavailable — you started the range mid-chain, or it
  aged out of your retention window — there is nothing to anchor against and verification stops
  there. Re-run starting **one sequence number above the lowest record you still hold**, so the first
  record in the range has a real predecessor. If that passes, everything you hold is intact. Starting
  from the lowest surviving record will *not* clear it: that record's predecessor is precisely what is
  missing.
- **A mismatch in the middle of the range, or a sequence gap.** This one is real. Something altered or
  removed records that should be there. Restore from backup and investigate.

### Customer audit

For SaaS customers who need a verified report of AI usage on their behalf, filter the ledger by `hierarchy_path` (which encodes attribution) or `X-Tolvyn-End-Customer` (joined via `request_id` → `requests` table). Export the slice — the `record_hash`, `previous_hash` and `hmac_signature` columns travel with it — and the customer verifies the chain independently. As with any other hand-off, they verify the hash chain without the signing secret, which is never shared.

---

## Operational notes

- **The ledger is not a queue.** Records are inserted synchronously inside the request's metering
  transaction. There is no separate "ledger lag" — if the request was metered, the ledger row exists.
  Two caveats on reading that as a strict one-to-one pairing. **Retention breaks it:** the nightly
  sweep deletes from `ledger_records` and from `requests` as separate statements, so the pairing
  does not survive a record ageing out, and does not hold across the two deletes. **And a ledger row
  attests that a request was metered, not that it completed:** a response stream truncated partway —
  by a provider reset or a timeout — is metered on the bytes that arrived and produces a record
  indistinguishable from a complete one.
- **The chain survives row-level security** because ledger appends run within the tenant-scoped transaction context. Cross-tenant reads are still blocked.
- **Backups must be physical, point-in-time, or logical with `pg_dump --no-data --schema-only` plus `pg_dump --data-only --table=ledger_records`.** Avoid tools that re-order rows during export — the SQL primary key on `sequence_number` is unique but verification walks in `sequence_number ASC` order, so any inserter that doesn't preserve order can break verification.
- **Don't manually edit the `ledger_records` table.** The application enforces RLS, advisory locks, and atomic inserts. Direct SQL bypasses all of that and almost certainly breaks the chain.

---

## See also

- [Reconciliation](reconciliation.md) — spot-check the ledger against provider invoices
- [Budgets](budgets.md) — what `budget_status` and `enforcement_action` reflect
- [Evidence Packages](evidence-packages.md) — hand a signed, independently verifiable slice of the chain to an auditor
- [API Reference: Ledger](../reference/api.md#ledger)
