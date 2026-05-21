---
name: bank-feed-processing
description: Process bank feed and credit card feed transactions by draining CPA-approved reviews first, resolving entities, checking open documents, applying bounded QuickBooks history, flagging uncertain items, recording confident items, and marking every item recorded. Use when the user mentions bank feed, bank transactions, categorize transactions, auto-categorize, unrecorded transactions, CPA review flags, or previewing feed transactions before posting.
---

| Situation | Phase |
|---|---|
| Process or auto-categorize bank feed | Phase 1 — Process Feed |
| 3+ confident transactions share type and source account | Phase 2 — Batch Confident Writes |
| User wants preview before posting | Phase 3 — Review Before Recording |
| User asks to flag one transaction | Phase 4 — Flag One Transaction |
| User asks for processing results | Phase 5 — Report Summary |

## Phase 1 — Process Feed

### Step 0: Drain CPA-approved queue first
`fetchWorkQueue(source="approvedReviews")`

Branching:
  0 matches → Proceed to Step 1.
  1 match   → Record the approved item using `effectiveCategory` verbatim; do not override with agent inference regardless of QuickBooks history. Store `reviewItemNumber`, `effectiveCategory`, entity, amount, txnDate, source account, bank memo, and any approved category/account IDs; then proceed to Step 5.
  >1 matches → Process each approved item before raw bank-feed inference begins. Skip any approved item with `alreadyFlagged=true`; it is already in the queue and must not be re-flagged.

Mode handling:
- Batch/async mode: never ask mid-batch; drain all approved reviews first.
- Interactive mode: before any write in Step 5, emit the required pre-write evidence.
- Month-end mode: propose the approved entries, but do not post without approval.

### Step 1: Fetch feed and resolve entity
`bankFeed(action="fetch")`

For each transaction, if `alreadyFlagged=true`, skip entirely and include it in the summary as skipped.

`qbMasterData(detailedInfo="vendor", filter=counterpartyName)`

or

`qbMasterData(detailedInfo="customer", filter=counterpartyName)`

Branching:
  0 matches → Treat as a new entity; first-ever transaction for any new entity must be human-cleared and must go to Step 4. Do not infer an account from agent-side memory; QuickBooks is the only memory and no vendor→account cache may be used.
  1 match   → Store `vendorId` or `customerId`, matched entity name, and entity type; continue to Step 2.
  >1 matches → Do not pick one. Go to Step 4 with the full candidate list in `aiReasoning` (MULTI-VENDOR AMBIGUITY GUARD blocks picking one entity).

Interactive prompt for >1 matches:
> I found multiple QuickBooks entities matching “[counterpartyName]”. I can’t safely choose one because the wrong entity corrupts AP/AR reports and the audit trail. Should I flag this transaction for CPA review with the candidate list?

Batch/async handling:
- Never ask mid-batch; call Step 4 with the candidate list.

### Step 2: Check open documents separately
For vendor flows:

`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

For customer flows:

`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

This is a dedicated open-document call with `outstandingOnly=true` and no date window; do not combine it with the bounded history fetch in Step 3 because outstanding status is independent of date and a date-scoped call can miss open documents from 6+ months ago.

Branching:
  0 matches → Continue to Step 3. For vendor expenses, this call is the pre-condition for `qbExpense` (EXPENSE TYPE GUARD blocks `qbExpense` if outstanding Bills were not checked).
  1 match   → If same entity and amount is within tolerance, or the bank memo provides a clear intent match, store the open `billId` or `invoiceId`; skip Step 3 because the open document already encodes the correct account; proceed to Step 5.
  >1 matches → If exactly one document clearly matches by amount or intent, store that document ID and proceed to Step 5; otherwise go to Step 4 with all candidate open documents and the bank memo.

Interactive prompt for multiple open documents:
> I found multiple outstanding documents for “[entityName]” and can’t determine which one this bank transaction pays. Should I flag it for CPA review with the open document list and bank memo?

