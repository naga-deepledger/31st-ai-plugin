---
name: accounts-receivable
description: Manage accounts receivable by creating invoices, recording customer payments, sales receipts, deposits, credits, refunds, aging reviews, and collections follow-up. Use when the user mentions invoices, customer payments, AR aging, collections, outstanding receivables, customer credits, refunds, or depositing Undeposited Funds.
---

Situation | Start at
---|---
Customer owes money later | Phase 1 — Create invoice
Customer pays an existing invoice | Phase 2 — Receive customer payment
Customer pays immediately with no invoice | Phase 3 — Record sales receipt
Move Undeposited Funds to bank | Phase 4 — Deposit customer funds
Customer credit, refund, aging, or collections | Phase 5 or Phase 6

## Phase 1 — Create invoice

### Step 1: Resolve customer, items, and terms
`qbMasterData(entityType="Customer", name=customerName)`

`qbMasterData(entityType="Item", name=itemOrServiceName)`

`qbMasterData(entityType="Term", name=salesTermName)`

Branching:
  0 matches → Interactive: ask:
  > I couldn’t find this customer/item/term in QuickBooks. Should I create it elsewhere first, or can you provide the exact QuickBooks name?
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Customer, item, or sales term not found in QuickBooks for requested invoice; cannot create AR transaction without valid IDs.")`
  
  1 match → Store `customerId`, `itemId`, and `termId`; proceed.
  
  >1 matches → Interactive: present the candidate names and IDs; ask:
  > I found multiple QuickBooks matches for this customer or item. Which one should I use? Please reply with the exact name or ID.
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Multiple QuickBooks customer or item matches found for invoice; do not pick one automatically because AR aging depends on the correct customerId.")`  
  `(MULTI-VENDOR AMBIGUITY GUARD blocks qbInvoice here until exactly one customer is resolved)`

### Step 2: Check open invoices separately before history
`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

Run this as a separate call with no date window before any history fetch or item inference; outstanding status is not date-bounded and a date-scoped history call can miss existing open invoices.

Branching:
  0 matches → Store `openDocStatus="no outstanding invoices"`; proceed.
  
  1 match → If the outstanding invoice appears to be for the same service, amount, period, PO, or memo, Interactive: ask:
  > This customer already has an outstanding invoice that may match this request: invoice #[invoiceNumber], dated [date], due [dueDate], balance $[balance]. Should I create a new invoice, update the existing invoice, or apply a payment to the existing invoice?
  
  Batch/async: `flagForReview(entityType="Invoice", entityId=invoiceId, aiReasoning="Possible duplicate open invoice found before creating new invoice: same customer and similar service/amount/period; user confirmation required.")`
  
  >1 matches → If one clearly matches the requested service/amount/period, use that as the candidate and ask the same prompt; otherwise Interactive: ask:
  > This customer has multiple outstanding invoices. Which invoice, if any, relates to this request, or should I create a new invoice?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Multiple outstanding invoices exist for customer; cannot determine whether requested invoice is new or duplicate.")`

### Step 3: Infer invoice items and terms from customer history
`qbFetchTransactions(entityId=customerId, entityType="Customer", lookbackDays=365)`

Apply the CONSISTENCY RULE before auto-populating line items or sales terms from history: use inferred items/terms only if all five pass: (a) ≥ 3 prior invoices in last 365 days, (b) dominant item/terms ≥ 70%, (c) no second item/terms ≥ 20%, (d) current amount within 5× median, (e) most recent dominant item/terms transaction < 180 days old. `(CONSISTENCY RULE GUARD blocks qbInvoice here if a history-inferred account/item/term is used without confirming all five criteria)`

Branching:
  0 matches → Interactive: ask:
  > I don’t have prior invoice history for this customer. What item/service, amount, and payment terms should I use?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="No prior invoice history for customer; cannot infer invoice item, amount pattern, or terms.")`
  
  1 match → Do not infer a recurring pattern from one transaction; Interactive: ask:
  > I found only one prior invoice, which is not enough to infer a pattern. Please confirm the item/service, amount, and terms for this invoice.
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Only one prior invoice found; insufficient history to infer item, amount, or terms under consistency rule.")`
  
  >1 matches → If all five consistency criteria pass, store `inferredItemId`, `inferredTerms`, `medianAmount`, and `historyBasis`; otherwise Interactive: ask:
  > Customer history is not consistent enough to auto-fill this invoice. Please confirm the item/service, amount, and payment terms.
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Customer invoice history failed consistency criteria for auto-populating item or terms; CPA/user confirmation required.")`

