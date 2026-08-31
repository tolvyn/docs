# Evidence Packages

An evidence package is a signed, self-contained ZIP that lets an auditor verify your AI spend **offline, without a TOLVYN account, without any TOLVYN secret, and without trusting your dashboard**. It contains the ledger records for a period, a signed manifest, the byte-level instructions for checking them by hand, and a working verifier that reproduces every hash from scratch.

The [Immutable Ledger](ledger.md) makes your spend tamper-evident inside TOLVYN. An evidence package is how you hand that property to someone else.

Available on **Scale** and **Enterprise**.

---

## Before you call it: the date range

Two things about the range catch people out, and both are cheaper to learn here than from a package that covers the wrong period.

**`from` and `to` are both required. There are no defaults.** Omitting either returns `400`. A range longer than 366 days returns `400`.

**A date-only value means midnight at the *start* of that day.** Both bounds are compared inclusively, but `to=2026-08-31` resolves to `2026-08-31T00:00:00Z` — so it covers the whole of 30 August and essentially none of 31 August. To cover the whole of August, pass one of:

```
?from=2026-08-01&to=2026-09-01
?from=2026-08-01T00:00:00Z&to=2026-08-31T23:59:59Z
```

This differs from the usage endpoints (`/v1/usage/by-key`, `/v1/usage/by-agent` and the rest), where a date-only `to` is advanced to include the whole named day. Until that inconsistency is resolved, **check `period_from` and `period_to` in `manifest.json` against what you intended** — they record exactly the bounds that were applied, and they are inside the signature.

The **Ledger Verify** page in the TOLVYN dashboard has a date-range picker that handles this for you. If you are producing a package for an auditor rather than automating it, that is the shortest route.

---

## What you get

`GET /v1/evidence-package` returns a ZIP containing five files.

| File | What it is |
|---|---|
| `manifest.json` | The signed attestation: tenant, period, record count, sequence range, genesis hash, chain-head hash, algorithms, TOLVYN's public key, and an Ed25519 signature over all of it. |
| `ledger.json` | Every ledger record in the period, each with its hash and its link to the previous record. |
| `verification.json` | TOLVYN's own chain-verification result at the moment the package was built. |
| `README.txt` | Step-by-step instructions for verifying the package by hand — exact field order, exact escaping rules, and known-answer test vectors to check your implementation against. |
| `verify_evidence.py` | A working reference verifier. Python 3.8+, standard library only, no install, no network. |

The README and the verifier are the point. **The package does not ask an auditor to trust a claim; it gives them the means to check it.**

### What the manifest records

```json
{
  "format_version": "tolvyn-evidence-1",
  "tenant_id": "...",
  "period_from": "2026-08-01T00:00:00Z",
  "period_to":   "2026-09-01T00:00:00Z",
  "record_count": 3,
  "seq_from": 1,
  "seq_to": 3,
  "genesis_hash": "...",
  "chain_head_hash": "...",
  "chain_valid": true,
  "generated_at": "...",
  "hash_algo": "SHA-256",
  "signature_algo": "Ed25519",
  "public_key": "...",
  "signature": "..."
}
```

