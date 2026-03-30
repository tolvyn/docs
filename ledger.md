# Immutable Ledger

Every AI request processed through TOLVYN is written to a hash-chained, HMAC-signed ledger. This provides a tamper-evident record of all AI spend.

© 2026 TOLVYN

---

## What the Ledger Does

For every proxied request, TOLVYN appends a ledger record that contains:

- Request ID and timestamp
- Provider, model, and model family
- Token counts (input, output, cached)
- Cost in microdollars
- Budget status and enforcement action
- The SHA-256 hash of the previous record (chain link)
- An HMAC-SHA256 signature over the record

Because each record includes the hash of its predecessor, any modification to a historical record breaks every subsequent record in the chain. This makes retroactive alteration detectable.

---

## Ledger Record Fields

| Field | Description |
|---|---|
| `sequence_number` | Monotonically increasing integer per tenant |
| `request_id` | UUID of the corresponding request |
| `cost_microdollars` | Cost in µ$ (divide by 1,000,000 for USD) |
| `provider` | `openai`, `anthropic`, or `google` |
| `model_id` | Exact model identifier returned by the provider |
| `model_family` | Normalized family (e.g. `gpt-4o`, `claude-3`) |
| `tokens_input` | Input token count |
| `tokens_output` | Output token count |
| `budget_status` | `ok` or `error` |
| `enforcement_action` | `allow`, `blocked`, or `provider_error` |
| `prev_hash` | SHA-256 of the previous record's canonical fields |
| `record_hash` | SHA-256 of this record's canonical fields |
| `hmac` | HMAC-SHA256 signature (server-side secret) |
| `created_at` | UTC timestamp |

---

## Verifying the Chain

### Via the Dashboard

Go to **Ledger Verify** in the left navigation. Click **Verify Chain**. TOLVYN re-hashes all records in sequence and confirms each `prev_hash` matches. The result shows:

- Total records verified
- Any broken links (none expected in normal operation)
- Verification timestamp

### Via the API

```bash
curl -s \
  -H "Authorization: Bearer <JWT>" \
  "https://api.tolvyn.io/v1/ledger/verify"
```

**Response:**

```json
{
  "valid": true,
  "records_checked": 1482,
  "broken_at_sequence": null,
  "verified_at": "2026-03-30T08:00:00Z"
}
```

If the chain is broken, `valid` is `false` and `broken_at_sequence` contains the sequence number of the first invalid record.

### Fetching Raw Records

```bash
curl -s \
  -H "Authorization: Bearer <JWT>" \
  "https://api.tolvyn.io/v1/ledger?limit=10"
```

Returns the 10 most recent ledger records for your organization.

---

## Why This Matters

The ledger answers questions like:

- "Did we really spend $12,000 in March, or was the invoice wrong?"
- "Can I prove to my CFO that this spend figure hasn't been altered?"
- "Which team's requests drove the cost spike on March 15th?"

Every record is written in the same atomic database transaction as the request row and budget update. If the metering transaction fails, no record is written — the ledger never contains orphaned or partial entries.