Batch/async handling:
- Never ask mid-batch; call Step 4 with all candidate open documents.

### Step 3: Infer account from bounded QuickBooks history
`qbFetchTransactions(entityId=entityId, entityType=entityType, lookbackDays=365)`

Apply the CONSISTENCY RULE before using any history-inferred account. Use the dominant account only if all five criteria pass:
1. At least 3 prior transactions in the last 365 days.
2. Dominant account is at least 70% of those transactions.
3. No second account is 20% or more of those transactions.
4. Current amount is within 5× the median of dominant-account transactions.
5. Most recent dominant-account transaction is less than 180 days old.

Branching:
  0 matches → Go to Step 4; there is insufficient QuickBooks precedent.
  1 match   → Go to Step 4 unless the full result set still satisfies all five criteria; one prior transaction alone is insufficient precedent.
  >1 matches → If all five criteria pass, store `categoryAccountId`, dominant-account share, median amount, most recent dominant-account transaction date, and the history summary; proceed to Step 5. If any criterion fails, go to Step 4.

Failure patterns that go to Step 4:
- Multi-purpose vendor such as Amazon, Costco, or Home Depot with 2+ accounts each at least 20%.
- Fewer than 3 prior transactions.
- Amount greater than 5× median.
- No dominant-account transaction in the last 180 days.
- Split history with no dominant account meeting the rule.

(CONSISTENCY RULE GUARD blocks history-inferred writes unless all five criteria were explicitly evaluated and passed. MATERIALITY GUARD also blocks unusually large transactions relative to vendor/customer history.)

### Step 4: Flag uncertain transaction for review
`flagForReview(tellerTransactionId=tellerTransactionId, aiReasoning=aiReasoning, suggestedCategory=suggestedCategory)`

`aiReasoning` must be specific and must include the bank-feed memo verbatim exactly as shown in the bank feed, plus vendor/customer name, history summary, transaction amount, transaction date, and the reason review is needed; include a suggested account only when a static keyword→account hint applies, and treat it as a suggestion only, never as authority to auto-record.

Branching:
  0 matches → Not applicable; this is a write to the review queue.
  1 match   → Store `reviewItemNumber` or review ID; proceed to Step 6.
  >1 matches → Not applicable.

Triggers that always lead here:
- New entity from Step 1.
- Multiple entity matches from Step 1.
- Multiple open documents without one clear match from Step 2.
- Split history or any failed CONSISTENCY RULE criterion from Step 3.
- Materiality anomaly where amount is greater than 5× median.
- Currency mismatch without FX handling.
- Unhandled tax data.
- Ambiguous bank description or unclear business purpose.

Interactive prompt before flagging a single transaction:
> I don’t have enough reliable QuickBooks evidence to record this transaction automatically. I’ll flag it for CPA review with the bank memo, amount, date, entity evidence, and history summary unless you want to stop here.

Batch/async handling:
- Never ask mid-batch; flag immediately with specific `aiReasoning`.

(FLAG FOR REVIEW QUALITY GUARD blocks vague or missing `aiReasoning`.)

### Step 5: Record confident transaction
Before any write, run a duplicate check covering the transaction date, amount, entity, memo, and bank transaction ID:

`qbFetchTransactions(entityId=entityId, entityType=entityType, startDate=txnDateMinus3Days, endDate=txnDatePlus3Days, amount=amount)`

A true duplicate requires same vendor/customer, date within 3 days, amount within 10%, and either matching memo text or matching `bankTransactionId`; identical recurring charges on different dates are not duplicates.

Set:

`idempotencyKey=hash(bankTransactionId + amount + txnDate + vendorId)`

For customer transactions, use the same idempotency format with `customerId` in the entity position when no `vendorId` exists:

`idempotencyKey=hash(bankTransactionId + amount + txnDate + customerId)`