### Step 4: Check for duplicate invoice near the transaction date
`qbFetchTransactions(transactionType="Invoice", entityId=customerId, startDate=txnDateMinus3Days, endDate=txnDatePlus3Days)`

Branching:
  0 matches → Proceed.
  
  1 match → If same customer, date within 3 days, amount within 10%, and matching memo text or bankTransactionId, Interactive: ask:
  > I found a possible duplicate invoice: #[invoiceNumber], dated [date], amount $[amount], memo “[memo]”. Please confirm this is not a duplicate before I create another invoice.
  
  Batch/async: `flagForReview(entityType="Invoice", entityId=invoiceId, aiReasoning="Potential duplicate invoice found: same customer, date within 3 days, amount within 10%, and matching memo/bankTransactionId.")`
  
  >1 matches → Interactive: show all possible duplicates and ask:
  > I found multiple possible duplicate invoices near this date and amount. Which one is correct, or should I create a new invoice?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Multiple possible duplicate invoices found near requested date and amount; cannot safely create another invoice.")`  
  `(DUPLICATE RESULT GUARD blocks qbInvoice here if a true duplicate is present)`

### Step 5: Build, confirm, and create the invoice
If the user’s request indicates payment was already received — “paid,” “sent payment,” “paid on the spot,” “already paid,” “received payment,” or “deposited” — route to Phase 3 instead. `(INCOME TYPE GUARD blocks qbInvoice here when payment was already received)`

Interactive confirmation prompt:
> Please confirm this invoice before I create it:
> - Customer: [customerName] ([customerId])
> - Date: [txnDate]
> - Due date: [dueDate]
> - Terms: [terms]
> - Line items: [lineSummary]
> - Total: $[totalAmount]
> - Email status: [emailStatus or not set]

Interactive mode: immediately before the write, emit:

Pre-write evidence:
- Entity: [customerName + customerId]
- Open-doc: [no outstanding invoices / invoiceId X reviewed]
- Account basis: [N txns, XX% dominant / open-doc match]
- Mode: interactive

`qbInvoice(customerId=customerId, txnDate=txnDate, dueDate=dueDate, lines=invoiceLines, termId=termId, emailStatus="NeedToSend")`

Branching:
  0 matches → If required fields are missing, return to Phase 1 Step 1 or Step 3; otherwise ask:
  > I’m missing required invoice fields: [missingFields]. Please provide them before I create the invoice.
  
  1 match → Store `invoiceId`, `invoiceNumber`, `totalAmount`, and `dueDate`; proceed to attachment if support exists.
  
  >1 matches → Not applicable for a create response; if QuickBooks returns multiple candidate references, stop and `flagForReview(entityType="Invoice", entityName=customerName, aiReasoning="QuickBooks returned ambiguous invoice creation response; verify created invoice before continuing.")`  
  `(WRITE SAFETY GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, SOURCE-CATEGORY COLLISION GUARD, MATERIALITY GUARD, CURRENCY GUARD, and INCOME TYPE GUARD validate qbInvoice here)`

### Step 6: Attach invoice support
`qbAttachFile(entityType="Invoice", entityId=invoiceId, fileSource=fileSource, fileId=fileId)`

Use for signed contracts, purchase orders, supporting documents from a portal, local file, drive, or user upload; preferred for audit-ready books.

Branching:
  0 matches → No support available; proceed without attachment and note that no backup was attached.
  
  1 match → Store `attachmentId`; invoice workflow complete.
  
  >1 matches → Interactive: ask:
  > I found multiple possible support files. Which file should I attach to invoice #[invoiceNumber]?
  
  Batch/async: `flagForReview(entityType="Invoice", entityId=invoiceId, aiReasoning="Multiple possible support files found for invoice; cannot determine correct attachment automatically.")`

## Phase 2 — Receive customer payment

### Step 1: Resolve customer and payment accounts
`qbMasterData(entityType="Customer", name=customerName)`

`qbMasterData(entityType="Account", name="Undeposited Funds")`

