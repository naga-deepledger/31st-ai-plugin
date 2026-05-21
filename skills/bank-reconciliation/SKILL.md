---
name: bank-reconciliation
description: Reconcile bank and credit card accounts by running health checks, clearing bank-feed and review queues, resolving duplicates/uncategorized items, then completing QuickBooks Online reconciliation via browser. Use when the user mentions reconciliation, bank matching, uncleared items, account health, bank-feed cleanup, or closing the books.
---

| Situation | Phase |
|---|---|
| Check account health, duplicates, uncategorized, outliers, past-due items | Phase 1 |
| Process unrecorded bank transactions before reconciling | Phase 2 |
| Bank-feed-processing flags items for CPA/user review | Phase 2 Step 3 |
| Complete the reconciliation in QuickBooks Online | Phase 3 |
| Month-end close with unresolved uncertainty | Phase 3 Step 1 |

## Phase 1 — Health Check

### Step 1: Identify reconcilable accounts

`qbMasterData(entityTypes=["account"])`

Branching:
- 0 matches → ask the user:
  > I could not find any Bank or Credit Card accounts in QuickBooks. Which account should I reconcile?
- 1 match → proceed with that Bank or Credit Card account; store `accountId`, `accountName`, and `accountType`.
- >1 matches → filter to Bank and Credit Card accounts; if the user did not specify one, ask:
  > Which account should I reconcile? I found these Bank/Credit Card accounts: [account names + IDs].

### Step 2: Run account health check

`qbAccountHealth(accountId=accountId, startDate=startDate, endDate=endDate)`

Evaluate returned flags:
- Duplicates — same amount + date + vendor; high severity if amount > $1,000.
- Uncategorized — booked to “Ask My Accountant” or “Uncategorized”; high severity if amount > $500.
- Outliers — amount exceeds 2 standard deviations from mean.
- Past-due — transactions with `dueDate` older than 7 days; this checks the `dueDate` field, not QuickBooks cleared status.
- Score — 0–100, with penalties: high = -5, medium = -3, low = -1.
- Score thresholds: 95–100 Audit-ready → proceed to reconcile; 90–94 Good → minor cleanup first; 80–89 Needs attention → resolve flags before reconciling; <80 Critical → fix issues before touching the reconciliation screen.

Branching:
- 0 flags and score 95–100 → proceed to Phase 2 if bank feed has not been checked for the period, otherwise Phase 3.
- 1 flag → route to Phase 1 Step 3 for duplicate flags or Phase 1 Step 4 for uncategorized flags; for outliers or past-due items, investigate with `qbFetchTransactions`.
- >1 flags → resolve duplicates first, then uncategorized items, then outliers/past-due items; do not open the reconciliation screen until the score is at least 95 or the user explicitly asks to inspect only.

### Step 3: Resolve duplicate flags

`qbFetchTransactions(transactionId=firstTransactionId)`

`qbFetchTransactions(transactionId=secondTransactionId)`

Verify the duplicate is true: same amount, vendor, and purpose. Two similar charges from one vendor can both be legitimate.

Supported voidable transaction types: `BillPayment`, `Invoice`, `Payment`, `SalesReceipt`, `CreditMemo`, `Purchase`, `RefundReceipt`, `Transfer`.

Unsupported via tools: `Bill`, `JournalEntry`, `Deposit`, `Expense`, `VendorCredit`.

Branching:
- 0 true duplicates → leave both transactions in place; re-run `qbAccountHealth(accountId=accountId, startDate=startDate, endDate=endDate)` if the flag appears stale.
- 1 true duplicate with supported voidable type → in interactive mode, ask before voiding:
  > I found a likely duplicate: [date, amount, vendor, memo/purpose]. Voiding preserves the audit trail better than deleting. Should I void transaction [transactionId]?
  
  Before calling the void tool, emit:
  
  Pre-write evidence:
  - Entity: [vendor/customer name + ID]
  - Open-doc: N/A — duplicate void target
  - Account basis: duplicate confirmed by fetched transaction pair
  - Mode: interactive
  
  Then call `qbVoidTransaction(transactionId=transactionId, transactionType=transactionType)` (VOID TRANSACTION GUARD blocks qbVoidTransaction unless the target was fetched and verified).
