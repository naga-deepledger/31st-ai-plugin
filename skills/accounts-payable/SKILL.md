---
name: accounts-payable
description: Manage accounts payable by entering bills, paying vendors, recording immediate-pay expenses, applying vendor credits, reviewing AP aging, and analyzing vendor spending. Activate when the user mentions bills, vendor payments, AP aging, bill pay, vendor credits, outstanding payables, or vendor spending.
---

| Situation | Phase |
|---|---|
| Vendor invoice/bill to pay later | Phase 1 — Enter a vendor bill |
| Pay an existing bill or batch of bills | Phase 2 — Pay vendor bills |
| Card/ACH/check purchase already paid | Phase 3 — Record an immediate-pay expense |
| Vendor refund, adjustment, returned goods, or credit memo | Phase 4 — Apply a vendor credit |
| Outstanding/overdue AP, DPO, or vendor spend review | Phase 5 — Review AP aging and vendor spending |

## Phase 1 — Enter a vendor bill

### Step 1: Resolve the vendor
`qbMasterData(detailedInfo="vendor", filter=vendorName)`

Branching:
- 0 matches → In interactive mode, ask:
  > I couldn't find vendor "[vendorName]" in QuickBooks. Should I create or use a different vendor name?
- 1 match → Store `vendorId`, `vendorName`, and matched vendor display name.
- >1 matches → Do not pick one. In interactive mode, ask:
  > I found multiple vendors matching "[vendorName]": [candidate list]. Which vendor should I use?
  In batch/async mode, call `flagForReview(entityType="Vendor", entityName=vendorName, aiReasoning="Multiple vendor matches found for bill entry: [candidate list]. Need CPA/user confirmation before creating AP.")` (MULTI-VENDOR AMBIGUITY GUARD blocks writes when unresolved).