Branching:
  0 matches → Interactive: ask:
  > I couldn’t find this customer or the Undeposited Funds account in QuickBooks. Please provide the exact QuickBooks customer name.
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Customer or Undeposited Funds account not found; cannot record ReceivePayment safely.")`
  
  1 match → Store `customerId` and `undepositedFundsAccountId`; proceed.
  
  >1 matches → Interactive: ask:
  > I found multiple matching customers. Which customer should receive this payment?
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Multiple matching customers found for ReceivePayment; cannot choose one automatically.")`  
  `(MULTI-VENDOR AMBIGUITY GUARD blocks qbReceivePayment here until exactly one customer is resolved)`

### Step 2: Find open invoices to apply the payment
`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

Branching:
  0 matches → Interactive: ask:
  > No outstanding invoices were found for this customer. Is this a new customer sale that should be recorded as a Sales Receipt, or is it general non-customer income that should be recorded as a Deposit?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="No outstanding invoices found for customer; ReceivePayment requires an invoice, so route must be confirmed.")`
  
  1 match → Store `invoiceId`, `invoiceNumber`, `invoiceBalance`, `dueDate`; proceed.
  
  >1 matches → Match by invoice number, amount, date, remittance memo, or customer instructions. If ambiguous, Interactive: ask:
  > This customer has multiple open invoices. Which invoice(s) should this payment apply to? Please specify invoice number(s) and amounts.
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Multiple open invoices found and payment remittance does not identify which invoice(s) to apply.")`  
  `(INCOME TYPE GUARD blocks qbReceivePayment here if no outstanding invoice exists)`

### Step 3: Check for duplicate payment
`qbFetchTransactions(transactionType="Payment", entityId=customerId, startDate=paymentDateMinus3Days, endDate=paymentDatePlus3Days)`

Branching:
  0 matches → Proceed.
  
  1 match → If same customer, date within 3 days, amount within 10%, and matching memo text or bankTransactionId, Interactive: ask:
  > I found a possible duplicate payment: dated [date], amount $[amount], customer [customerName], memo “[memo]”. Please confirm this is not already recorded before I proceed.
  
  Batch/async: `flagForReview(entityType="Payment", entityId=paymentId, aiReasoning="Potential duplicate ReceivePayment found: same customer, date within 3 days, amount within 10%, and matching memo/bankTransactionId.")`
  
  >1 matches → Interactive: list possible duplicates and ask:
  > I found multiple possible duplicate payments. Which one, if any, is the payment you want me to record?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Multiple possible duplicate payments found near requested payment date and amount.")`  
  `(DUPLICATE RESULT GUARD blocks qbReceivePayment here if a true duplicate is present)`

### Step 4: Allocate payment, partial payment, overpayment, and credits
If payment is less than the selected invoice balance, apply the amount received and leave the invoice partially outstanding. If payment equals the selected invoice balance, close the invoice. If payment exceeds selected invoice balance, first check whether the excess applies to other open invoices; if not, QuickBooks creates an unapplied credit and the response `unappliedAmount` shows the excess.

For existing customer credits, include them in the next ReceivePayment using `creditMemos`.

Branching:
  0 matches → No selected invoice or credit allocation; return to Phase 2 Step 2.
  
  1 match → Store `invoices=[{invoiceId=invoiceId, amount=appliedAmount}]` and `creditMemos=creditMemoArrayOrEmpty`.
  
  >1 matches → Allocate across selected invoices by invoice number and amount. If unclear, Interactive: ask:
  > The payment can apply to multiple invoices. Please confirm the invoice numbers and amount to apply to each. If the payment exceeds all selected invoices, should the excess remain as an unapplied credit?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Payment allocation across multiple invoices or unapplied overpayment is ambiguous; cannot allocate automatically.")`

### Step 5: Record the ReceivePayment
Interactive mode: immediately before the write, emit:

Pre-write evidence:
- Entity: [customerName + customerId]
- Open-doc: [invoiceId X applied]
- Account basis: [open-doc match]
- Mode: interactive

`qbReceivePayment(customerId=customerId, totalAmount=paymentAmount, paymentDate=paymentDate, invoices=invoices, creditMemos=creditMemos)`

