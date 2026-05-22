---
name: accounts-receivable
description: Create invoices, receive customer payments, record sales receipts, deposit Undeposited Funds, apply customer credits, and review AR aging. Activate when the user mentions invoices, customer payments, AR aging, collections, outstanding receivables, customer credits, refunds, or depositing Undeposited Funds.
---

## AR routing

| Situation | Tool path |
|-----------|-----------|
| Customer pays immediately, no invoice | `qbSalesReceipt` → `qbDeposit` |
| Send invoice, collect later | `qbInvoice` → `qbReceivePayment` → `qbDeposit` |
| Payment against existing invoice | `qbReceivePayment` → `qbDeposit` |
| Non-customer deposit (interest, reimbursement) | `qbDeposit` directly |
| Customer return or credit | `qbCredit(creditType="customer")` → apply in `qbReceivePayment` |
| Cash refund to customer | `qbRefundReceipt` |

**ReceivePayment does NOT put money in the bank** — it moves funds to Undeposited Funds. `qbDeposit` is the required second step to move to the bank account.

## Invoice workflow

1. `qbMasterData(entityType="Customer", name=customerName)` + `qbMasterData(entityType="Item", name=itemName)` + `qbMasterData(entityType="Term", name=termName)`; >1 customer matches → MULTI-VENDOR AMBIGUITY GUARD blocks; ask or flag
2. **Open-doc check** — separate call, no date window: `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`. Outstanding status not date-bounded; scoped call misses old invoices. 1+ match that fits → ask if this is a new invoice or a duplicate
3. **History inference** (if item/terms unknown): `qbFetchTransactions(entityId=customerId, lookbackDays=365)` — 5-criteria rule: (a) ≥3 prior invoices in 365 days, (b) dominant item/terms ≥70%, (c) no second ≥20%, (d) amount within 5× median, (e) most recent <180 days old (CONSISTENCY RULE GUARD)
4. Duplicate check: `qbFetchTransactions(transactionType="Invoice", entityId=customerId, ±3 days)`
5. If context says "already paid" → route to SalesReceipt (INCOME TYPE GUARD blocks `qbInvoice` when payment received)
6. `qbInvoice(customerId, txnDate, dueDate, lines, termId, emailStatus="NeedToSend")`
7. `qbAttachFile(entityType="Invoice", entityId=invoiceId, ...)`

## ReceivePayment workflow

1. `qbMasterData(entityType="Customer", name)` + `qbMasterData(entityType="Account", name="Undeposited Funds")`
2. Open-doc check: `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)` — INCOME TYPE GUARD blocks `qbReceivePayment` if no outstanding invoice; route to SalesReceipt or Deposit
3. Duplicate check: `qbFetchTransactions(transactionType="Payment", entityId=customerId, ±3 days)`
4. Allocate: exact match → apply full amount; partial → leaves invoice partially outstanding; overpayment → excess becomes unapplied credit (returned as `unappliedAmount`)
5. `qbReceivePayment(customerId, totalAmount, paymentDate, invoices=[{invoiceId, amount}], creditMemos=[])`
6. `qbAttachFile(entityType="Payment", entityId=paymentId, ...)`

## SalesReceipt workflow

1. `qbMasterData` for customer + item + "Undeposited Funds" account
2. Open-doc check: `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)` — INCOME TYPE GUARD blocks `qbSalesReceipt` if outstanding invoice exists (would double-count income, leave AR unpaid); route to `qbReceivePayment` instead
3. Duplicate check ±3 days
4. `qbSalesReceipt(customerId, txnDate, depositToAccountId=undepositedFundsId, lines)`

## Deposit workflow

1. `qbMasterData` for "Undeposited Funds" + target bank account
2. `qbFetchTransactions(accountId=undepositedFundsId, startDate, endDate)` — find pending items
3. Group items to match bank statement lump sum (multiple payments often deposit as one line)
4. Duplicate check ±3 days
5. DEPOSIT TYPE GUARD blocks `qbDeposit` if used to close an invoice or record a single customer sale (must go through `qbReceivePayment` or `qbSalesReceipt` first)
6. `qbDeposit(depositAccountId=bankAccountId, txnDate, lines=linkedUndepositedFundsLines)`

## Customer credit / refund

Credit memo → apply in next `qbReceivePayment` using `creditMemos=[{creditMemoId, amount}]`
Cash refund → `qbRefundReceipt(customerId, depositAccountId=bankAccountId, txnDate, lines)`

## AR aging

`qbReports(reportType="AgedReceivables", asOfDate)` + `qbReports(reportType="AgedReceivablesDetail", asOfDate)`

DSO = AR / (Annual Revenue / 365)

Flag: invoices >90 days → discuss bad debt write-off with CPA; DSO increase month-over-month → collections risk; single customer >30% of AR → concentration risk. Delegate bad debt JE to journal-entries skill.

## Gotchas

1. **`qbReceivePayment` → Undeposited Funds, not bank** — always follow with `qbDeposit` to move funds to bank
2. **Open-doc check must be separate, no date window** — scoped calls miss old open invoices
3. **INCOME TYPE GUARD**: SalesReceipt when outstanding invoice exists = double-counted income; Invoice when already paid = wrong tool; ReceivePayment when no invoice = wrong tool
4. **DEPOSIT TYPE GUARD**: `qbDeposit` cannot close an invoice or record a direct customer sale — go through `qbReceivePayment` or `qbSalesReceipt` first
5. **Undeposited Funds grouping** — multiple payments often appear as one lump sum on the bank statement; match to bank line before calling `qbDeposit`
