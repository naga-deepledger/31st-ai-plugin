---
name: bank-feed-processing
description: Process bank feed transactions — implement the v2 transaction recording framework (Steps 0–6): CPA-approved queue first, entity resolution with multi-vendor disambiguation, separate open-document check, deterministic consistency rule for account inference, flagging with bank memo context, idempotent writes, and mark-recorded. Use when the user mentions bank feed, bank transactions, categorize transactions, or auto-categorize.
---

# Bank Feed Processing Skill

Process unrecorded bank and credit card transactions from the connected bank feed using the v2 transaction recording framework. Each transaction follows a deterministic 7-step path: CPA queue → entity resolve → open-document check → account inference → flag or record → mark recorded.

## Trigger

Activate when the user wants to:
- Process bank feed transactions
- Categorize bank transactions
- Review unrecorded transactions
- Flag transactions for CPA review

## Design Invariants

- QuickBooks is the only memory. No vendor→account agent-side cache, ever.
- Fully async — no mid-batch user prompts. Anything uncertain → `flagForReview`.
- First transaction for any new entity must be human-cleared (sets the historical precedent the system will rely on).
- Skills express intent; hooks enforce invariants (EXPENSE TYPE GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, FLAG QUALITY GUARD, CONSISTENCY RULE GUARD, MATERIALITY GUARD, CURRENCY GUARD, TAX GUARD).
- Idempotent writes: always pass `idempotencyKey = hash(bankTransactionId + amount + txnDate + vendorId)` on every write call.

## Workflow: Process Bank Feed (v2 Framework)

### Step 0 — CPA Queue First (highest precedence)

**Fetch approved reviews** — `fetchWorkQueue(source="approvedReviews")` → returns all CPA-approved items awaiting recording.

For each approved item:
- Record using the `effectiveCategory` verbatim — do NOT override with agent inference, regardless of QB history.
- Call `fetchWorkQueue(source="markRecorded", reviewItemNumber=N, qbTransactionId=ID)` immediately after recording.
- Skip any item with `alreadyFlagged=true` — it is already in the queue; do not re-flag.

CPA-approved items are processed before any bank-feed inference begins. Once the approved queue is drained, proceed to the raw bank feed.

### Step 1 — Fetch Bank Feed + Resolve Entity

**Fetch transactions** — `bankFeed(action="fetch")` → raw unrecorded bank transactions.

For each transaction, check `alreadyFlagged=true` — if set, skip entirely (already queued for CPA).

**Resolve entity** — `qbMasterData(detailedInfo="vendor"|"customer", filter=counterpartyName)`.

Interpret the result count:
- **0 matches** — treat as a new entity. A first-ever transaction for any new entity must go to `flagForReview` (Step 4). Do not attempt to infer an account.
- **1 match** — use that `vendorId` / `customerId`. Continue to Step 2.
- **>1 matches** — call `flagForReview` with the full candidate list in `aiReasoning`; do NOT pick one. Picking the wrong vendor corrupts AP/AR reports and the audit trail. Continue to next transaction.

### Step 2 — Open-Document Check (separate call, no date window)

This step runs as a **dedicated call** with `outstandingOnly=true` — it is NOT combined with the history fetch in Step 3. Outstanding status is independent of date; a date-scoped call will silently miss open documents from 6+ months ago.

**For vendor flows:**
`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

**For customer flows:**
`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

Evaluate the result:
- **Matching open document found** (same vendor + amount within tolerance, or clear intent match) → use `qbBillPayment` (vendor) or `qbReceivePayment` (customer). The open document already encodes the correct account — do NOT re-infer. Skip to Step 5.
- **No match** → continue to Step 3. The EXPENSE TYPE GUARD hook will verify this call was made before any `qbExpense` write.

### Step 3 — Account Inference (bounded history)

**Fetch bounded history** — `qbFetchTransactions(entityId=..., entityType=..., lookbackDays=365)`.

Apply the **CONSISTENCY RULE** — use the dominant account ONLY IF ALL five criteria hold:

| Criterion | Threshold |
|-----------|-----------|
| (a) Prior transactions in last 365 days | ≥ 3 |
| (b) Dominant account share | ≥ 70% of those transactions |
| (c) No second account | < 20% of those transactions |
| (d) Current amount vs dominant-account median | within 5× median |
| (e) Age of most recent dominant-account transaction | < 180 days old |

If **all five pass** → proceed to Step 5 with the dominant account.

If **any criterion fails** → go to Step 4 (flag). Do not pick the most frequent account as a fallback; the CONSISTENCY RULE GUARD hook will block inconsistent writes.

Common failure patterns that require flagging:
- Multi-purpose vendor (Amazon, Costco, Home Depot) with 2+ accounts each ≥ 20% — split history detected
- Fewer than 3 prior transactions — insufficient precedent
- Amount > 5× median — materiality anomaly (also caught by MATERIALITY GUARD hook)
- No dominant-account transaction in last 180 days — dormant relationship

### Step 4 — Flag for Review

**Call** — `flagForReview(aiReasoning=..., suggestedCategory=optional)`.