Branching:
  0 matches → If QuickBooks rejects because no invoice is linked, return to Phase 2 Step 2. If amount allocation is invalid, return to Phase 2 Step 4.
  
  1 match → Store `paymentId`, `appliedAmount`, and `unappliedAmount`. Remind the user that ReceivePayment reduces AR and increases Undeposited Funds; it does not put money in the bank.
  
  >1 matches → Not applicable for a create response; if QuickBooks returns an ambiguous response, `flagForReview(entityType="Payment", entityName=customerName, aiReasoning="QuickBooks returned ambiguous ReceivePayment response; verify payment posting and invoice application.")`  
  `(WRITE SAFETY GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, DUPLICATE RESULT GUARD, INCOME TYPE GUARD, CONSISTENCY RULE GUARD, and CURRENCY GUARD validate qbReceivePayment here)`

### Step 6: Attach payment support
`qbAttachFile(entityType="Payment", entityId=paymentId, fileSource=fileSource, fileId=fileId)`

Use for bank remittance, ACH confirmation, card settlement detail, or payment confirmation; preferred for audit-ready books.

Branching:
  0 matches → No support available; proceed without attachment and note that no backup was attached.
  
  1 match → Store `attachmentId`; payment workflow complete.
  
  >1 matches → Interactive: ask:
  > I found multiple possible remittance files. Which one should I attach to this payment?
  
  Batch/async: `flagForReview(entityType="Payment", entityId=paymentId, aiReasoning="Multiple possible remittance attachments found for customer payment; cannot choose automatically.")`

## Phase 3 — Record sales receipt

### Step 1: Resolve customer, items, and Undeposited Funds
`qbMasterData(entityType="Customer", name=customerName)`

`qbMasterData(entityType="Item", name=itemOrServiceName)`

`qbMasterData(entityType="Account", name="Undeposited Funds")`

Branching:
  0 matches → Interactive: ask:
  > I couldn’t find this customer, item/service, or Undeposited Funds in QuickBooks. Please provide the exact QuickBooks name.
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Missing customer, item/service, or Undeposited Funds account for SalesReceipt; cannot record customer sale safely.")`
  
  1 match → Store `customerId`, `itemId`, and `undepositedFundsAccountId`; proceed.
  
  >1 matches → Interactive: ask:
  > I found multiple matching customers or items. Which exact customer and item/service should I use?
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Multiple customer or item matches found for SalesReceipt; cannot choose automatically.")`

### Step 2: Verify no outstanding invoice should be paid instead
`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

Branching:
  0 matches → Proceed with SalesReceipt.
  
  1 match → Interactive: ask:
  > This customer has outstanding invoice #[invoiceNumber] for $[balance], dated [date], due [dueDate]. Should I apply this as a Receive Payment against that invoice instead of recording a Sales Receipt?
  
  Batch/async: `flagForReview(entityType="Invoice", entityId=invoiceId, aiReasoning="Customer has outstanding invoice; SalesReceipt would double-count income and leave invoice unpaid.")`
  
  >1 matches → Interactive: ask:
  > This customer has multiple outstanding invoices. Should this payment be applied to one or more invoices as a Receive Payment instead of a Sales Receipt?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Customer has multiple outstanding invoices; SalesReceipt may double-count income and leave AR unpaid.")`  
  `(INCOME TYPE GUARD blocks qbSalesReceipt here when outstanding invoices exist)`

### Step 3: Infer receipt items from history when needed
`qbFetchTransactions(entityId=customerId, entityType="Customer", lookbackDays=365)`

Use Phase 1 Step 3 for the required consistency evaluation before using any history-inferred item, term, or account.

Branching:
  0 matches → Interactive: ask:
  > I don’t have prior sales history for this customer. What item/service and amount should I use for the Sales Receipt?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="No prior customer sales history; cannot infer SalesReceipt item or account.")`
  
  1 match → Interactive: ask:
  > I found only one prior customer transaction, which is not enough to infer a pattern. Please confirm the item/service and amount.
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Only one prior customer transaction; insufficient history to infer SalesReceipt item or account.")`
  
  >1 matches → If Phase 1 Step 3 criteria pass, store `inferredItemId` and `historyBasis`; otherwise ask/flag using Phase 1 Step 3 recovery.

### Step 4: Check for duplicate SalesReceipt
`qbFetchTransactions(transactionType="SalesReceipt", entityId=customerId, startDate=txnDateMinus3Days, endDate=txnDatePlus3Days)`