### Step 2: Check for an existing open bill
`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

Run this as a separate call with no date window before history fetch or account inference.

Branching:
- 0 matches → Store `openDocStatus="no outstanding bills"` and proceed.
- 1 match → If the open bill matches the vendor, amount, date, due date, document number, or memo from the user request, do not create a duplicate bill; switch to Phase 2 Step 3 using the returned `billId`. If it is unrelated, store the open bill list as context and proceed.
- >1 matches → If one or more open bills match the user request, switch to Phase 2 Step 3 using the matching `billId` values. If unclear, in interactive mode ask:
  > This vendor already has outstanding bills: [bill list with billId, date, due date, amount, balance]. Is this a new bill, or should I pay one of these existing bills?
  In batch/async mode, call `flagForReview(entityType="Bill", entityId=vendorId, aiReasoning="Vendor has multiple outstanding bills and the new bill request may duplicate an existing payable: [bill list].")`.

### Step 3: Infer or confirm the expense account
`qbFetchTransactions(entityId=vendorId, entityType="Vendor", lookbackDays=365)`

Use a history-inferred account only if all five consistency criteria pass: (a) ≥ 3 prior transactions in the last 365 days, (b) dominant account ≥ 70%, (c) no second account ≥ 20%, (d) current amount within 5× the median of dominant-account transactions, and (e) most recent dominant-account transaction < 180 days old.

Branching:
- 0 matches → In interactive mode, ask:
  > I don't have enough history for vendor "[vendorName]" to infer an expense category. Which expense account should I use for this bill?
  In batch/async mode, call `flagForReview(entityType="Vendor", entityId=vendorId, aiReasoning="No 365-day vendor history available to infer expense account for bill from [vendorName].")`.
- 1 match → If the five consistency criteria cannot all be evaluated and passed, use the same recovery as 0 matches.
- >1 matches → Store `historyTxnCount`, dominant account, dominant percentage, second-account percentage, median, most recent dominant-account date, and `accountBasis`. If all five criteria pass, store `accountId`; otherwise use the same recovery as 0 matches. (CONSISTENCY RULE GUARD blocks history-inferred account use unless these criteria were explicitly evaluated.)

### Step 4: Verify account IDs
`qbMasterData(entityTypes=["account"], filter=accountNameOrType)`

Branching:
- 0 matches → In interactive mode, ask:
  > I couldn't find the expense account "[accountNameOrType]" in QuickBooks. Which account should I use?
  In batch/async mode, call `flagForReview(entityType="Account", entityName=accountNameOrType, aiReasoning="Expense account for bill was not found in QuickBooks master data.")`.
- 1 match → Store `accountId` and account name.
- >1 matches → In interactive mode, ask:
  > I found multiple accounts matching "[accountNameOrType]": [candidate list]. Which account should I use for this bill?
  In batch/async mode, call `flagForReview(entityType="Account", entityName=accountNameOrType, aiReasoning="Multiple expense account matches found for bill coding: [candidate list].")`.

### Step 5: Confirm and record the bill
In month-end mode, propose the bill and do not post without approval.

Before the write in interactive mode, emit:

Pre-write evidence:
- Entity: [vendorName + vendorId]
- Open-doc: [no outstanding bills / billId X applied]
- Account basis: [N txns, XX% dominant / open-doc match]
- Mode: interactive

Then ask:
> Please confirm I should record this bill: vendor [vendorName], amount $[total], bill date [txnDate], due date [dueDate], expense category [accountName], memo/reference [docNumber or memo].

After confirmation:
`qbBill(vendorId=vendorId, txnDate=txnDate, dueDate=dueDate, lines=[{accountId=accountId, amount=amount, description=description}], docNumber=docNumber, memo=memo)`

Branching:
- 0 matches → If confirmation is denied or missing in interactive mode, do not write; ask what to change.
- 1 match → Store `billId`, `txnDate`, `dueDate`, `total`, and bill status.
- >1 matches → Not applicable for write result; if QuickBooks returns ambiguity or validation error, stop and use Troubleshooting.

### Step 6: Attach supporting documentation
`qbAttachFile(entityType="Bill", entityId=billId, fileSource=fileSource, fileName=fileName)`

Use a portal download, local file, drive file, or user upload when available; attachments are preferred for audit-ready books.

Branching:
- 0 matches → If no file is available, proceed without attachment and note “no attachment provided.”
- 1 match → Store `attachmentId`.
- >1 matches → In interactive mode, ask:
  > I found multiple possible bill documents: [file list]. Which one should I attach?
  In batch/async mode, call `flagForReview(entityType="Bill", entityId=billId, aiReasoning="Multiple possible bill attachments found; need confirmation of correct supporting document.")`.

## Phase 2 — Pay vendor bills

### Step 1: Resolve the vendor
`qbMasterData(detailedInfo="vendor", filter=vendorName)`

Branching:
- 0 matches → In interactive mode, ask:
  > I couldn't find vendor "[vendorName]" in QuickBooks. Which vendor should I use for this bill payment?
- 1 match → Store `vendorId`, `vendorName`, and matched vendor display name.
- >1 matches → Do not pick one. In interactive mode, ask:
  > I found multiple vendors matching "[vendorName]": [candidate list]. Which vendor should I pay?
  In batch/async mode, call `flagForReview(entityType="Vendor", entityName=vendorName, aiReasoning="Multiple vendor matches found for bill payment: [candidate list]. Need confirmation before paying AP.")`.

### Step 2: Find outstanding bills
`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

Skip this call only when Phase 1 Step 2 or Phase 3 Step 2 already returned the relevant outstanding `billId` values in this conversation.

Branching:
- 0 matches → In interactive mode, ask:
  > I don't see any outstanding bills for [vendorName]. Did you want to record an immediate payment as an Expense instead?
  In batch/async mode, call `flagForReview(entityType="Vendor", entityId=vendorId, aiReasoning="Bill payment requested but no outstanding bills were found for vendor [vendorName].")`.
- 1 match → Store `billId`, open balance, bill date, due date, and amount.
- >1 matches → Select bills by exact user-provided amount, date, due date, bill number, memo, or “pay all.” If unclear, in interactive mode ask:
  > [vendorName] has multiple outstanding bills: [bill list with billId, date, due date, amount, balance]. Which bill(s) should I pay, and should I pay the full balance or a partial amount?
  In batch/async mode, call `flagForReview(entityType="Bill", entityId=vendorId, aiReasoning="Multiple outstanding bills found for requested payment and payment allocation is unclear: [bill list].")`.

### Step 3: Choose the payment account
`qbMasterData(entityTypes=["account"], filter=paymentAccountNameOrType)`

Use Bank or credit card account IDs only.

Branching:
- 0 matches → In interactive mode, ask:
  > Which bank or credit card account should I use to pay [vendorName]?
  In batch/async mode, call `flagForReview(entityType="Account", entityName=paymentAccountNameOrType, aiReasoning="Payment account for bill payment was not found in QuickBooks master data.")`.