The `aiReasoning` field must include all of the following (enforced by FLAG QUALITY GUARD hook):
- Vendor/customer name + history summary (e.g., "Past splits: Office Supplies 60%, Computer Equipment 40% — no dominant account")
- Bank-feed memo verbatim (e.g., `"AWS HOSTING SJC1 AMZN.COM"`) — never omit; this is primary evidence for the CPA
- Suggested account if a static keyword→account hint applies (SUGGEST ONLY — never auto-records)
- Transaction amount and date

Triggers that always lead to Step 4:
- New entity (0 `qbMasterData` matches)
- Multi-vendor disambiguation (>1 `qbMasterData` match)
- Split history (no dominant account meeting the consistency rule)
- Materiality anomaly (amount > 5× median)
- Currency mismatch or unhandled tax data (also blocked by CURRENCY GUARD / TAX GUARD hooks)
- Any consistency rule criterion fails

After `flagForReview`, call `fetchWorkQueue(source="markRecorded", ...)` — Step 6 — to prevent re-processing.

### Step 5 — Record

Select the correct tool based on the transaction type:

| Situation | Tool |
|-----------|------|
| Matched outstanding Bill | `qbBillPayment` |
| Matched outstanding Invoice | `qbReceivePayment` |
| Vendor expense, paid immediately | `qbExpense` |
| Vendor bill, pay later | `qbBill` |
| Customer sale, paid immediately | `qbSalesReceipt` |
| Customer sale, pay later | `qbInvoice` |
| Non-customer deposit | `qbDeposit` |

Required fields on every write:
- `vendorId` / `customerId` (from Step 1)
- `txnDate`, source `accountId`, `lines` with the inferred or matched account
- `memo` — include the bank-feed memo verbatim (never blank this field)
- `idempotencyKey` — `hash(bankTransactionId + amount + txnDate + vendorId)` — prevents duplicate writes on retry after partial failure

### Step 6 — Mark Recorded

**Call** — `fetchWorkQueue(source="markRecorded", reviewItemNumber=N, qbTransactionId=ID)`.

This step is mandatory on EVERY path — CPA-approved items (Step 0), successfully recorded transactions (Step 5), and flagged items (Step 4). Without it, the transaction reappears in the next bank-feed fetch.

## Workflow: Batch Processing

When 3+ transactions share the same type and source account:

1. **Group** by transaction type (Expense, BillPayment, Deposit, etc.)
2. **Run Steps 0–4 per transaction** — entity resolve, open-document check, and consistency rule are per-item operations; they cannot be batched because each transaction may have a different vendor and different history
3. **Submit confident records** via `qbBatch` — only transactions that passed the consistency rule and are ready to write
4. **Check response** for per-item success/failure; retry failed items individually with the same `idempotencyKey` (safe to retry — idempotent)
5. **Flag remainder** individually via `flagForReview`

## Workflow: Review Mode

When the user wants to preview transactions before recording:

1. `bankFeed(action="fetch")` — pull all unprocessed
2. Present in a table: date, description, amount, suggested category, consistency status
3. Do NOT record anything until explicitly confirmed

## Workflow: Flag a Specific Transaction

When the user wants to flag a particular transaction manually:

1. Identify the transaction by description, amount, or date
2. `flagForReview(tellerTransactionId=..., aiReasoning=...)` with the user's stated reason plus the bank memo verbatim
3. `fetchWorkQueue(source="markRecorded", ...)` — prevent re-processing
4. Confirm: "Flagged for CPA review with reason: [reason]"

## Workflow: Report Summary

After processing, present:

```
Bank Feed Summary — [Date]
═══════════════════════════
Processed: X transactions
Recorded:  Y ($total)
Flagged:   Z for CPA review
Skipped:   W (alreadyFlagged / duplicates)
```

Include a table of flagged items with the `aiReasoning` summary and bank memo.

## Safety Checklist

- [ ] CPA-approved queue (`fetchWorkQueue approvedReviews`) processed first — `effectiveCategory` used verbatim, never overridden
- [ ] `alreadyFlagged=true` transactions skipped — not re-flagged
- [ ] Entity resolved via `qbMasterData` — 0 matches → flag, 1 match → continue, >1 matches → flag with candidate list
- [ ] Open-document check run as a **separate call** with `outstandingOnly=true` and **no date window** — before any account inference
- [ ] Consistency rule applied: all five criteria checked before using history-inferred account
- [ ] Bank-feed memo included verbatim in `aiReasoning` on every flag
- [ ] Idempotency key (`hash(bankTransactionId + amount + txnDate + vendorId)`) on every write
- [ ] `fetchWorkQueue(source="markRecorded")` called on every path — record, flag, and CPA-approved items

## Common Mistakes to Avoid

- Re-flagging a transaction with `alreadyFlagged=true` — check this field before any `flagForReview` call
- Combining the open-document check with the history fetch — they use different scopes; merge them and you silently miss old open bills
- Picking one vendor when `qbMasterData` returns multiple matches — always flag with the candidate list
- Using history-inferred account when fewer than 3 prior transactions exist — insufficient precedent; always flag
- Using the most-frequent account on a split-history vendor — flag; do not guess
- Omitting the bank-feed memo from `aiReasoning` — CPAs need the raw memo to decide faster
- Omitting `idempotencyKey` from writes — duplicates on retry
- Not calling `fetchWorkQueue(source="markRecorded")` after flagging — transaction reappears next run
- Overriding a CPA-approved `effectiveCategory` with agent inference