Branching:
  0 matches → Proceed.
  
  1 match → If same customer, date within 3 days, amount within 10%, and matching memo text or bankTransactionId, Interactive: ask:
  > I found a possible duplicate Sales Receipt: dated [date], amount $[amount], memo “[memo]”. Please confirm this is not already recorded before I proceed.
  
  Batch/async: `flagForReview(entityType="SalesReceipt", entityId=salesReceiptId, aiReasoning="Potential duplicate SalesReceipt found: same customer, date within 3 days, amount within 10%, and matching memo/bankTransactionId.")`
  
  >1 matches → Interactive: list possible duplicates and ask:
  > I found multiple possible duplicate Sales Receipts. Which one, if any, matches this sale?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Multiple possible duplicate SalesReceipts found near requested sale date and amount.")`

### Step 5: Create the SalesReceipt
Interactive mode: immediately before the write, emit:

Pre-write evidence:
- Entity: [customerName + customerId]
- Open-doc: [no outstanding invoices]
- Account basis: [N txns, XX% dominant / open-doc match]
- Mode: interactive

`qbSalesReceipt(customerId=customerId, txnDate=txnDate, depositToAccountId=undepositedFundsAccountId, lines=salesReceiptLines)`

Branching:
  0 matches → Missing required fields; return to Phase 3 Step 1 or Step 3.
  
  1 match → Store `salesReceiptId`, `totalAmount`, and `undepositedFundsAccountId`; remind the user that funds are in Undeposited Funds until Phase 4 deposits them to the bank.
  
  >1 matches → Not applicable for a create response; if ambiguous, `flagForReview(entityType="SalesReceipt", entityName=customerName, aiReasoning="QuickBooks returned ambiguous SalesReceipt creation response; verify before deposit.")`  
  `(WRITE SAFETY GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, DUPLICATE RESULT GUARD, INCOME TYPE GUARD, SOURCE-CATEGORY COLLISION GUARD, MATERIALITY GUARD, CONSISTENCY RULE GUARD, and CURRENCY GUARD validate qbSalesReceipt here)`

## Phase 4 — Deposit customer funds

### Step 1: Resolve deposit accounts
`qbMasterData(entityType="Account", name="Undeposited Funds")`

`qbMasterData(entityType="Account", name=bankAccountName)`

Branching:
  0 matches → Interactive: ask:
  > I couldn’t find Undeposited Funds or the bank account in QuickBooks. Which exact bank account should receive this deposit?
  
  Batch/async: `flagForReview(entityType="Account", entityName=bankAccountName, aiReasoning="Undeposited Funds or target bank account not found; cannot create customer deposit.")`
  
  1 match → Store `undepositedFundsAccountId` and `bankAccountId`; proceed.
  
  >1 matches → Interactive: ask:
  > I found multiple matching bank accounts. Which bank account should receive the deposit?
  
  Batch/async: `flagForReview(entityType="Account", entityName=bankAccountName, aiReasoning="Multiple bank accounts match requested deposit account; cannot choose automatically.")`

### Step 2: Find pending Undeposited Funds items
`qbFetchTransactions(accountId=undepositedFundsAccountId, startDate=startDate, endDate=endDate)`

Branching:
  0 matches → If the context involves a customer with an outstanding invoice, route to Phase 2; otherwise Interactive: ask:
  > I don’t see pending customer payments or Sales Receipts in Undeposited Funds for this period. Is this a non-customer deposit, or should I first record a Receive Payment or Sales Receipt?
  
  Batch/async: `flagForReview(entityType="Account", entityId=undepositedFundsAccountId, aiReasoning="No Undeposited Funds items found for requested deposit period; cannot link deposit lines to customer payments.")`
  
  1 match → Store `paymentOrSalesReceiptId`, `amount`, and `txnType`; proceed.
  
  >1 matches → Store candidate items and proceed to grouping.

### Step 3: Group items to match the bank statement
Match deposits to lump sums on the bank statement; multiple payments often deposit as one bank line.

