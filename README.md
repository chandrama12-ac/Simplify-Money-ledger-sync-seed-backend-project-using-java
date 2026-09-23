# Ledger Sync — Simplify Money Backend Engineering Assignment

Java backend that reads a user's raw bank SMS/email, turns them into a
deduplicated, categorized transaction ledger, and produces three reconciled
output files. Built on top of the Simplify Money `ledger-sync-seed`
scaffolding.

> Built for the assignment's own README first — this file documents *this
> fork's* solution: what's implemented, how it's structured, and how to run it.
> Numbers marked `<update>` below should be replaced with your actual latest
> `./verify.sh` output before submission — don't leave placeholders in.

---

## 1. What this service does

1. **Ingest** raw SMS/email messages (`fixtures/corpus-a.jsonl`).
2. **Parse** each message per its sender/format into a normalized fragment.
3. **Deduplicate** — merge fragments that evidence the same underlying
   transaction (multi-channel confirmations, re-synced duplicates) into one
   record.
4. **Categorize** every transaction as exactly one of `SPEND`, `INCOME`,
   `MICRO`, or `TRANSFER`.
5. **Report** three files: `ledger.json`, `summary.json`,
   `reconciliation.json`.
6. **Migrate** the ledger's storage from SQL to a document store, with a
   re-runnable `Backfill` and a field-level `ConsistencyChecker`.

---

## 2. Architecture

### 2.1 Package layout

```
src/main/java/in/simplifymoney/ledgersync/
  model/       RawMessage, NormalizedTxn (frozen), Category (frozen), Direction
  json/        minimal JSON reader/writer (JDK-only, no external dep)
  parse/       one parser per message format (HDFC SMS, ICICI SMS, Email, ...)
  ingest/      reads a corpus file, drives parsing + dedup, saves results
  store/       SqlStore, DocumentStore, Backfill, ConsistencyChecker
  report/      writes ledger.json, summary.json, reconciliation.json
  App.java     entry point — migrate | ingest | report
  SelfCheck.java
```

`model/NormalizedTxn.java`, `model/Category.java`, and
`NormalizedTxnContractTest.java` are frozen — untouched in this fork.

### 2.2 Data flow

```
corpus-a.jsonl
      │
      ▼
 ┌─────────┐   per-sender/format match   ┌──────────────┐
 │  parse/ │ ───────────────────────────▶│ ParsedTxn     │
 └─────────┘                             │ (fragment)    │
                                          └──────┬────────┘
                                                 ▼
                                   ┌───────────────────────────┐
                                   │ ingest/ — TransactionKey   │
                                   │ lookup + merge             │
                                   │ (dedup happens here)       │
                                   └──────────┬──────────────┘
                                              ▼
                                   ┌───────────────────────┐
                                   │ Category assignment    │
                                   │ SPEND/INCOME/MICRO/    │
                                   │ TRANSFER                │
                                   └──────────┬────────────┘
                                              ▼
                          ┌───────────────────────────────────┐
                          │ store/ — SqlStore (source of truth) │
                          └──────────────┬──────────────────┘
                                         ▼
                          ┌───────────────────────────────────┐
                          │ report/ — ledger.json / summary.json│
                          │           reconciliation.json       │
                          └───────────────────────────────────┘

                          ┌───────────────────────────────────┐
   SqlStore ───Backfill──▶│ store/DocumentStore                │
                          └──────────────┬──────────────────┘
                                         ▼
                          ┌───────────────────────────────────┐
                          │ ConsistencyChecker — field-level    │
                          │ diff, SqlStore vs DocumentStore     │
                          └───────────────────────────────────┘
```

### 2.3 Transaction identity (why dedup works)

`message_id` is rejected as an identity key — it's a phone-upload artifact,
not the identity of the underlying transaction. Instead:

```
TransactionKey = accountLast4 + occurredAt(normalized to UTC instant) + direction + amount
```

Ingestion keeps an identity map keyed by this tuple. A new message either
opens a new entry or merges into an existing one (adding its `message_id` to
the evidence set). Re-ingesting the same or an overlapping corpus is a no-op
by construction — this is what makes the pipeline idempotent under replay,
per the assignment's re-run requirement.

Rejected alternatives: `message_id` (upload artifact, not transaction
identity), merchant string (varies by channel for the same event), synthetic
auto-increment id (breaks idempotency on replay).

### 2.4 Category rules

| Category | Rule | In `spend`/`income`? |
|---|---|---|
| `SPEND` | Money left the user and is gone | Yes (`spend`) |
| `INCOME` | Money arrived and is theirs | Yes (`income`) |
| `MICRO` | UPI debit ≤ ₹100 | No — rolled up to `micro_count`/`micro_total` only |
| `TRANSFER` | One leg of the user's own money moving between their own accounts | No — excluded from both |

A self-transfer is identified as a matched debit/credit pair, equal amount,
close in time, across the user's own tracked accounts — not by
sender/counterparty label alone (an external inbound credit labeled "SELF" is
`INCOME`, not `TRANSFER`, since it has no matching outbound leg in this
ledger).

### 2.5 Document store

Three declared access patterns drive the document shape (not a ported
relational schema):

1. one account's transactions for one month, newest first
2. running per-category totals for an account
3. given a message id, which transaction did it produce

Store choice: `<DynamoDB / MongoDB — state which, and why, once decided>`.

Examined-vs-returned at 100,000 transactions (`ScannedCount`/`Count` or
`totalDocsExamined`/`nReturned`):

| Query | Examined | Returned |
|---|---|---|
| Monthly transactions by account | `<update>` | `<update>` |
| Running category totals | `<update>` | `<update>` |
| Transaction by message id | `<update>` | `<update>` |

---

## 3. Workflow — how to run it

```bash
# Compile + run the pipeline end to end, no network needed
./verify.sh

# Full test suite (needs network once, for JUnit)
./gradlew test

# Individual steps
./gradlew run --args="migrate"
./gradlew run --args="ingest fixtures/corpus-a.jsonl"
./gradlew run --args="report submission/"
```

`submission/` will contain `ledger.json`, `summary.json`, and
`reconciliation.json` after the `report` step.

### 3.1 Checking against the checkpoint

```bash
diff <(jq -S . submission/summary.json) <(jq -S . fixtures/corpus-a-totals.json)
```

`fixtures/corpus-a-totals.json` gives expected transaction count, opening/
closing balance, and category totals per account — no row-level answers. If
the numbers don't match, that goes in the write-up with a real explanation,
not a forced fix.

### 3.2 Document store + migration

```bash
docker compose up --build      # brings up the service + document store
./gradlew run --args="backfill"
./gradlew run --args="consistency-check"
```

---

## 4. Current status

_(fill in from your actual latest run before submitting — do not leave
placeholder text in the final version)_

- `./verify.sh` currently reports: `<update — transaction count vs. 257 expected>`
- Parsers implemented: `<update — which formats are handled>`
- Deduplication: `<update — implemented / not yet>`
- Categorization: `<update — SPEND/INCOME only, or all four?>`
- `reconciliation.json`: `<update — writing yet?>`
- Document store / Backfill / ConsistencyChecker: `<update>`
- Incident `INC-2026-09-11`: `<update — reproduced / fixed / test added?>`

---

## 5. Known gaps

List anything genuinely unfinished here, plainly — per the assignment's own
instruction, "ran out of time" is fine, pretending it works is not.

- `<update>`
