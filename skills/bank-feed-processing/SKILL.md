---
name: bank-feed-processing
description: Process bank and credit card feed transactions by draining CPA-approved reviews first, resolving entities, checking open documents, applying bounded QuickBooks history, flagging uncertain items, recording confident items, and marking every item. Activate when the user mentions bank feed, bank transactions, categorize transactions, unrecorded transactions, CPA review flags, or previewing feed transactions.
---

## Routing

| Situation | Phase |
|-----------|-------|
| Process or auto-categorize bank feed | Phase 1 |
| 3+ confident transactions share type + source account | Phase 2 |
| Preview before posting | Phase 3 |
| Flag one specific transaction | Phase 4 |
| Show processing results | Phase 5 |

## Phase 1 — Process Feed (7 steps, in order)

### Step 0: Drain CPA-approved queue first

`fetchWorkQueue(source="approvedReviews")`

Process ALL approved items before any raw bank-feed inference. For each approved item: use `effectiveCategory` verbatim — never override with agent inference. Skip items with `alreadyFlagged=true` (already in queue; re-flagging creates duplicates). After each approved item, go to Step 6.

### Step 1: Fetch feed + resolve entity

`bankFeed(action="fetch")` — skip items where `alreadyFlagged=true`; include in summary as "skipped".

`qbMasterData(detailedInfo="vendor"|"customer", filter=counterpartyName)`

- 0 matches → new entity; first-ever transaction for a new entity MUST go to Step 4. Never infer an account from agent memory for new entities — QB history is the only valid source.
- 1 match → store `vendorId`/`customerId`, proceed to Step 2
- >1 matches → MULTI-VENDOR AMBIGUITY GUARD blocks; go to Step 4 with full candidate list in `aiReasoning`

### Step 2: Open-document check (separate call, no date window)