- 1 match → Store `bankAccountId`, payment account name, and account currency.
- >1 matches → In interactive mode, ask:
  > I found multiple payment accounts matching "[paymentAccountNameOrType]": [candidate list]. Which bank or credit card account should I use?
  In batch/async mode, call `flagForReview(entityType="Account", entityName=paymentAccountNameOrType, aiReasoning="Multiple possible payment accounts found for bill payment: [candidate list].")`.

### Step 4: Confirm and record the bill payment
Partial payments are allowed by paying less than the full bill balance; the bill remains partially outstanding. Multiple bills to one vendor may be paid in a single BillPayment using the `bills` array.

In month-end mode, propose the payment and do not post without approval.

Before the write in interactive mode, emit:

Pre-write evidence:
- Entity: [vendorName + vendorId]
- Open-doc: [billId X applied]
- Account basis: [open-doc match]
- Mode: interactive

Then ask:
> Please confirm I should record this bill payment: vendor [vendorName], payment date [paymentDate], payment account [paymentAccountName], bills [bill list with billId and amount applied], total payment $[totalPayment].

After confirmation:
`qbBillPayment(vendorId=vendorId, paymentDate=paymentDate, bankAccountId=bankAccountId, bills=[{billId=billId, amount=amount}], totalAmount=totalPayment, memo=memo)`

Branching:
- 0 matches → If confirmation is denied or missing in interactive mode, do not write; ask what to change.
- 1 match → Store `billPaymentId`, paid bill IDs, amount applied per bill, and payment account.
- >1 matches → Not applicable for write result; if QuickBooks returns ambiguity or validation error, stop and use Troubleshooting.

### Step 5: Attach remittance support
`qbAttachFile(entityType="BillPayment", entityId=billPaymentId, fileSource=fileSource, fileName=fileName)`

Attach remittance advice or bank confirmation when available; attachments are preferred for audit-ready books.

Branching:
- 0 matches → If no file is available, proceed without attachment and note “no attachment provided.”
- 1 match → Store `attachmentId`.
- >1 matches → In interactive mode, ask:
  > I found multiple possible payment confirmations: [file list]. Which one should I attach?
  In batch/async mode, call `flagForReview(entityType="BillPayment", entityId=billPaymentId, aiReasoning="Multiple possible bill payment attachments found; need confirmation of correct remittance or bank confirmation.")`.

### Step 6: Process a same-type batch of bill payments
`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

`qbMasterData(entityTypes=["account"], filter=paymentAccountNameOrType)`

`qbFetchTransactions(transactionType="BillPayment", startDate=batchStartDate, endDate=batchEndDate)`

`qbBatch(operationType="BillPayment", operations=[{vendorId=vendorId, paymentDate=paymentDate, bankAccountId=bankAccountId, bills=[{billId=billId, amount=amount}], totalAmount=totalPayment}])`

Branching:
- 0 matches → If any vendor has no outstanding bills, exclude that item and call `flagForReview(entityType="Vendor", entityId=vendorId, aiReasoning="Batch bill payment item has no outstanding bill to apply payment against.")`.
- 1 match → Include only validated BillPayment operations with verified vendor, account, outstanding bill, and duplicate-check coverage.
- >1 matches → If payment allocation is ambiguous for any vendor, exclude that item and call `flagForReview(entityType="Bill", entityId=vendorId, aiReasoning="Batch bill payment item has multiple outstanding bills and unclear allocation; not asking mid-batch.")`. (BATCH SAFETY GUARD blocks mixed transaction types, missing master data, or missing date-range duplicate check.)

## Phase 3 — Record an immediate-pay expense

### Step 1: Resolve vendor and source account
`qbMasterData(detailedInfo="vendor", filter=vendorName)`

`qbMasterData(entityTypes=["account"], filter=sourceAccountNameOrType)`

Branching:
- 0 matches → In interactive mode, ask:
  > I couldn't find the vendor or payment account for this expense. Which vendor and bank/credit card account should I use?
  In batch/async mode, call `flagForReview(entityType="Purchase", entityName=vendorName, aiReasoning="Immediate-pay expense is missing a resolved vendor or source bank/credit card account.")`.
- 1 match → Store `vendorId`, `vendorName`, `sourceAccountId`, source account name, and account currency.
- >1 matches → Do not pick one. In interactive mode, ask:
  > I found multiple matching vendors or accounts: [candidate list]. Which vendor and source account should I use?
  In batch/async mode, call `flagForReview(entityType="Purchase", entityName=vendorName, aiReasoning="Multiple vendor or source-account matches found for immediate-pay expense: [candidate list].")`.

### Step 2: Pre-satisfy the expense open-document check
`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

