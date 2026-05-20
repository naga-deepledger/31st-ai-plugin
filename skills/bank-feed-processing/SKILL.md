---
name: bank-feed-processing
description: Process bank feed transactions — fetch raw bank data, look up vendor history in QuickBooks, record high-confidence transactions, and flag unknowns for CPA review. Use when the user mentions bank feed, bank transactions, categorize transactions, or auto-categorize.
---

# Bank Feed Processing Skill

Process unrecorded bank and credit card transactions from the connected bank feed. Resolve each transaction against QuickBooks history to infer the correct account, record where confident, and flag the rest for CPA review.

## Trigger

Activate when the user wants to:
- Process bank feed transactions
- Categorize bank transactions
- Review unrecorded transactions
- Flag transactions for CPA review

## Workflow: Process Bank Feed

### Step 1: Fetch Transactions
```
bankFeed(action="fetch")
```
Returns raw bank transactions. Check `alreadyFlagged=true` on each item — those are already in the CPA review queue, skip them entirely; do not flag again.

### Step 2: For Each Transaction — Resolve and Evaluate

1. **Resolve vendor** — `qbMasterData(detailedInfo="vendor", filter=counterpartyName)` → get `vendorId`
2. **Fetch QB history** — `qbFetchTransactions(entityId=vendorId, entityType="Vendor", startDate, endDate)` — this one call answers three questions:
   - What account has this vendor been categorized to before? → use that account (consistent history = record)
   - Is there an outstanding Bill? → use `qbBillPayment` not `qbExpense`
   - Is there an outstanding Invoice? → use `qbReceivePayment` not `qbDeposit`
   - Does an identical transaction already exist (same amount + date)? → duplicate, skip
3. **No QB history** → `flagForReview` — do not guess

### Step 3: Record or Flag

**Consistent QB history:**
1. `qbMasterData` — get source account ID (bank/CC)
2. Record with the account from history using the correct tool
3. `fetchWorkQueue(source="markRecorded")` — prevent re-processing

**No history or ambiguous:**
1. `flagForReview` with specific `aiReasoning`:
   - "Vendor not found in QuickBooks — no transaction history"
   - "Multiple past accounts used for this vendor: [list]"
   - "Description ambiguous: [description]"
2. Include `suggestedCategory` when you have a reasonable guess

### Step 4: Batch When Possible

When 3+ transactions share the same type and source account:
1. Group by transaction type (Expense, Deposit, etc.)
2. Run one `qbMasterData` lookup for all vendor/customer IDs
3. Run one `qbFetchTransactions` duplicate check covering the full date range
4. Submit via `qbBatch`
5. Check response for per-item success/failure
6. Retry failed items individually

### Step 5: Report Summary

Present results:
```
Bank Feed Summary — [Date]
═══════════════════════════
Processed: X transactions
Recorded:  Y ($total)
Flagged:   Z for CPA review
Skipped:   W (duplicates)
```

Include a table of flagged items with reasoning.

## Workflow: Review Mode

When the user wants to see transactions before recording:

1. `bankFeed(action="fetch")` — pull all unprocessed
2. Present in a table with: date, description, amount, suggested category, confidence level
3. Let the user pick which to process
4. Do NOT record anything until explicitly told

## Workflow: Flag a Specific Transaction

When the user wants to flag a particular transaction:

1. Identify the transaction by description, amount, or date
2. `flagForReview(tellerTransactionId=..., aiReasoning=...)` with the user's reason
3. Confirm: "Flagged for CPA review with reason: [reason]"

## Workflow: Process CPA-Approved Items

CPA-approved items from the review queue take priority:

1. `fetchWorkQueue(source="approvedReviews")` — get approved items
2. Record using the `effectiveCategory` from the CPA's approval
3. Mark recorded: `fetchWorkQueue(source="markRecorded", reviewItemNumber=N, qbTransactionId=ID)`
4. **Never override** a CPA-approved category with agent judgment

## Safety Checklist

- [ ] `alreadyFlagged` transactions skipped — not re-flagged
- [ ] Vendor resolved via `qbMasterData` before fetching history
- [ ] QB history fetched via `qbFetchTransactions` — account inferred, not guessed
- [ ] Outstanding bills/invoices checked — correct tool selected
- [ ] Duplicate check — identical transaction not already in QB
- [ ] CPA-approved items processed first and categories preserved
- [ ] Transactions marked as recorded to prevent re-processing

## Common Mistakes to Avoid

- Re-flagging a transaction that already has `alreadyFlagged=true` — check this field first
- Guessing an account without checking QB history — history is the source of truth
- Recording an expense when an outstanding bill exists → use BillPayment
- Recording a deposit when an outstanding invoice exists → use ReceivePayment
- Not marking transactions as recorded → they reappear in the next fetch
- Overriding a CPA-approved category with a different agent guess
