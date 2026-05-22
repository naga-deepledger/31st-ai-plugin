---
name: accounts-payable
description: Enter vendor bills, pay existing bills, record immediate-pay expenses, apply vendor credits, and review AP aging. Activate when the user mentions bills, vendor payments, bill pay, vendor credits, outstanding payables, AP aging, or vendor spending.
---

## AP routing

| Situation | Tool path |
|-----------|-----------|
| Vendor invoice received, pay later | `qbBill` → `qbBillPayment` |
| Pay an existing outstanding bill | `qbBillPayment` |
| Expense already paid (card/ACH/check) | `qbExpense` |
| Vendor refund or credit memo | `qbCredit(creditType="vendor")` → `qbBillPayment` |
| Review outstanding AP, DPO, vendor spend | `qbReports(AgedPayables)` |

## Bill workflow

1. `qbMasterData(detailedInfo="vendor", filter=vendorName)` — resolve `vendorId`; >1 matches → MULTI-VENDOR AMBIGUITY GUARD blocks; ask or flag
2. **Open-doc check** — separate call, no date window:
   `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`
   Outstanding status is not date-bounded — a date-scoped call silently misses old open bills.
   - 1+ match that fits the request → switch to `qbBillPayment` workflow instead of creating a duplicate bill
3. **Account inference** — `qbFetchTransactions(entityId=vendorId, lookbackDays=365)` — use dominant account only if all 5 criteria pass: (a) ≥3 txns in last 365 days, (b) dominant ≥70%, (c) no second account ≥20%, (d) amount within 5× median, (e) most recent dominant txn <180 days old. Any criterion fails → ask or `flagForReview` (CONSISTENCY RULE GUARD)
4. `qbMasterData(entityTypes=["account"], filter=accountName)` — verify account ID
5. `qbBill(vendorId, txnDate, dueDate, lines=[{accountId, amount, description}], docNumber, memo)`
6. `qbAttachFile(entityType="Bill", entityId=billId, ...)` — attach invoice when available

## Bill payment workflow

1. Resolve vendor → find outstanding bills: `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`
2. Resolve payment account: `qbMasterData(entityTypes=["account"], filter=paymentAccountName)` — Bank or CC accounts only
3. `qbBillPayment(vendorId, paymentDate, bankAccountId, bills=[{billId, amount}], totalAmount, memo)`
4. `qbAttachFile(entityType="BillPayment", entityId=billPaymentId, ...)`

Batch (3+ same-type payments): `qbBatch(operationType="BillPayment", operations=[...])` — BATCH SAFETY GUARD requires homogeneous type + verified IDs + date-range duplicate check covering all items.

## Expense workflow

1. `qbMasterData` for vendor + source account (Bank or CC)
2. **Open-doc check first**: `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)` — EXPENSE TYPE GUARD blocks `qbExpense` if outstanding bill exists; route to `qbBillPayment` instead
3. **Bill language check**: if user says "received a bill", "net 30", "pay later", "on account", or "on credit" → route to bill workflow, not expense
4. Account inference — same 5-criteria rule as bill workflow
5. Verify source ≠ category: SOURCE-CATEGORY COLLISION GUARD blocks if they match
6. `qbExpense(paymentType, accountId=sourceAccountId, vendorId, txnDate, lines=[{accountId=categoryAccountId, amount}], memo)`
7. `qbAttachFile(entityType="Purchase", entityId=purchaseId, ...)` — **use `"Purchase"` not `"Expense"`** (QB API quirk for expense attachments)

## Vendor credit workflow

1. Resolve vendor + credit account
2. Duplicate check: `qbFetchTransactions(transactionType="VendorCredit", entityId=vendorId, ±3 days)`
3. `qbCredit(creditType="vendor", vendorId, txnDate, lines, memo)`
4. Apply via bill payment: `qbBillPayment(..., bills=[{billId, amount, txnType="Bill"}, {billId=vendorCreditId, amount, txnType="VendorCredit"}], totalAmount=netOrZero)`

## AP aging / vendor spend

`qbReports(reportType="AgedPayables", asOfDate)` + `qbReports(reportType="AgedPayablesDetail", asOfDate)`

DPO = AP / (Annual COGS / 365) — pull `qbReports(ProfitAndLoss, fiscalYear)` for annual COGS.

Flag: vendor spend up 20%+ without corresponding revenue growth. Delegate bank matching / reconciliation to bank-reconciliation skill.

## Gotchas

1. **Open-doc check must be a separate call with no date window** — outstanding status is not date-bounded; a scoped call silently misses bills older than the window
2. **EXPENSE TYPE GUARD blocks `qbExpense` when outstanding bill exists** — always run open-doc check before `qbExpense`
3. **Attachment entityType for Expense is `"Purchase"`** — not `"Expense"`; QB API uses the Purchase entity internally
4. **Bill language → route to `qbBill`** — "net 30", "pay later", "received a bill" signals a payable, not a paid expense
5. **CONSISTENCY RULE GUARD**: all 5 history criteria must pass or ask/flag — never use a history-inferred account if any criterion fails