Run this before attempting `qbExpense`.

Branching:
- 0 matches → Store `openDocStatus="no outstanding bills"` and proceed.
- 1 match → Do not record an Expense. In interactive mode, ask:
  > This vendor has an outstanding bill for $[amount] dated [date], due [dueDate]. Should I record this as a Bill Payment against that bill instead?
  If yes, switch to Phase 2 Step 3 using returned `billId`. In batch/async mode, call `flagForReview(entityType="Bill", entityId=billId, aiReasoning="Immediate-pay expense request conflicts with outstanding bill for vendor [vendorName]; should likely be BillPayment, not Expense.")`.
- >1 matches → Do not record an Expense. In interactive mode, ask:
  > This vendor has outstanding bills: [bill list with billId, date, due date, amount, balance]. Should I apply this payment to one of those bills instead of recording a new Expense?
  In batch/async mode, call `flagForReview(entityType="Bill", entityId=vendorId, aiReasoning="Immediate-pay expense request conflicts with multiple outstanding bills for vendor [vendorName]; need allocation before recording.")`. (EXPENSE TYPE GUARD blocks qbExpense if outstanding bills exist.)

### Step 3: Check for bill language
If the user request says “received a bill,” “received an invoice from vendor,” “bill from,” “net 30,” “net 15,” “due in,” “pay later,” “on account,” or “on credit,” do not record an Expense.

Branching:
- 0 matches → No bill/payable language detected; proceed.
- 1 match → In interactive mode, ask:
  > It sounds like this hasn't been paid yet — should I record it as a Bill (owe now, pay later) instead of an Expense (already paid)?
  In batch/async mode, call `flagForReview(entityType="Purchase", entityName=vendorName, aiReasoning="User wording suggests a bill/payable rather than an already-paid expense: [matched phrase].")`.
- >1 matches → Same recovery as 1 match. (EXPENSE TYPE GUARD blocks qbExpense when bill language is present.)

### Step 4: Infer or confirm the expense category
`qbFetchTransactions(entityId=vendorId, entityType="Vendor", lookbackDays=365)`

Apply Phase 1 Step 3 for account inference.

Branching:
- 0 matches → In interactive mode, ask:
  > I don't have enough history for vendor "[vendorName]" to infer an expense category. Which expense account should I use?
  In batch/async mode, call `flagForReview(entityType="Vendor", entityId=vendorId, aiReasoning="No 365-day vendor history available to infer expense category for immediate-pay expense from [vendorName].")`.
- 1 match → If Phase 1 Step 3 criteria cannot all be evaluated and passed, use the same recovery as 0 matches.
- >1 matches → Store `historyTxnCount`, dominant account, dominant percentage, second-account percentage, median, most recent dominant-account date, and `accountBasis`. If Phase 1 Step 3 criteria pass, store `categoryAccountId`; otherwise use the same recovery as 0 matches.

### Step 5: Verify source and category accounts differ
`qbMasterData(entityTypes=["account"], filter=categoryAccountNameOrType)`

Branching:
- 0 matches → In interactive mode, ask:
  > I couldn't find expense category "[categoryAccountNameOrType]" in QuickBooks. Which category account should I use?
  In batch/async mode, call `flagForReview(entityType="Account", entityName=categoryAccountNameOrType, aiReasoning="Expense category account was not found in QuickBooks master data.")`.
- 1 match → Store `categoryAccountId`; if `sourceAccountId == categoryAccountId`, stop and ask:
  > The source account and expense category are both [accountName]. The source account is where money comes from, and the line category is what it was spent on. Which expense category should I use instead?
  In batch/async mode, call `flagForReview(entityType="Purchase", entityName=vendorName, aiReasoning="Source account equals expense category account for immediate-pay expense; would create zero-net entry.")`.
- >1 matches → In interactive mode, ask:
  > I found multiple category accounts matching "[categoryAccountNameOrType]": [candidate list]. Which expense category should I use?
  In batch/async mode, call `flagForReview(entityType="Account", entityName=categoryAccountNameOrType, aiReasoning="Multiple expense category account matches found: [candidate list].")`. (SOURCE-CATEGORY COLLISION GUARD blocks source/category account collisions.)