- 1 true duplicate with unsupported type → call `flagForReview(transactionId=transactionId, aiReasoning="Duplicate appears true, but transaction type [type] cannot be voided via tools; CPA must handle in QuickBooks. Matched pair: [dates, amounts, vendor, purpose].")`.
- >1 duplicate candidates → do not pick automatically; in interactive mode ask:
  > I found multiple possible duplicate pairs for this account. Which transaction should be voided, if any? [candidate list with date, amount, vendor, memo, transactionId]
  
  In batch/async mode, call `flagForReview(accountId=accountId, aiReasoning="Multiple duplicate candidates found during reconciliation health check; agent must not choose which transaction to void. Candidates: [candidate list].")`.

After any void or review flag, re-run:

`qbAccountHealth(accountId=accountId, startDate=startDate, endDate=endDate)`

Branching:
- 0 duplicate flags remain → proceed to Phase 1 Step 4 or Phase 2.
- 1 duplicate flag remains → repeat Phase 1 Step 3.
- >1 duplicate flags remain → continue duplicate cleanup before uncategorized cleanup or reconciliation.

### Step 4: Resolve uncategorized flags

`qbFetchTransactions(accountId=accountId, startDate=startDate, endDate=endDate, accountNames=["Ask My Accountant","Uncategorized"])`

For each uncategorized transaction, resolve the counterparty:

`qbMasterData(detailedInfo="vendor", filter=counterpartyName)`

Branching:
- 0 matches → call `flagForReview(transactionId=transactionId, aiReasoning="Uncategorized reconciliation item has no matching vendor in QuickBooks. Vendor/counterparty: [counterpartyName]. Bank memo: [memo]. Amount/date: [amount] on [date]. Do not infer account for a new vendor.")` (FLAG FOR REVIEW QUALITY GUARD enforces specific `aiReasoning`).
- 1 match → store `vendorId`, `vendorName`; continue.
- >1 matches → call `flagForReview(transactionId=transactionId, aiReasoning="Multiple vendors matched uncategorized reconciliation item; agent must not pick one. Counterparty: [counterpartyName]. Candidates: [candidate list]. Bank memo: [memo]. Amount/date: [amount] on [date].")` (MULTI-VENDOR AMBIGUITY GUARD blocks writes when multiple candidates exist).

Before account inference, run the open-document check as a separate call with no date window:

`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

Branching:
- 0 matches → fetch bounded vendor history:
  
  `qbFetchTransactions(entityId=vendorId, entityType="Vendor", lookbackDays=365)`
  
  Apply the CONSISTENCY RULE before any history-based re-categorization: use the dominant account only if all five criteria pass: (a) ≥ 3 prior transactions in the last 365 days, (b) dominant account ≥ 70%, (c) no second account ≥ 20%, (d) current amount within 5× the median of dominant-account transactions, and (e) most recent dominant-account transaction < 180 days old. If any criterion fails, call `flagForReview(transactionId=transactionId, suggestedCategory=suggestedCategory, aiReasoning="Uncategorized item cannot be safely categorized from history. Vendor: [vendorName]. History split: [account counts/percentages]. Bank memo: [memo]. Amount/date: [amount] on [date].")` (CONSISTENCY RULE GUARD blocks history-inferred writes unless these five criteria were evaluated and passed).
- 1 match → the outstanding Bill encodes the correct account; do not infer from history. If the uncategorized item is actually a payment of that Bill, route the item to the vendor payable/payment workflow instead of re-categorizing it.
- >1 matches → do not infer from history; in interactive mode ask:
  > This vendor has multiple outstanding bills. Which bill, if any, does this bank transaction pay? [bill list with amount, date, due date, billId]
  
  In batch/async mode, call `flagForReview(transactionId=transactionId, aiReasoning="Uncategorized bank transaction may relate to multiple outstanding bills; cannot choose bill automatically. Vendor: [vendorName]. Bills: [bill list]. Bank memo: [memo]. Amount/date: [amount] on [date].")`.

If re-categorizing or otherwise writing in interactive mode, emit before the write:

Pre-write evidence:
- Entity: [vendorName + vendorId]
- Open-doc: [no outstanding bills / billId X applied]
- Account basis: [N txns, XX% dominant / open-doc match]
- Mode: interactive

Then use the appropriate transaction-specific update path if available; if no safe update path is available, call `flagForReview(transactionId=transactionId, aiReasoning="Uncategorized item has a likely category but no safe supported update path in this workflow. Vendor: [vendorName]. Suggested category: [accountName + accountId]. Bank memo: [memo]. Amount/date: [amount] on [date].")`.

After cleanup:

`qbAccountHealth(accountId=accountId, startDate=startDate, endDate=endDate)`

Branching:
- 0 uncategorized flags remain → proceed to Phase 2.
- 1 uncategorized flag remains → repeat Phase 1 Step 4.
- >1 uncategorized flags remain → continue resolving or flagging; do not open the reconciliation screen for completion until the queue is clear or explicitly approved for inspection only.

## Phase 2 — Delegate Bank Feed Processing

### Step 1: Fetch unrecorded bank-feed items

`bankFeed(action="fetch", accountId=accountId, sinceDate=startDate)`

`bankFeed` returns transactions that exist at the bank but are not yet recorded in QuickBooks. Skip items with `alreadyFlagged=true`.

Branching:
- 0 matches → no unrecorded bank-feed items remain for the period; proceed to Phase 2 Step 3.
- 1 match → delegate that item to the `bank-feed-processing` skill; do not record it directly in this skill.
- >1 matches → delegate the full period/account batch to the `bank-feed-processing` skill; do not record them directly in this skill.

### Step 2: Hand off recording decisions to bank-feed-processing

Delegate to `bank-feed-processing` with `accountId=accountId`, `startDate=startDate`, `endDate=endDate`, and the fetched bank-feed item list.

The delegated skill owns entity resolution, duplicate checks, open-document checks, account inference, and write-tool selection. This reconciliation skill must not re-implement those recording flows.

Bank line type routing preserved for the delegated skill:
- Vendor payment / expense → `qbExpense`
- Customer deposit / payment received → `qbDeposit`
- Transfer between accounts → `qbTransfer`
- Payroll or complex entry → `qbJournalEntry`
- Unclear / needs CPA → `flagForReview`

After every bank-feed transaction is either recorded or flagged, the delegated workflow must mark it processed to prevent re-processing:

`fetchWorkQueue(source="markRecorded", bankTransactionId=bankTransactionId, status="recordedOrFlagged")`

Branching:
- 0 delegated items recorded or flagged → stop and report that bank-feed-processing did not complete; do not open the reconciliation screen.
- 1 delegated item recorded or flagged → proceed to Phase 2 Step 3.
- >1 delegated items recorded or flagged → proceed to Phase 2 Step 3 after all items are marked recorded/flagged.

Hooks enforced during delegated writes: WRITE SAFETY GUARD, DUPLICATE RESULT GUARD, EXPENSE TYPE GUARD, DEPOSIT TYPE GUARD, INCOME TYPE GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, SOURCE-CATEGORY COLLISION GUARD, MATERIALITY GUARD, CONSISTENCY RULE GUARD, CURRENCY GUARD, MULTI-VENDOR AMBIGUITY GUARD.

### Step 3: Handle the bank-feed review queue

`fetchWorkQueue(source="bank-feed-processing", accountId=accountId, startDate=startDate, endDate=endDate, status="flagged")`

Branching:
- 0 matches → the bank-feed review queue is clear for the reconciliation period; proceed to Phase 3 if health score allows.
- 1 match → do not finalize reconciliation. In interactive mode ask:
  > One bank-feed item is flagged for review for this reconciliation period: [summary]. Should I wait while you/your CPA resolve it, or open the reconciliation screen for inspection only without finishing?
  
  In batch/async mode, call `flagForReview(accountId=accountId, aiReasoning="Reconciliation blocked because one bank-feed item remains flagged for the period. Item: [summary]. Do not finalize until the review queue is cleared.")`.
- >1 matches → do not finalize reconciliation. In interactive mode ask:
  > [N] bank-feed items are flagged for review for this reconciliation period. I should not complete reconciliation until they are resolved. Do you want me to stop here, or open the reconciliation screen for inspection only?
  
  In batch/async mode, call `flagForReview(accountId=accountId, aiReasoning="Reconciliation blocked because multiple bank-feed items remain flagged for the period. Count: [N]. Items: [summaries]. Do not finalize until the review queue is cleared.")`.

If mode is month-end, propose needed entries or review actions only; do not post or finalize without explicit approval.

## Phase 3 — Reconcile in QB UI via Computer Use

### Step 1: Decide whether browser automation may start

Use browser automation through the `computer-use` skill only after Phase 1 health check and Phase 2 bank-feed processing are complete.

Branching:
- 0 unresolved health flags and 0 flagged bank-feed queue items → activate `computer-use` for full reconciliation.
- 1 unresolved health flag or 1 flagged bank-feed item → interactive mode may open the reconciliation screen for inspection only, but must not finish; batch/async mode must call `flagForReview(accountId=accountId, aiReasoning="Reconciliation cannot be finalized because one unresolved item remains before opening/completing the QB reconciliation UI. Item: [summary].")`.
- >1 unresolved health flags or >1 flagged bank-feed items → wait for queue cleanup; do not use browser automation for completion. In interactive mode ask:
  > There are [N] unresolved items for this reconciliation period. I should not complete the reconciliation yet. Should I stop here, or open QuickBooks for inspection only?

### Step 2: Open QuickBooks Online reconciliation screen

Activate `computer-use` skill: open QuickBooks Online → Bookkeeping → Reconcile.

Select the Bank or Credit Card account identified in Phase 1.

Branching:
- 0 matching accounts visible in the UI → stop and ask:
  > I cannot find [accountName] in the QuickBooks reconciliation screen. Should I stop so you can verify account permissions or the account name?
- 1 matching account visible → select it and continue.
- >1 matching accounts visible → ask:
  > I see multiple matching accounts in QuickBooks: [account list]. Which one should I reconcile?

### Step 3: Enter statement details

In the QuickBooks reconciliation UI, enter:
- Beginning balance — should match QuickBooks’ calculated opening balance.
- Ending balance — from the bank statement.
- Statement end date.

Branching:
- 0 beginning-balance difference → continue.
- 1 beginning-balance mismatch → stop and ask:
  > QuickBooks’ beginning balance does not match the expected opening balance. I should not continue until this is investigated. Should I stop here and report the mismatch?
- >1 statement values missing or inconsistent → stop and ask:
  > I’m missing or seeing inconsistent statement details: [missing/inconsistent fields]. Please provide the exact ending balance and statement end date before I continue.

### Step 4: Mark statement transactions cleared

Using the browser, check off each transaction that appears on the bank statement. The running difference should approach $0.00.

If a suspicious item needs investigation while the reconciliation screen stays open, use:

`qbFetchTransactions(accountId=accountId, startDate=startDate, endDate=endDate)`

Branching:
- 0 suspicious or uncleared statement items remain → proceed to Phase 3 Step 5.
- 1 suspicious or unmatched item remains → stop marking and ask:
  > I found one item that does not clearly match the statement: [date, amount, payee, memo]. Should I investigate this transaction before continuing?
- >1 suspicious or unmatched items remain → stop marking and ask:
  > I found [N] items that do not clearly match the statement. I should investigate these before reconciliation can be completed: [summary list]. Should I pause here?

### Step 5: Evaluate reconciliation difference

Force-adjust threshold: $0.00. Any non-zero difference must halt reconciliation; do not force-finish or create an adjustment for a non-zero difference because it creates a discrepancy that compounds every month.

Branching:
- 0.00 difference → report the difference and wait for explicit user confirmation before finishing.
  
  Ask:
  > The reconciliation difference is $0.00. Do you want me to click “Finish now” / “Complete reconciliation”?
- 1 non-zero difference → halt and ask:
  > The reconciliation difference is $[difference]. My threshold for force-adjusting is $0.00, so I will not finalize. Should I keep the screen open while we investigate, or stop and send this to the CPA?
- >1 categories of differences/uncleared groups → halt and summarize:
  > The reconciliation is not ready to finish. Difference: $[difference]. Uncleared groups: [summary]. I will not click “Finish now.” Should I investigate these groups or stop here?

### Step 6: Finalize only after explicit confirmation

Never click “Finish now” or “Complete reconciliation” without explicit user instruction, even if the difference is $0.00.

Before finalizing in interactive mode, emit:

Pre-write evidence:
- Entity: [accountName + accountId]
- Open-doc: bank-feed/review queue clear for reconciliation period
- Account basis: health score [score], reconciliation difference $0.00
- Mode: interactive

Then, only after the user explicitly confirms, use browser automation to click “Finish now” / “Complete reconciliation.”

Branching:
- 0 explicit confirmation → stop and leave the reconciliation unfinalized; report the current difference and uncleared items.
- 1 explicit confirmation and difference is $0.00 → finalize, then download/save the QuickBooks reconciliation report PDF for audit records.
- >1 conflicting user instructions or any non-zero difference → do not finalize; ask:
  > I need a clear instruction before finalizing. The current difference is $[difference]. Should I stop, investigate, or — only if the difference is $0.00 — finish the reconciliation?

## Troubleshooting

VOID TRANSACTION GUARD blocks qbVoidTransaction — transaction was not fetched or the target ID was not in the most recent `qbFetchTransactions` results → re-fetch both duplicate candidates and verify the void target in Phase 1 Step 3.

CONSISTENCY RULE GUARD blocks a history-inferred categorization/write — the five consistency criteria were not explicitly evaluated or did not pass → fetch bounded vendor history and evaluate the rule in Phase 1 Step 4; if any criterion fails, call `flagForReview`.

MULTI-VENDOR AMBIGUITY GUARD blocks a write — `qbMasterData` returned multiple vendor/customer candidates → do not pick one; call `flagForReview` with the full candidate list in Phase 1 Step 4 or let `bank-feed-processing` handle the item in Phase 2.

FLAG FOR REVIEW QUALITY GUARD blocks flagForReview — `aiReasoning` is missing, too short, or generic → retry `flagForReview` with vendor/customer name, amount/date, memo verbatim, candidate list or history split, and the exact reason CPA input is needed.

WRITE SAFETY GUARD blocks a delegated write — `qbMasterData` or `qbFetchTransactions` was not called before the write → return the item to `bank-feed-processing` Phase 2 Step 2 so it can perform master-data lookup and duplicate check before recording.

qbAccountHealth returns a low score after cleanup — unresolved duplicate, uncategorized, outlier, or past-due flags remain → repeat Phase 1 Step 2 and route each remaining flag through Phase 1 Step 3 or Step 4 before attempting Phase 3 completion.

QuickBooks beginning balance mismatch — prior reconciled transactions may have been changed, deleted, voided, or uncleared → stop in Phase 3 Step 3 and report the mismatch; do not continue to finalization.

Non-zero reconciliation difference — one or more statement transactions are missing, duplicated, uncleared, or incorrectly dated/amounted → halt in Phase 3 Step 5; use `qbFetchTransactions(accountId=accountId, startDate=startDate, endDate=endDate)` to investigate while the browser remains open.