Branching:
  0 matches → Interactive: ask:
  > The Undeposited Funds items do not match the bank statement deposit amount. Which payments should be included in this deposit?
  
  Batch/async: `flagForReview(entityType="Account", entityId=bankAccountId, aiReasoning="Undeposited Funds items do not reconcile to requested bank statement deposit amount.")`
  
  1 match → Store `linkedUndepositedFundsLines` and `depositTotal`; proceed.
  
  >1 matches → Interactive: ask:
  > I found multiple combinations that could match the bank deposit of $[bankDepositAmount]. Which payments should be included?
  
  Batch/async: `flagForReview(entityType="Account", entityId=bankAccountId, aiReasoning="Multiple Undeposited Funds groupings match the bank deposit amount; cannot select automatically.")`

### Step 4: Check for duplicate deposit
`qbFetchTransactions(transactionType="Deposit", accountId=bankAccountId, startDate=depositDateMinus3Days, endDate=depositDatePlus3Days)`

Branching:
  0 matches → Proceed.
  
  1 match → If same bank account, date within 3 days, amount within 10%, and matching memo text or bankTransactionId, Interactive: ask:
  > I found a possible duplicate deposit: dated [date], amount $[amount], memo “[memo]”. Please confirm this is not already recorded before I proceed.
  
  Batch/async: `flagForReview(entityType="Deposit", entityId=depositId, aiReasoning="Potential duplicate deposit found: same bank account, date within 3 days, amount within 10%, and matching memo/bankTransactionId.")`
  
  >1 matches → Interactive: ask:
  > I found multiple possible duplicate deposits near this date and amount. Should I create a new deposit or use one of these existing deposits?
  
  Batch/async: `flagForReview(entityType="Account", entityId=bankAccountId, aiReasoning="Multiple possible duplicate deposits found near requested bank deposit date and amount.")`

### Step 5: Record and verify the deposit
Use LinkedTxn lines for existing ReceivePayments or SalesReceipts from Undeposited Funds. Direct deposit lines are only for non-customer accounts such as Interest Income, Owner’s Equity, Other Income, Insurance Proceeds, Refunds Received, or Miscellaneous Income. `(DEPOSIT TYPE GUARD blocks qbDeposit here if a deposit is used to close an invoice, record a single customer sale, or use suspicious customer-facing income accounts)`

Interactive mode: immediately before the write, emit:

Pre-write evidence:
- Entity: [bankAccountName + bankAccountId]
- Open-doc: [no outstanding invoices / linked ReceivePayment or SalesReceipt IDs]
- Account basis: [Undeposited Funds linked items match bank statement deposit]
- Mode: interactive

`qbDeposit(depositAccountId=bankAccountId, txnDate=depositDate, lines=linkedUndepositedFundsLines)`

Branching:
  0 matches → If linked items are missing, return to Phase 4 Step 2; if bank account is wrong, return to Phase 4 Step 1.
  
  1 match → Store `depositId`; verify `depositTotal` matches the bank statement line. For reconciliation workflow, delegate bank matching to the bank-reconciliation skill.
  
  >1 matches → Not applicable for a create response; if ambiguous, `flagForReview(entityType="Deposit", entityName=bankAccountName, aiReasoning="QuickBooks returned ambiguous deposit response; verify deposit before reconciliation.")`  
  `(WRITE SAFETY GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, DUPLICATE RESULT GUARD, DEPOSIT TYPE GUARD, SOURCE-CATEGORY COLLISION GUARD, MATERIALITY GUARD, CONSISTENCY RULE GUARD, and CURRENCY GUARD validate qbDeposit here)`

## Phase 5 — Apply customer credits and issue refunds

### Step 1: Create a customer credit memo
Use credit memos for returns, billing adjustments, and promotional discounts.

`qbMasterData(entityType="Customer", name=customerName)`

`qbMasterData(entityType="Item", name=itemOrServiceName)`

`qbFetchTransactions(transactionType="CreditMemo", entityId=customerId, startDate=txnDateMinus3Days, endDate=txnDatePlus3Days)`

Interactive mode: immediately before the write, emit:

Pre-write evidence:
- Entity: [customerName + customerId]
- Open-doc: [customer credit memo requested]
- Account basis: [item/account resolved from master data]
- Mode: interactive

`qbCredit(creditType="customer", customerId=customerId, txnDate=txnDate, lines=creditLines)`