### Step 6: Confirm and record the expense
In month-end mode, propose the expense and do not post without approval.

Before the write in interactive mode, emit:

Pre-write evidence:
- Entity: [vendorName + vendorId]
- Open-doc: [no outstanding bills]
- Account basis: [N txns, XX% dominant / open-doc match]
- Mode: interactive

Then ask:
> Please confirm I should record this expense: vendor [vendorName], payment date [paymentDate], payment type [paymentType], paid from [sourceAccountName], amount $[amount], category [categoryAccountName], memo [memo].

After confirmation:
`qbExpense(paymentType=paymentType, accountId=sourceAccountId, vendorId=vendorId, txnDate=paymentDate, lines=[{accountId=categoryAccountId, amount=amount, description=description}], memo=memo)`

Branching:
- 0 matches → If confirmation is denied or missing in interactive mode, do not write; ask what to change.
- 1 match → Store `purchaseId`, `txnDate`, `amount`, source account, and category account.
- >1 matches → Not applicable for write result; if QuickBooks returns ambiguity or validation error, stop and use Troubleshooting.

### Step 7: Attach the receipt
`qbAttachFile(entityType="Purchase", entityId=purchaseId, fileSource=fileSource, fileName=fileName)`

Use entityType `"Purchase"` for Expense attachments in the QuickBooks API. Attach receipts from portal, local file, drive, or user upload when available; attachments are preferred for audit-ready books.

Branching:
- 0 matches → If no receipt is available, proceed without attachment and note “no attachment provided.”
- 1 match → Store `attachmentId`.
- >1 matches → In interactive mode, ask:
  > I found multiple possible receipts: [file list]. Which one should I attach?
  In batch/async mode, call `flagForReview(entityType="Purchase", entityId=purchaseId, aiReasoning="Multiple possible expense receipts found; need confirmation of correct support.")`.

## Phase 4 — Apply a vendor credit

### Step 1: Resolve vendor and credit account
`qbMasterData(detailedInfo="vendor", filter=vendorName)`

`qbMasterData(entityTypes=["account"], filter=creditAccountNameOrType)`

Branching:
- 0 matches → In interactive mode, ask:
  > I couldn't find the vendor or credit account for this vendor credit. Which vendor and account should I use?
  In batch/async mode, call `flagForReview(entityType="VendorCredit", entityName=vendorName, aiReasoning="Vendor credit is missing a resolved vendor or account.")`.
- 1 match → Store `vendorId`, `vendorName`, and `creditAccountId`.
- >1 matches → In interactive mode, ask:
  > I found multiple matching vendors or accounts: [candidate list]. Which vendor and account should I use for this credit?
  In batch/async mode, call `flagForReview(entityType="VendorCredit", entityName=vendorName, aiReasoning="Multiple vendor or credit account matches found for vendor credit: [candidate list].")`.