Select the write tool:
- Matched outstanding Bill → delegate full write logic to `accounts-payable`, then call `qbBillPayment(vendorId=vendorId, billId=billId, txnDate=txnDate, accountId=sourceAccountId, amount=amount, memo=bankMemo, idempotencyKey=idempotencyKey)`
- Matched outstanding Invoice → delegate full write logic to `accounts-receivable`, then call `qbReceivePayment(customerId=customerId, invoiceId=invoiceId, txnDate=txnDate, accountId=sourceAccountId, amount=amount, memo=bankMemo, idempotencyKey=idempotencyKey)`
- Vendor expense, paid immediately and no prior bill → delegate full write logic to `accounts-payable`, then call `qbExpense(vendorId=vendorId, txnDate=txnDate, accountId=sourceAccountId, lines=[{accountId=categoryAccountId, amount=amount}], memo=bankMemo, idempotencyKey=idempotencyKey)`
- Customer sale, paid immediately → delegate full write logic to `accounts-receivable`, then call `qbSalesReceipt(customerId=customerId, txnDate=txnDate, accountId=sourceAccountId, lines=[{accountId=categoryAccountId, amount=amount}], memo=bankMemo, idempotencyKey=idempotencyKey)`
- Customer sale, pay later → delegate full write logic to `accounts-receivable`, then call `qbInvoice(customerId=customerId, txnDate=txnDate, lines=[{accountId=categoryAccountId, amount=amount}], memo=bankMemo, idempotencyKey=idempotencyKey)`
- Non-customer deposit → delegate full write logic to `accounts-receivable`, then call `qbDeposit(txnDate=txnDate, accountId=sourceAccountId, lines=[{accountId=categoryAccountId, amount=amount}], memo=bankMemo, idempotencyKey=idempotencyKey)`

Required fields on every write:
- `vendorId` or `customerId` from Step 1, except true non-customer deposit.
- `txnDate`.
- Source `accountId`.
- `lines` with inferred or matched account.
- `memo=bankMemo`, with the bank-feed memo verbatim and never blank.
- `idempotencyKey`.

Interactive pre-write output:
Pre-write evidence:
- Entity: [name + ID]
- Open-doc: [no outstanding bills / billId X applied]
- Account basis: [N txns, XX% dominant / open-doc match]
- Mode: interactive

Interactive confirmation prompt:
> Confirm I should record this bank-feed transaction in QuickBooks using the evidence above?

Month-end handling:
> I can propose these bank-feed entries for month-end, but I will not post them without approval. Do you approve posting the proposed entries?

Branching:
  0 matches → If the duplicate check found no true duplicate and the write succeeds, store `qbTransactionId`; proceed to Step 6. If month-end approval is missing, present proposed entries only and stop before the write.
  1 match   → If the duplicate check found a true duplicate, skip the write and proceed to Step 6 using the existing transaction ID only if it is the same bank-feed item; otherwise go to Step 4.
  >1 matches → Do not write; go to Step 4 with duplicate candidates.

(WRITE SAFETY GUARD requires `qbMasterData` and `qbFetchTransactions` before writes. DUPLICATE RESULT GUARD blocks true duplicates. EXPENSE TYPE GUARD blocks `qbExpense` when outstanding Bills exist or bill/payable language indicates `qbBill`/`qbBillPayment`. VENDOR/ACCOUNT RESOLUTION GUARD blocks IDs not returned by `qbMasterData`. SOURCE-CATEGORY COLLISION GUARD blocks source account and line account collisions. CURRENCY GUARD blocks cross-currency writes without `exchangeRate` or explicit FX handling.)

### Step 6: Mark recorded
For CPA-approved items, successful writes, duplicate skips that map to the same bank-feed item, and flagged items:

`fetchWorkQueue(source="markRecorded", reviewItemNumber=reviewItemNumber, qbTransactionId=qbTransactionId)`

For raw bank-feed transactions without a review item number:

`fetchWorkQueue(source="markRecorded", tellerTransactionId=tellerTransactionId, qbTransactionId=qbTransactionId)`

Branching:
  0 matches → If the mark-recorded call fails or returns no confirmation, include the item in the summary as “recorded/flagged but not marked”; do not re-record because Step 5 idempotency protects retries.
  1 match   → Store mark-recorded confirmation; include item in summary.
  >1 matches → Stop and go to Step 4 or support review; multiple mark-recorded targets indicate queue identity ambiguity.

## Phase 2 — Batch Confident Writes

### Step 1: Group eligible confident records
Use this phase only when 3+ transactions share the same transaction type and source account after Phase 1 Steps 0–4 have run per transaction.

Branching:
  0 matches → Return to Phase 1 Step 5 for individual writes.
  1 match   → Return to Phase 1 Step 5 for individual write.
  >1 matches → Group by transaction type and source account; continue to Step 2.

### Step 2: Prepare homogeneous batch
Run Phase 1 Steps 0–4 per transaction before batching; entity resolution, open-document check, duplicate check readiness, and consistency evaluation are per-item operations and cannot be batched because each transaction may have a different vendor and different history.

`qbFetchTransactions(transactionType=transactionType, startDate=batchStartDate, endDate=batchEndDate, accountId=sourceAccountId)`

Branching:
  0 matches → Continue to Step 3 with confident items.
  1 match   → If it is a true duplicate for one item, remove that item from the batch and handle it through Phase 1 Step 6 or Step 4.
  >1 matches → Remove duplicate-risk items; only non-duplicate confident records continue.

### Step 3: Submit batch and handle per-item responses
`qbBatch(operations=batchOperations)`

Branching:
  0 matches → If the batch call fails entirely, retry failed items individually through Phase 1 Step 5 with the same `idempotencyKey`; idempotency makes retries safe.
  1 match   → Check per-item success/failure; for each success, run Phase 1 Step 6; for each failure, retry individually through Phase 1 Step 5 with the same `idempotencyKey`.
  >1 matches → Treat as per-item responses; mark successes, retry failures individually, and flag unresolved failures through Phase 1 Step 4.

(BATCH SAFETY GUARD blocks `qbBatch` if master data was not verified, transaction types are mixed, or duplicate coverage does not span all batch items.)

### Step 4: Flag batch remainder
`flagForReview(tellerTransactionId=tellerTransactionId, aiReasoning=aiReasoning, suggestedCategory=suggestedCategory)`

Branching:
  0 matches → Not applicable.
  1 match   → Run Phase 1 Step 6.
  >1 matches → Not applicable.

Batch/async handling:
- Never ask mid-batch; flag each remainder individually.

## Phase 3 — Review Before Recording

### Step 1: Fetch unprocessed feed
`bankFeed(action="fetch")`

Branching:
  0 matches → Tell the user there are no unprocessed bank-feed transactions.
  1 match   → Build one preview row.
  >1 matches → Build a preview table.

### Step 2: Present preview table
Present columns: date, description, amount, suggested category, consistency status.

Do not record anything until explicitly confirmed.

Interactive prompt:
> Review the proposed categorizations below. Reply “record” to post the confident items, “flag all uncertain” to send uncertain items to CPA review, or tell me which rows to change.

Branching:
  0 matches → No preview rows; stop.
  1 match   → If user confirms recording, route the item through Phase 1 Steps 1–6.
  >1 matches → If user confirms recording, route each item through Phase 1 Steps 1–6 or Phase 2 when batch criteria are met.

## Phase 4 — Flag One Transaction

### Step 1: Identify the transaction
`bankFeed(action="fetch")`

Match by user-provided description, amount, or date.

Branching:
  0 matches → Prompt the user:
> I couldn’t find a bank-feed transaction matching that description, amount, or date. Please provide the transaction date, amount, and bank memo.
  1 match   → Store `tellerTransactionId`, amount, date, and bank memo; continue to Step 2.
  >1 matches → Prompt the user:
> I found multiple matching bank-feed transactions. Please specify the exact date, amount, or memo for the one you want to flag.

### Step 2: Flag with user reason and memo
`flagForReview(tellerTransactionId=tellerTransactionId, aiReasoning=aiReasoning)`

`aiReasoning` must include the user’s stated reason and the bank memo verbatim.

Branching:
  0 matches → Not applicable.
  1 match   → Store `reviewItemNumber`; continue to Step 3.
  >1 matches → Not applicable.

### Step 3: Mark flagged transaction recorded
`fetchWorkQueue(source="markRecorded", tellerTransactionId=tellerTransactionId, reviewItemNumber=reviewItemNumber)`

Branching:
  0 matches → Tell the user it was flagged but not marked recorded, and include it in the summary for follow-up.
  1 match   → Confirm:
> Flagged for CPA review with reason: [reason]
  >1 matches → Stop and escalate; multiple queue targets matched one bank-feed transaction.

## Phase 5 — Report Summary

### Step 1: Summarize processing results
Use stored results from Phases 1–4.

Branching:
  0 matches → Report that no transactions were processed.
  1 match   → Present the summary for the single transaction.
  >1 matches → Present totals and flagged-item table.

Format:

Bank Feed Summary — [Date]
═══════════════════════════
Processed: X transactions
Recorded:  Y ($total)
Flagged:   Z for CPA review
Skipped:   W (alreadyFlagged / duplicates)

Include a table of flagged items with `aiReasoning` summary and bank memo.

## Troubleshooting

WRITE SAFETY GUARD blocks write — `qbMasterData` or `qbFetchTransactions` was not called before a write → run Phase 1 Step 1 for entity lookup and Phase 1 Step 5 for duplicate check, then retry.

EXPENSE TYPE GUARD blocks `qbExpense` — outstanding Bills exist or bill/payable language indicates the vendor transaction should be a Bill or Bill Payment → run Phase 1 Step 2 and delegate to `accounts-payable` in Phase 1 Step 5.

CONSISTENCY RULE GUARD blocks write — a history-inferred account was used without explicitly passing all five criteria → rerun Phase 1 Step 3; if any criterion fails, run Phase 1 Step 4.

FLAG FOR REVIEW QUALITY GUARD blocks `flagForReview` — `aiReasoning` is missing, vague, or omits specific evidence → rebuild `aiReasoning` in Phase 1 Step 4 with entity name, history summary, transaction amount/date, review reason, and bank memo verbatim.

MULTI-VENDOR AMBIGUITY GUARD blocks write — `qbMasterData` returned multiple matching vendors or customers → run Phase 1 Step 4 with the full candidate list in `aiReasoning`.

DUPLICATE RESULT GUARD blocks write — `qbFetchTransactions` found same entity, date within 3 days, amount within 10%, and matching memo or `bankTransactionId` → do not write; run Phase 1 Step 5 branching for duplicate handling and Phase 1 Step 6 if it maps to the same bank-feed item.

BATCH SAFETY GUARD blocks `qbBatch` — master data is missing, transaction types are mixed, or duplicate coverage does not span all batch items → split by transaction type, rerun Phase 2 Step 2, then retry Phase 2 Step 3.

CURRENCY GUARD blocks write — transaction currency differs from source account currency and no `exchangeRate` or FX handling is present → run Phase 1 Step 4 with currency mismatch details.

SOURCE-CATEGORY COLLISION GUARD blocks write — source bank/credit card account equals a line category account → correct the source account or category account in Phase 1 Step 5.

`fetchWorkQueue(source="markRecorded")` fails — transaction may reappear in the next feed even though it was recorded or flagged → keep the `qbTransactionId` or `reviewItemNumber`, do not re-record, and retry Phase 1 Step 6.