Branching:
  0 matches → If customer or item is missing, ask:
  > I’m missing the customer or item/service for this credit memo. Please provide the exact QuickBooks name.
  
  Batch/async: `flagForReview(entityType="Customer", entityName=customerName, aiReasoning="Missing customer or item for customer credit memo; cannot create credit safely.")`
  
  1 match → Store `creditMemoId` and `creditAmount`.
  
  >1 matches → If duplicate credit risk exists, Interactive: ask:
  > I found possible duplicate credit memos for this customer near the same date and amount. Should I still create a new credit?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Possible duplicate or ambiguous customer credit memo found near requested date and amount.")`  
  `(WRITE SAFETY GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, DUPLICATE RESULT GUARD, and CURRENCY GUARD validate qbCredit here)`

### Step 2: Apply a credit memo to a customer payment
Use Phase 2 Step 4 and include `creditMemos` in the ReceivePayment allocation.

`qbReceivePayment(customerId=customerId, totalAmount=paymentAmount, paymentDate=paymentDate, invoices=invoices, creditMemos=creditMemos)`

Branching:
  0 matches → No open invoice to apply credit against; ask/flag using Phase 2 Step 2.
  
  1 match → Store `paymentId`, `creditApplied`, and remaining `unappliedAmount`.
  
  >1 matches → If multiple invoices or credits could apply, ask/flag using Phase 2 Step 4.

### Step 3: Issue a cash refund to the customer
`qbMasterData(entityType="Customer", name=customerName)`

`qbMasterData(entityType="Item", name=itemOrServiceName)`

`qbMasterData(entityType="Account", name=bankAccountName)`

`qbFetchTransactions(transactionType="RefundReceipt", entityId=customerId, startDate=startDate, endDate=endDate)`

Interactive mode: immediately before the write, emit:

Pre-write evidence:
- Entity: [customerName + customerId]
- Open-doc: [refund requested / customer credit reviewed]
- Account basis: [refund item and bank account resolved from master data]
- Mode: interactive

`qbRefundReceipt(customerId=customerId, depositAccountId=bankAccountId, txnDate=txnDate, lines=refundLines)`

Branching:
  0 matches → If no duplicate RefundReceipt exists and IDs are resolved, proceed; if IDs are missing, return to lookup.
  
  1 match → If same customer, date, amount, and memo indicate a duplicate, Interactive: ask:
  > I found a possible duplicate refund receipt for this customer: dated [date], amount $[amount]. Please confirm this is not already refunded.
  
  Batch/async: `flagForReview(entityType="RefundReceipt", entityId=refundReceiptId, aiReasoning="Possible duplicate customer refund receipt found for same customer/date/amount.")`
  
  >1 matches → Interactive: ask:
  > I found multiple possible refund receipts for this customer. Which one relates to this refund, or should I create a new refund?
  
  Batch/async: `flagForReview(entityType="Customer", entityId=customerId, aiReasoning="Multiple possible refund receipts found; cannot safely issue another refund.")`  
  `(WRITE SAFETY GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, DUPLICATE RESULT GUARD, and CURRENCY GUARD validate qbRefundReceipt here)`

## Phase 6 — Review AR aging and collections

### Step 1: Pull AR aging reports
`qbReports(reportType="AgedReceivables")`

`qbReports(reportType="AgedReceivablesDetail")`

Branching:
  0 matches → Report no AR balances found; if the user expected balances, ask:
  > QuickBooks is showing no aged receivables. Do you want me to check recent invoices or customer payments for posting issues?
  
  Batch/async: `flagForReview(entityType="Report", entityName="AgedReceivables", aiReasoning="AgedReceivables report returned no balances but user requested AR aging review; verify posting or report filters.")`
  
  1 match → Store summary and detail report data; proceed.
  
  >1 matches → Use the summary for totals and detail for invoice-level follow-up; proceed.

### Step 2: Calculate DSO and aging buckets
Calculate Days Sales Outstanding as `DSO = AR / (Annual Revenue / 365)`. Present balances in aging buckets: Current, 1-30, 31-60, 61-90, and 90+.