Vendor: `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`
Customer: `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

This must be a separate call with no date filter — outstanding status is not date-bounded, and a date-scoped call silently misses open documents from months ago.

- 0 matches → EXPENSE TYPE GUARD pre-condition satisfied; proceed to Step 3
- 1 match + clear intent match → open doc encodes the correct account; skip Step 3, go to Step 5
- >1 matches → one clearly matches by amount/intent → Step 5; ambiguous → Step 4 with all candidates + bank memo

### Step 3: Infer account from bounded history

`qbFetchTransactions(entityId, lookbackDays=365)`

Use dominant account only if **all 5 criteria pass**:
1. ≥ 3 prior transactions in last 365 days
2. Dominant account ≥ 70% of those transactions
3. No second account ≥ 20% of those transactions
4. Current amount within 5× median of dominant-account transactions
5. Most recent dominant-account transaction < 180 days old

Any failure → Step 4. Common failure patterns: multi-purpose vendors (Amazon, Costco, Home Depot) with 2+ accounts each ≥20%; fewer than 3 txns; amount >5× median; no dominant txn in 180 days.

(CONSISTENCY RULE GUARD blocks history-inferred writes unless all 5 criteria were explicitly evaluated and passed. MATERIALITY GUARD blocks unusually large transactions.)

### Step 4: Flag uncertain transaction

`flagForReview(tellerTransactionId, aiReasoning, suggestedCategory)`

`aiReasoning` **must include** the bank-feed memo verbatim exactly as shown + entity name + history summary + amount + date + reason for review.

`suggestedCategory` = hint only; never authority to auto-record.

Triggers that always land here: new entity (Step 1), multiple entity matches (Step 1), multiple open docs without clear match (Step 2), any failed consistency criterion (Step 3), amount >5× median, currency mismatch, ambiguous bank description.

Then → Step 6.

### Step 5: Record confident transaction

**Duplicate check first:**
`qbFetchTransactions(entityId, startDate=txnDateMinus3Days, endDate=txnDatePlus3Days, amount=amount)`

True duplicate = same vendor/customer + date within 3 days + amount within 10% + matching memo OR `bankTransactionId`. Identical recurring charges on different dates are NOT duplicates.

**Set idempotency key:**
`idempotencyKey = hash(bankTransactionId + amount + txnDate + vendorId)` — use `customerId` if no `vendorId`.

**Write tool selection:**
| Condition | Tool |
|-----------|------|
| Outstanding Bill matched | `qbBillPayment(vendorId, billId, txnDate, accountId=sourceAccountId, amount, memo=bankMemo, idempotencyKey)` |
| Outstanding Invoice matched | `qbReceivePayment(customerId, invoiceId, txnDate, accountId=sourceAccountId, amount, memo=bankMemo, idempotencyKey)` |
| Vendor expense, no bill | `qbExpense(vendorId, txnDate, accountId=sourceAccountId, lines, memo=bankMemo, idempotencyKey)` |
| Customer immediate sale | `qbSalesReceipt(customerId, txnDate, accountId=sourceAccountId, lines, memo=bankMemo, idempotencyKey)` |
| Customer, invoice and collect later | `qbInvoice(customerId, txnDate, lines, memo=bankMemo, idempotencyKey)` |
| Non-customer deposit | `qbDeposit(txnDate, accountId=sourceAccountId, lines, memo=bankMemo, idempotencyKey)` |

Required on every write: entity ID (except true non-customer deposit), `txnDate`, source `accountId`, `lines` with inferred/matched account, `memo=bankMemo` verbatim and never blank, `idempotencyKey`.

Delegate full write logic to `accounts-payable` or `accounts-receivable` skill before calling the write tool.

Active guards on every write: WRITE SAFETY GUARD, DUPLICATE RESULT GUARD, EXPENSE TYPE GUARD, DEPOSIT TYPE GUARD, INCOME TYPE GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, SOURCE-CATEGORY COLLISION GUARD, CURRENCY GUARD.

True duplicate found → skip write, go to Step 6 with existing transaction ID.

### Step 6: Mark recorded

CPA-approved item: `fetchWorkQueue(source="markRecorded", reviewItemNumber, qbTransactionId)`
Raw bank-feed item: `fetchWorkQueue(source="markRecorded", tellerTransactionId, qbTransactionId)`

If mark-recorded fails → include in summary as "recorded/flagged but not marked"; do NOT re-record — idempotency protects retries.

## Phase 2 — Batch Confident Writes

Use only when 3+ transactions share the same type AND source account after per-item Steps 0–4 have run for each.

`qbFetchTransactions(transactionType, startDate=batchStart, endDate=batchEnd, accountId=sourceAccountId)` — remove any duplicate-risk items.

`qbBatch(operations=batchOperations)` — BATCH SAFETY GUARD blocks if: master data not verified, types are mixed, or duplicate coverage doesn't span all batch items.

Per-item responses: mark successes (Step 6), retry failures individually with same `idempotencyKey` (idempotency makes retries safe), flag unresolved failures (Step 4).

## Phase 3 — Preview Before Recording

`bankFeed(action="fetch")` → present table: date, description, amount, suggested category, consistency status.

Do not record anything until explicitly confirmed.
> Review these categorizations. Reply "record" to post confident items, "flag all uncertain" for CPA review, or tell me which rows to change.

## Phase 4 — Flag One Transaction

`bankFeed(action="fetch")` → match by user-provided description/amount/date.
`flagForReview(tellerTransactionId, aiReasoning)` — `aiReasoning` must include user's reason + bank memo verbatim.
`fetchWorkQueue(source="markRecorded", tellerTransactionId, reviewItemNumber)`

## Phase 5 — Summary

```
Bank Feed Summary — [Date]
═══════════════════════════
Processed: X transactions
Recorded:  Y ($total)
Flagged:   Z for CPA review
Skipped:   W (alreadyFlagged / duplicates)
```

Include flagged items table with `aiReasoning` summary and bank memo.

## Gotchas

1. **CPA queue drain (Step 0) before inference** — approved categories always take precedence over agent inference; drain first or approved work gets overwritten
2. **`alreadyFlagged=true` items: skip entirely** — include in summary as "skipped"; never re-flag; that creates duplicates in the review queue
3. **Open-doc check (Step 2) must be a separate call with no date window** — one `qbFetchTransactions` call cannot serve both open-doc and history purposes
4. **`flagForReview` `aiReasoning` must include bank memo verbatim** — FLAG QUALITY GUARD blocks generic or short reasoning; the memo is what the CPA uses to identify the transaction
5. **`idempotencyKey` makes retries safe** — if mark-recorded fails, don't re-record; the write already succeeded and the idempotency key prevents a duplicate on retry