### Step 2: Check open bills and duplicates
`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

`qbFetchTransactions(transactionType="VendorCredit", entityId=vendorId, startDate=startDate, endDate=endDate)`

Branching:
- 0 matches → If no outstanding bills exist, create the credit on account and store `openDocStatus="no outstanding bills"`.
- 1 match → Store `billId`, open balance, date, due date, and amount for optional application.
- >1 matches → Store open bill list. In interactive mode, ask:
  > This vendor has multiple open bills: [bill list with billId, date, due date, amount, balance]. Should I apply the credit to one of these bills now, or leave it as an unapplied vendor credit?
  In batch/async mode, create the credit only if duplicate check is clear and leave application for review by calling `flagForReview(entityType="VendorCredit", entityName=vendorName, aiReasoning="Vendor credit can be created, but multiple open bills exist and credit application allocation is unclear.")`.

### Step 3: Confirm and create the vendor credit
In month-end mode, propose the credit and do not post without approval.

Before the write in interactive mode, emit:

Pre-write evidence:
- Entity: [vendorName + vendorId]
- Open-doc: [no outstanding bills / billId X available]
- Account basis: [credit account confirmed]
- Mode: interactive

Then ask:
> Please confirm I should create this vendor credit: vendor [vendorName], date [txnDate], amount $[amount], account [creditAccountName], reason [description].

After confirmation:
`qbCredit(creditType="vendor", vendorId=vendorId, txnDate=txnDate, lines=[{accountId=creditAccountId, amount=amount, description=description}], memo=memo)`

Branching:
- 0 matches → If confirmation is denied or missing in interactive mode, do not write; ask what to change.
- 1 match → Store `vendorCreditId`, amount, vendor, and account.
- >1 matches → Not applicable for write result; if QuickBooks returns ambiguity or validation error, stop and use Troubleshooting.

### Step 4: Apply the credit through a bill payment
Use `qbBillPayment` to apply the VendorCredit in the `bills` array with `txnType="VendorCredit"`.

Before the write in interactive mode, emit:

Pre-write evidence:
- Entity: [vendorName + vendorId]
- Open-doc: [billId X applied]
- Account basis: [open-doc match]
- Mode: interactive

Then ask:
> Please confirm I should apply vendor credit [vendorCreditId] for $[creditAmount] to bill [billId] for [vendorName]. Net payment will be $[netPaymentAmount] from [paymentAccountName], or $0 if the credit fully covers the bill.

If no cash payment is needed:
`qbBillPayment(vendorId=vendorId, paymentDate=paymentDate, bankAccountId=bankAccountId, bills=[{billId=billId, amount=billAmount, txnType="Bill"}, {billId=vendorCreditId, amount=creditAmount, txnType="VendorCredit"}], totalAmount=0, memo=memo)`

If a net cash payment is needed:
`qbBillPayment(vendorId=vendorId, paymentDate=paymentDate, bankAccountId=bankAccountId, bills=[{billId=billId, amount=billAmount, txnType="Bill"}, {billId=vendorCreditId, amount=creditAmount, txnType="VendorCredit"}], totalAmount=netPaymentAmount, memo=memo)`

Branching:
- 0 matches → If no open bill exists, leave the VendorCredit unapplied and note it is available for a future bill.
- 1 match → Store `billPaymentId`, applied `vendorCreditId`, applied `billId`, and net payment amount.
- >1 matches → In interactive mode, ask which bills to apply the credit against. In batch/async mode, call `flagForReview(entityType="VendorCredit", entityId=vendorCreditId, aiReasoning="Multiple possible bills exist for vendor credit application; allocation requires confirmation.")`.

## Phase 5 — Review AP aging and vendor spending

### Step 1: Pull AP aging reports
`qbReports(reportType="AgedPayables", asOfDate=asOfDate)`

`qbReports(reportType="AgedPayablesDetail", asOfDate=asOfDate)`

Branching:
- 0 matches → Tell the user there are no open payables as of `asOfDate`.
- 1 match → Store AP summary and line-item details.
- >1 matches → Consolidate by vendor and bill; present aging buckets Current, 1-30, 31-60, 61-90, and 90+.

### Step 2: Calculate DPO
`qbReports(reportType="ProfitAndLoss", startDate=fiscalYearStartDate, endDate=fiscalYearEndDate)`

Calculate `Days Payable Outstanding = AP / (Annual COGS / 365)`.

Branching:
- 0 matches → Present AP aging without DPO and state that Annual COGS was unavailable.
- 1 match → Store Annual COGS and calculate DPO.
- >1 matches → Use the annual Profit and Loss covering the requested fiscal year; if unclear, ask:
  > Which fiscal year should I use to calculate DPO?

### Step 3: Analyze aging concerns
Use the AP summary and detail report to flag:
- Bills past due → late payment penalties and vendor relationship risk.
- AP growing faster than revenue → cash flow pressure.
- Large upcoming payments → cash planning needed.

Branching:
- 0 matches → If no concerns are found, summarize current AP and upcoming due dates.
- 1 match → Present the concern, affected vendors, bill IDs, due dates, and amounts.
- >1 matches → Rank concerns by overdue age, amount, and vendor criticality.

### Step 4: Recommend payment priorities and term negotiations
Present bills to prioritize for payment and vendors to negotiate terms with.

Branching:
- 0 matches → State that no payment priority changes are recommended.
- 1 match → Recommend the single highest-priority action.
- >1 matches → Group recommendations into “pay now,” “schedule,” and “negotiate terms.”

### Step 5: Pull vendor spending report
`qbReports(reportType="VendorExpenses", startDate=currentPeriodStartDate, endDate=currentPeriodEndDate)`

`qbReports(reportType="VendorExpenses", startDate=priorPeriodStartDate, endDate=priorPeriodEndDate)`

Branching:
- 0 matches → State that no vendor spending data was found for the period.
- 1 match → Store current-period vendor spend.
- >1 matches → Compare current period vs prior period; identify top vendors by total spend, fast-growing vendors, and new vendors.

### Step 6: Flag vendor spend increases
Flag vendor spend up 20%+ without corresponding revenue growth.

Branching:
- 0 matches → No vendor spend spikes found.
- 1 match → Present vendor, current spend, prior spend, percentage increase, and revenue context.
- >1 matches → Rank vendors by percentage increase and dollar increase.

### Step 7: Delegate bank matching and reconciliation
If the user asks to match bill payments to bank feed transactions, reconcile payment clearing, or investigate bank statement differences, delegate to the bank reconciliation or bank-feed matching skill instead of duplicating that workflow.

Branching:
- 0 matches → Continue AP-only analysis.
- 1 match → Route to the relevant bank/reconciliation skill with `vendorId`, `billPaymentId`, payment account, amount, and date.
- >1 matches → Route each bank/reconciliation issue separately with the related payment IDs.

## Troubleshooting

WRITE SAFETY GUARD blocks qbBill, qbBillPayment, qbExpense, qbCredit, or qbBatch — missing `qbMasterData` lookup or missing `qbFetchTransactions` duplicate check → return to Phase 1 Step 1 and Step 2 for bills, Phase 2 Step 1 and Step 2 for bill payments, Phase 3 Step 1 and Step 2 for expenses, or Phase 4 Step 1 and Step 2 for vendor credits.

DUPLICATE RESULT GUARD blocks a write — qbFetchTransactions found same vendor/customer, date within 3 days, amount within 10%, and matching memo or bankTransactionId → show the matching transaction details and ask the user to confirm it is not a duplicate before returning to the relevant confirmation step.

EXPENSE TYPE GUARD blocks qbExpense — outstanding Bills exist or bill/payable language was detected → return to Phase 3 Step 2 or Step 3 and switch to Phase 2 if the payment should apply to an existing bill.

VENDOR/ACCOUNT RESOLUTION GUARD blocks a write — vendorId, customerId, or accountId was not returned by the most recent `qbMasterData` call → re-run `qbMasterData` in the relevant resolve step and verify the entity/account name before writing.

SOURCE-CATEGORY COLLISION GUARD blocks qbExpense — source bank/credit card account equals the line category account → return to Phase 3 Step 5 and select a different expense category.

CONSISTENCY RULE GUARD blocks a history-inferred account — the account basis was not explicitly evaluated or did not pass Phase 1 Step 3 → return to Phase 1 Step 3 or Phase 3 Step 4 and ask for confirmation or call `flagForReview`.

MATERIALITY GUARD blocks a transaction — amount is more than 5× the vendor/customer median from fetched history → call `flagForReview(entityType="Transaction", entityId=vendorId, aiReasoning="Amount $[amount] is more than 5x historical median $[median] for vendor [vendorName]; requires CPA review before posting.")`.

CURRENCY GUARD blocks a write — transaction currency differs from source account currency and no exchange rate was provided → add `exchangeRate=exchangeRate` where supported or use a journal entry with explicit currency conversion lines; do not auto-record cross-currency AP activity without FX handling.

BATCH SAFETY GUARD blocks qbBatch — mixed transaction types, missing master data, or missing date-range duplicate check → return to Phase 2 Step 6 and split batches by transaction type after verifying IDs and duplicate coverage.

FLAG FOR REVIEW QUALITY GUARD blocks flagForReview — `aiReasoning` is missing, under 20 characters, or generic → retry with specific context including vendor, amount, date, candidate list, ambiguity, and requested AP action.

QuickBooks validation error “Object Not Found” — billId, vendorCreditId, vendorId, or accountId is stale or not in the current company file → re-run `qbMasterData` and `qbFetchTransactions` in the relevant phase before retrying.

QuickBooks validation error “Amount exceeds open balance” — bill payment or vendor credit application exceeds the remaining bill balance → return to Phase 2 Step 2 or Phase 4 Step 4 and apply only up to the open balance.

Attachment error with Expense — wrong attachment entity type used → return to Phase 3 Step 7 and use `qbAttachFile(entityType="Purchase", entityId=purchaseId, fileSource=fileSource, fileName=fileName)`.