Branching:
  0 matches → If annual revenue is unavailable, present aging without DSO and ask:
  > I can show the aging buckets, but I need annual revenue to calculate DSO. What annual revenue figure should I use?
  
  Batch/async: `flagForReview(entityType="Report", entityName="AgedReceivables", aiReasoning="Annual revenue unavailable for DSO calculation; aging buckets can be shown but DSO cannot be computed.")`
  
  1 match → Store `DSO`, bucket totals, and customer balances.
  
  >1 matches → If multiple revenue figures exist, use the year-to-date or annual revenue basis specified by the user; otherwise ask/flag for the correct annual revenue basis.

### Step 3: Flag AR concerns
Flag:
- Invoices over 90 days → discuss bad debt write-off with CPA.
- Rising DSO trend → collection process needs attention.
- Single customer > 30% of AR → concentration risk.

`flagForReview(entityType="Invoice", entityId=invoiceId, aiReasoning="Invoice #[invoiceNumber] for customer [customerName] is over 90 days outstanding; discuss bad debt write-off with CPA before any month-end entry.")`

Branching:
  0 matches → No AR concerns; provide a clean aging summary.
  
  1 match → Present the concern and recommended follow-up.
  
  >1 matches → Prioritize by age, amount, and customer concentration.

Month-end: propose bad debt or allowance entries only; do not post without approval. Delegate any approved journal-entry posting to the month-end or journal-entry skill.

### Step 4: Recommend collections follow-up
Produce a prioritized list of customers to follow up with, including customer name, invoice number, due date, balance, aging bucket, and suggested action.

Branching:
  0 matches → No follow-up needed.
  
  1 match → Provide one recommended follow-up action.
  
  >1 matches → Sort by 90+ first, then largest balance, then oldest due date.

## Troubleshooting

INCOME TYPE GUARD blocks qbReceivePayment — no outstanding invoice exists, or the payment should be a SalesReceipt/Deposit instead → recover at Phase 2 Step 2.

INCOME TYPE GUARD blocks qbSalesReceipt — customer has outstanding invoices and SalesReceipt would double-count income while leaving AR unpaid → recover at Phase 3 Step 2 and route to Phase 2.

INCOME TYPE GUARD blocks qbInvoice — context says customer already paid, so Invoice is wrong because Invoice means owed later → recover at Phase 1 Step 5 and route to Phase 3.

DEPOSIT TYPE GUARD blocks qbDeposit — deposit is trying to close an invoice, record a single customer sale, or use a customer-facing income account directly → recover at Phase 4 Step 5; use Phase 2 for invoice payments or Phase 3 for immediate customer sales.

MULTI-VENDOR AMBIGUITY GUARD blocks AR write — qbMasterData returned multiple customer matches and the agent picked one → recover at Phase 1 Step 1, Phase 2 Step 1, or Phase 3 Step 1 by asking interactively or calling flagForReview in batch.

CONSISTENCY RULE GUARD blocks qbInvoice/qbSalesReceipt — a history-inferred item, term, or account was used without evaluating the required history criteria → recover at Phase 1 Step 3.

DUPLICATE RESULT GUARD blocks AR write — qbFetchTransactions found same customer, date within 3 days, amount within 10%, and matching memo/bankTransactionId → recover at Phase 1 Step 4, Phase 2 Step 3, Phase 3 Step 4, or Phase 4 Step 4.

VENDOR/ACCOUNT RESOLUTION GUARD blocks write — customerId, itemId, or accountId was not returned by the most recent qbMasterData lookup → recover at the relevant lookup step before retrying the write.

SOURCE-CATEGORY COLLISION GUARD blocks qbSalesReceipt/qbDeposit/qbInvoice — source account equals a line category account and would create a zero-net entry → recover at Phase 3 Step 5 or Phase 4 Step 5 by selecting the correct source and line accounts.

CURRENCY GUARD blocks AR write — transaction currency differs from the source account currency and no exchangeRate or currency conversion information was provided → recover at the write step by adding exchangeRate or flagging for review.

FLAG FOR REVIEW QUALITY GUARD blocks flagForReview — aiReasoning is missing, too short, or generic → rewrite aiReasoning with the specific customer, transaction, amount, date, and why CPA/user input is required.

qbReports returns no AgedReceivables — report filters or posting may not include expected open invoices → recover at Phase 6 Step 1 and offer to check recent invoices and payments.

qbAttachFile fails — file source, fileId, or entityId is invalid → recover at Phase 1 Step 6 or Phase 2 Step 6 by reselecting the support file and verifying the created Invoice or Payment ID.