`format_version` describes **the shape of this ZIP**, not the format of the records inside it. Those are different things — see [Record format versions](#record-format-versions).

---

## Verifying a package

Three commands, offline:

```bash
unzip tolvyn-evidence-<tenant>-20260801-20260901.zip -d evidence/
cd evidence/
python3 verify_evidence.py
```

The verifier exits `0` on success and non-zero on any failure. A failure names the sequence number and what did not match, and never reports partial success as success.

Before trusting it against your data, check it against its own known answers:

```bash
python3 verify_evidence.py --self-test
```

```
ok    genesis                bafde89c041e1756082b933aaf16cad8e65dec48de748479352f657e89dd6da5
ok    plain record           40f8340f397d45ddd44df48c9f65a6cab155c8ee517134d5bed2711c75c983eb
ok    hierarchy with R&D     a907528da0cf5dfa539835bf3f4be056961fdc83cf2be6b5359d37bf88706404
ok    hierarchy with A<B     671d87678c434335f967c9a6705d53f104d2a332c7945b4e728fba7c9da36bed
ok    v2, agent_id null      31a5112569f824ad17d4b51d17fd1442fc3dbc03206380cc755d09f1996be5c3
ok    v2, agent_id set       a9ab16d1a57c8b30731cd425bd52a3df203abc9b941621a9e87de4ef6a5e6281

self-test: 6/6 known answers reproduced
```

These are the same vectors quoted in the package's `README.txt`, and the same ones pinned in TOLVYN's own test suite.

### Verifying by hand

An auditor who prefers not to run our script can reproduce everything from `README.txt`. Two points are worth surfacing here because they are where hand-written verifiers usually diverge:

**Do not use an RFC 8785 / JCS canonicalizer.** "Canonical JSON" is a published standard and reaching for it is the natural move — but RFC 8785 sorts object keys alphabetically, and the record payload is **not** key-sorted. A JCS canonicalizer produces a different SHA-256 for every record, including records with no unusual characters at all.

**The record payload escapes three HTML characters that most JSON libraries leave alone.** Inside string values, `&` becomes `\u0026`, `<` becomes `\u003c`, and `>` becomes `\u003e` (this comes from Go's `encoding/json` defaults). It is load-bearing: a team named `R&D` appears in `hierarchy_path` as `org/<tenant>/team/R\u0026D`, and a verifier that emits a literal `&` computes a different hash and wrongly reports tampering on an intact chain. `U+2028` and `U+2029` are escaped too, and invalid UTF-8 becomes `\ufffd`. **All other non-ASCII passes through as raw UTF-8** and is *not* `\u`-escaped — `café` and `日本` appear literally.

The two known-answer vectors in `README.txt` distinguish these failures: if you reproduce the plain record but not the `R&D` one, your escaping is wrong; if you reproduce neither, check field order first.

### Verifying the public key independently

The manifest carries TOLVYN's public key and `verify_evidence.py` uses that enclosed copy. That proves internal consistency, not authenticity. To check authenticity, fetch the key from a second source and confirm it matches:

```bash
curl https://api.tolvyn.io/v1/evidence-package/pubkey
```

That endpoint is **public and unauthenticated**, deliberately: an auditor verifying a package is by definition someone without a TOLVYN account, and requiring a login to fetch the key would collapse the property the package exists to provide.

### What about `hmac_signature`?

It is present in `ledger.json` and you **cannot and need not** verify it. It is keyed on a TOLVYN-held secret, so verifying it would require that secret — and there is one signing secret per deployment, so anyone holding it could mint a valid signature for any record of any tenant. That is exactly what an auditor must not be able to do. Integrity needs no secret, and authenticity is the Ed25519 manifest signature.

---

## What the signature proves — and what it does not

This section is reproduced inside every package, in the same words. It is here because it should be read before a package is relied on, not after.

Verification proves two things and only two: that these records form an unbroken hash chain, and that TOLVYN signed the manifest describing them.

**The signature attests that TOLVYN observed and charged this. It does not attest that the attribution fields were independently verified.**

Specifically, `agent_id` and `hierarchy_path` (which carries team and service) can be set by the calling client per request, using the `X-Tolvyn-Agent`, `X-Tolvyn-Team` and `X-Tolvyn-Service` headers. Where a header is sent it overrides the value bound to the API key. TOLVYN records what it resolved, meters against it, and signs that — so these fields are the customer's own labelling of the customer's own traffic.

That makes them reliable for what they are for: **allocating cost between teams, services and agents inside one organisation**, with an integrity guarantee that the labels were not altered after the fact. It does **not** make them proof of origin against a party who might benefit from mislabelling, because the party sending the header and the party being attributed are the same organisation.

What **is** independently anchored:

- `tenant_id` comes from the authenticated API key, never from a header.
- Cost, token counts, provider and model come from TOLVYN's own metering of the provider exchange.
- `sequence_number`, `previous_hash` and the chain structure are TOLVYN's, not the client's.

### Contiguity is proven within the package, not against the chain's origin

A package covering sequences 100–150 proves those 51 records are contiguous and correctly anchored to record 99. It does **not** prove that records 1–99 still exist, or that nothing was removed from before the range.

This is the same limitation the ledger page records under [what it does NOT prove](ledger.md#what-ledger-integrity-proves--and-what-it-does-not-prove): a record removed from the *start* of a chain leaves no gap between two survivors, only a chain that begins later than it once did. Retention removes oldest-first, so this is the expected state of any chain past its [retention window](ledger.md#retention-and-the-verified-range), not a sign of tampering.

If completeness from origin matters for your audit, request a package whose period starts at the beginning of the chain and check that `seq_from` is `1`.

### And the limits the ledger itself has

An evidence package inherits every limitation of the underlying ledger. It does not prove the content of prompts or responses (TOLVYN does not store them), that the provider's invoice matches (use [Reconciliation](reconciliation.md)), or that requests which bypassed the proxy were captured — they never reached the ledger at all.

---

## Record format versions

Every record declares its own format, and a verifier applies **that record's** rules to **that record**.

- **Version 1** — the original fourteen hashed fields. `record_format_version` is not part of the hashed payload.
- **Version 2** (from 2026-08-31) — sixteen hashed fields: `record_format_version` leads the payload and is itself hashed, and `agent_id` is appended at the end. `agent_id` is a JSON string, or the bare literal `null` when no agent applies — never `""`, never omitted.

A record with **no** `record_format_version` field is version 1: packages issued before 2026-08-30 predate the field.

**A package may legitimately contain both versions.** A period spanning the change contains records written under each, and they link normally — `previous_hash` is an opaque hash, so a version 2 record follows a version 1 record with no special handling. Version 1 records are never re-hashed or re-issued, so a hash verified today stays verifiable.

**Read the version per record, not per package.** The manifest's `format_version` describes the shape of the ZIP; it says nothing about the records inside.

**An unrecognised version must be reported as unverifiable, and counts as a failure.** Do not fall back to version 1. A fallback either produces a mismatch you then explain away, or — worse — agrees by coincidence and attests a record that was never actually checked. Unverifiable is not the same as valid.

> If you are hand-building a version 2 payload and your digest matches the *version 1* known answer, you have left `record_format_version` out of the payload. The full sixteen-field list is enumerated in the package's `README.txt`; build from that list rather than by adding two entries to the version 1 list from memory.

---

## API reference

### `GET /v1/evidence-package`

Requires a JWT. Requires the **Scale** or **Enterprise** plan. Returns `application/zip`.

| Parameter | Required | Format | Notes |
|---|---|---|---|
| `from` | **Yes** | RFC3339 or `YYYY-MM-DD` | Start of the period. Inclusive. |
| `to` | **Yes** | RFC3339 or `YYYY-MM-DD` | End of the period. Inclusive of the instant — and a date-only value is that day's **midnight**. See [the date range](#before-you-call-it-the-date-range). |

```bash
curl -H "Authorization: Bearer $TOLVYN_TOKEN" \
  "https://api.tolvyn.io/v1/evidence-package?from=2026-08-01&to=2026-09-01" \
  -o evidence.zip
```

The response filename is `tolvyn-evidence-<tenant_id>-<from>-<to>.zip`, with dates as `YYYYMMDD`.

**Errors**

| Status | Code | Cause |
|---|---|---|
| `400` | `invalid_from` / `invalid_to` | Missing, or not RFC3339 / `YYYY-MM-DD`. |
| `400` | `invalid_range` | `from` is not strictly before `to`. |
| `400` | `range_too_large` | The period exceeds 366 days. |
| `402` | `feature_locked` | The tenant's plan does not include evidence packages. |
| `503` | `not_configured` | Signing is not configured on this deployment. TOLVYN returns an error rather than an unsigned package — a package is either signed or it is not produced. |

Parameter validation runs **before** the plan check, so a malformed range returns `400` rather than `402` even on a plan without the feature.

### `GET /v1/evidence-package/pubkey`

**No authentication.** Returns TOLVYN's Ed25519 public key, so an auditor can verify a manifest signature without trusting the copy inside the package.

```bash
curl https://api.tolvyn.io/v1/evidence-package/pubkey
```

---

## There is no CLI command for this

The TOLVYN CLI has no `evidence` command, and no flag on an existing command produces a package.

`tolvyn ledger verify` asks the API to verify a chain and prints the result; `tolvyn ledger export` writes the records as CSV. Neither produces, downloads, or checks an evidence package, and neither is a substitute for one — the CSV carries no signature and no verifier.

To obtain a package, use the **Ledger Verify** page in the dashboard or call the endpoint directly. To verify one, use the `verify_evidence.py` inside it.

---

## Plan availability

| Plan | Evidence packages |
|---|---|
| Free | — |
| Starter | — |
| Growth | — |
| Scale | Yes |
| Enterprise | Yes |

On Free, Starter and Growth the endpoint returns `402 feature_locked`. The [ledger CSV export](ledger.md#exporting-the-ledger) is the export to use there — it carries the records and their hashes, so an auditor can still re-derive the chain, but there is no signed manifest and no bundled verifier.

---

## See also

- [Immutable Ledger](ledger.md) — how records are chained, and what chain integrity proves
- [Reconciliation](reconciliation.md) — comparing TOLVYN's figures against a provider invoice
- [REST API](../reference/api.md) — the full endpoint reference
