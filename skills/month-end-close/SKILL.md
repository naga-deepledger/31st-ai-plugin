---
name: month-end-close
description: Runs the month-end close workflow for QuickBooks Online: pre-close health checks, AP/AR cleanup, proposed adjusting entries, close proposal, approved posting, and portal completion. Activate when the user mentions closing the books, month-end close, close period, close status, adjusting entries, accruals, deferrals, depreciation, or preparing financial statements for review.
---

| Situation | Phase |
|---|---|
| User asks “are we ready to close?” or “run close checks” | Phase 1 |
| Open AP, AR, bank-feed, uncategorized, or reconciliation issues exist | Phase 2 |
| Accruals, deferrals, depreciation, reversals, or adjusting entries are needed | Phase 3 |
| User wants the close package/checklist before posting | Phase 4 |
| CPA/user approved the close proposal and entries | Phase 5 |

## Phase 1 — Run pre-close health check

### Step 1: Start the close run
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: not evaluated yet
- Account basis: close-run initialization only
- Mode: interactive

`closeRun(operation="start", period="YYYY-MM", periodLabel="Month Year")`

Branching:
  0 matches → Ask for the close period:
> Which month should I close? Please provide the period as YYYY-MM.
  1 match → Store `closeRunId`, `period`, `periodLabel`, and status `in_progress`; proceed.
  >1 matches → Use the existing open close run for `period="YYYY-MM"` if one exists; otherwise ask:
> I found more than one close run for this period. Which close run should I continue?

### Step 2: Read bank and credit card accounts
`qbMasterData(entityTypes=["account"])`

Branching:
  0 matches → `flagForReview(aiReasoning="No bank or credit card accounts were returned by qbMasterData for the close health check; cannot verify reconciliation status for YYYY-MM.")`
  1 match → Store the account if type is Bank or Credit Card; proceed to health check.
  >1 matches → Store all Bank and Credit Card accounts for the closing month health check.

### Step 3: Check reconciliation health for each account
Read: bank and credit card account reconciliation status, uncategorized items, duplicates, outliers, and unresolved bank-feed activity for the closing month.

`qbAccountHealth(accountId=accountId, period="YYYY-MM")`

Branching:
  0 matches → `flagForReview(aiReasoning="qbAccountHealth returned no health result for account accountId during YYYY-MM close; reconciliation readiness cannot be verified.")`
  1 match → If health score is `>= 90`, mark account ready; if below `90`, route the account to Phase 2 Step 1 for duplicates, uncategorized items, and outlier cleanup.
  >1 matches → Store the latest health result for the account and route conflicting health results to review with `flagForReview(aiReasoning="Multiple qbAccountHealth results were returned for account accountId in YYYY-MM; cannot determine the reconciliation health score reliably.")`

### Step 4: Read AP, AR, and open-item reports
Read: aged payables, aged receivables, outstanding bills, outstanding invoices, recurring transactions expected in the period, payroll entries, sales tax liability, undeposited funds, Uncategorized/Ask My Accountant, suspense/clearing accounts, and edits to closed periods.

`qbReports(reportType="AgedPayables", period="YYYY-MM")`

`qbReports(reportType="AgedReceivables", period="YYYY-MM")`

`qbReports(reportType="BalanceSheet", period="YYYY-MM")`

`qbReports(reportType="TransactionList", period="YYYY-MM", accountNames=["Undeposited Funds", "Uncategorized", "Ask My Accountant", "Suspense", "Clearing", "Payroll", "Sales Tax"])`

`qbReports(reportType="TransactionList", startDate="prior-close-date", endDate="YYYY-MM-DD", filter="changes_since_last_close")`

Branching:
  0 matches → `flagForReview(aiReasoning="Required AP, AR, balance sheet, or transaction-list reports did not return data for YYYY-MM; close readiness cannot be verified.")`
  1 match → Store the report result and proceed if no open items require cleanup.
  >1 matches → Store all report results; route any past-due AR over 90 days as potential bad debt write-off, past-due AP as late payment risk, and any non-zero Uncategorized, Ask My Accountant, Suspense, or Clearing activity to Phase 2.

### Step 5: Update reconciliation and open-item status
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: AP/AR/open-item reports read
- Account basis: account health results stored for all Bank and Credit Card accounts
- Mode: interactive

`closeRun(operation="updateStep", step={id: "reconciliation", status: "pass|action_needed", evidence: accountHealthSummary})`

`closeRun(operation="updateChecks", checks=[{id: 7, status: "pass|action_needed", evidence: bankReconciliationEvidence}])`

Branching:
  0 matches → If the portal update fails, retry once; if it fails again, `flagForReview(aiReasoning="closeRun update failed for reconciliation status in YYYY-MM after qbAccountHealth checks completed.")`
  1 match → Proceed to Phase 2 if any account score is below `90` or open items exist; otherwise proceed to Phase 3.
  >1 matches → Use the close run identified in Phase 1 Step 1 and do not update any other close run.

## Phase 2 — Review and clear open items

### Step 1: Clear bank-feed and reconciliation issues
Delegate bank-feed cleanup to the bank-feed skill; do not duplicate bank-feed categorization logic here.

Read: account-health findings from Phase 1, unmatched bank-feed items, duplicates, uncategorized transactions, and outliers.

`qbAccountHealth(accountId=accountId, period="YYYY-MM")`

Branching:
  0 matches → `flagForReview(aiReasoning="Cannot clear bank-feed or reconciliation issues for account accountId because qbAccountHealth returned no findings for YYYY-MM.")`
  1 match → If duplicates, uncategorized items, or outliers exist, delegate to bank-feed skill; if resolved, rerun Phase 1 Step 3.
  >1 matches → Delegate all conflicting account-health findings to bank-feed skill and require rerun of Phase 1 Step 3 before proposal.

### Step 2: Review and clear AP issues
Delegate bill entry, bill payment, vendor credits, and AP cleanup to the AP skill; do not duplicate AP posting logic here.

Read: Aged Payables and open bills.

`qbReports(reportType="AgedPayables", period="YYYY-MM")`

`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, endDate="YYYY-MM-DD")`

Branching:
  0 matches → Mark AP as clear if Aged Payables also has no open obligations; otherwise `flagForReview(aiReasoning="Aged Payables shows obligations but qbFetchTransactions found no outstanding Bills for YYYY-MM; AP report and transaction detail do not agree.")`
  1 match → Verify the bill is in the correct period; if past due, route to AP skill as late payment risk.
  >1 matches → Verify all open bills match obligations and are in the correct period; route incorrect-period bills, missing bills, vendor credits, and past-due AP to AP skill.

### Step 3: Review and clear AR issues
Delegate invoice entry, receive payment, credits, bad debt review, and AR cleanup to the AR skill; do not duplicate AR posting logic here.

Read: Aged Receivables and open invoices.

`qbReports(reportType="AgedReceivables", period="YYYY-MM")`

`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, endDate="YYYY-MM-DD")`

Branching:
  0 matches → Mark AR as clear if Aged Receivables also has no open invoices; otherwise `flagForReview(aiReasoning="Aged Receivables shows open balances but qbFetchTransactions found no outstanding Invoices for YYYY-MM; AR report and transaction detail do not agree.")`
  1 match → Verify the invoice is in the correct period; if past due over `90` days, route to AR skill as potential bad debt write-off.
  >1 matches → Verify all open invoices match expectations and are in the correct period; route unapplied payments, incorrect-period invoices, credits, and AR over `90` days past due to AR skill.

### Step 4: Update AP and AR close status
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: outstanding Bills and Invoices reviewed
- Account basis: Aged Payables and Aged Receivables report match reviewed transaction detail
- Mode: interactive

`closeRun(operation="updateStep", step={id: "ap_ar", status: "pass|action_needed", evidence: apArEvidence})`

`closeRun(operation="updateChecks", checks=[{id: 5, status: "pass|action_needed", evidence: apEvidence}, {id: 6, status: "pass|action_needed", evidence: arEvidence}])`

Branching:
  0 matches → If the portal update fails, retry once; if it fails again, `flagForReview(aiReasoning="closeRun update failed for AP/AR status in YYYY-MM after AP and AR review completed.")`
  1 match → Proceed to Phase 3 when AP and AR cleanup is complete or documented as action_needed.
  >1 matches → Use the close run identified in Phase 1 Step 1 and do not update any other close run.

## Phase 3 — Prepare adjusting entries

### Step 1: Find prior-month accruals and reversals
Delegate journal-entry drafting and reversal logic to the journal-entries skill; do not post entries in this phase.

Read: prior-month journal entries for accruals, deferrals, prepaid amortization, depreciation, and entries marked for reversal.

`qbFetchTransactions(transactionType="JournalEntry", startDate="prior-month-start", endDate="prior-month-end", memoContains="accrual|deferral|prepaid|depreciation|reversal")`

Branching:
  0 matches → Store “no prior-month accrual reversals found” and proceed to new adjusting-entry review.
  1 match → Store the prior journal entry and determine whether failing to reverse it would double-count expenses or revenue.
  >1 matches → Store all prior-month journal entries; propose reversal entries for any accruals or deferrals that should reverse before new entries are proposed.

### Step 2: Propose accruals and deferrals
Month-end mode: propose entries only; do not post without approval.

Read: unpaid vendor obligations, invoices, prepaid asset balances, deferred revenue balances, prior-month adjusting entries, and current-period expense/revenue cut-off evidence.

`closeRun(operation="addEntries", adjustingEntries=[{type: "accrual|prepaid_amortization|deferred_revenue|reversal", status: "proposed", date: "YYYY-MM-DD", lines: proposedLines, memo: proposedMemo, evidence: sourceEvidence}])`

Branching:
  0 matches → If no accruals, deferrals, prepaid amortization, or reversals are needed, store “no proposed accrual/deferral entries.”
  1 match → Store the proposed entry with `status: "proposed"` and source evidence.
  >1 matches → Store all proposed entries with `status: "proposed"`; use `qbBatch` only in Phase 5 after approval and only for homogeneous journal-entry operations (BATCH SAFETY GUARD blocks qbBatch here unless Phase 5 approval and duplicate coverage exist).

Proposed entry patterns:
- Accrued expenses: Debit Expense, Credit Accrued Liabilities.
- Prepaid amortization: Debit Expense, Credit Prepaid Asset.
- Deferred revenue: Debit Deferred Revenue, Credit Revenue.
- Prior-month accrual reversals: reverse the prior journal-entry debit and credit lines before proposing new accruals.

### Step 3: Propose depreciation
Month-end mode: propose entries only; do not post without approval.

Read: prior depreciation journal entries, fixed asset balances, accumulated depreciation accounts, and prior monthly depreciation amounts.

`qbFetchTransactions(transactionType="JournalEntry", startDate="prior-365-days-start", endDate="YYYY-MM-DD", memoContains="depreciation")`

Branching:
  0 matches → Propose depreciation only if source evidence supports the fixed asset and useful-life schedule; otherwise `flagForReview(aiReasoning="No prior depreciation journal entries were found for YYYY-MM close; depreciation amount cannot be confirmed against history.")`
  1 match → Compare the proposed monthly amount to the prior amount; if inconsistent, require explanation in the proposed memo before adding the entry.
  >1 matches → Review monthly consistency; if the proposed depreciation is inconsistent with prior months without explanation, `flagForReview(aiReasoning="Proposed depreciation for YYYY-MM is inconsistent with prior monthly depreciation journal entries and no explanation was provided.")`

`closeRun(operation="addEntries", adjustingEntries=[{type: "depreciation", status: "proposed", date: "YYYY-MM-DD", lines: [{account: "Depreciation Expense", debit: amount}, {account: "Accumulated Depreciation", credit: amount}], memo: "Monthly depreciation — asset description", evidence: depreciationEvidence}])`

Branching:
  0 matches → Store “no proposed depreciation entry.”
  1 match → Store the proposed depreciation entry with `status: "proposed"` and memo including asset description, e.g. `Monthly depreciation — Office Equipment`.
  >1 matches → Store all proposed depreciation entries with `status: "proposed"` and asset-specific memos.

### Step 4: Update adjusting-entry close status
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: AP/AR reviewed; adjusting entries are proposals only
- Account basis: prior journal entries and source schedules read
- Mode: interactive

`closeRun(operation="updateStep", step={id: "accruals", status: "pass|action_needed", evidence: accrualEvidence})`

`closeRun(operation="updateStep", step={id: "depreciation", status: "pass|action_needed", evidence: depreciationEvidence})`

`closeRun(operation="updateChecks", checks=[{id: 11, status: "pass|action_needed", evidence: prepaidEvidence}, {id: 12, status: "pass|action_needed", evidence: depreciationEvidence}, {id: 13, status: "pass|action_needed", evidence: accrualEvidence}])`

Branching:
  0 matches → If the portal update fails, retry once; if it fails again, `flagForReview(aiReasoning="closeRun update failed for proposed accruals, prepaid amortization, depreciation, or deferrals in YYYY-MM.")`
  1 match → Proceed to Phase 4.
  >1 matches → Use the close run identified in Phase 1 Step 1 and do not update any other close run.

## Phase 4 — Build close run proposal

### Step 1: Read financial statements
Read: Profit and Loss, Balance Sheet, Cash Flow, prior month comparatives, same month last year comparatives, revenue, expenses, net income, gross margin, cash change, assets, liabilities, and equity.

`qbReports(reportType="ProfitAndLoss", period="YYYY-MM", compareTo=["prior_month", "same_month_last_year"])`

`qbReports(reportType="BalanceSheet", period="YYYY-MM")`

`qbReports(reportType="CashFlow", period="YYYY-MM")`

Branching:
  0 matches → `flagForReview(aiReasoning="Required financial statement reports did not return for YYYY-MM; cannot prepare close proposal or financial review.")`
  1 match → Store available report and fetch missing required statements before continuing.
  >1 matches → Compare P&L to prior month and same month last year, verify Balance Sheet Assets = Liabilities + Equity, and reconcile Cash Flow to cash change on the Balance Sheet.

### Step 2: Analyze statement issues
Read: financial statement variances, large/unusual entries, revenue recognition, partial-month comparability, and cash-flow tie-out.

`qbReports(reportType="TransactionList", period="YYYY-MM", filter="large_unusual_entries")`

Branching:
  0 matches → Store “no large/unusual transaction list returned”; continue if financial statements tie out.
  1 match → If the item is unexplained and greater than `2x average`, mark check 9 action_needed.
  >1 matches → Mark check 9 action_needed for any unexplained amount greater than `2x average`; flag gross margin variance greater than `5%` versus prior periods; do not compare a partial month to a full month without noting the partial-month limitation.

### Step 3: Assemble the 16-point checklist
Read: all Phase 1–4 evidence and proposed adjusting entries.

`closeRun(operation="updateChecks", checks=[
{id: 1, label: "Trial Balance", verify: "Debits = credits", status: "pass|action_needed", evidence: trialBalanceEvidence},
{id: 2, label: "Undeposited Funds", verify: "Balance at $0 or near-zero", status: "pass|action_needed", evidence: undepositedFundsEvidence},
{id: 3, label: "Uncategorized Transactions", verify: "Nothing in Uncategorized/Ask My Accountant", status: "pass|action_needed", evidence: uncategorizedEvidence},
{id: 4, label: "Suspense Accounts", verify: "Clearing accounts at zero", status: "pass|action_needed", evidence: suspenseEvidence},
{id: 5, label: "AP Accuracy", verify: "Open bills match obligations", status: "pass|action_needed", evidence: apEvidence},
{id: 6, label: "AR Accuracy", verify: "Open invoices match expectations", status: "pass|action_needed", evidence: arEvidence},
{id: 7, label: "Bank Reconciliation", verify: "All bank/CC accounts reconciled", status: "pass|action_needed", evidence: bankReconciliationEvidence},
{id: 8, label: "Recurring Transactions", verify: "All expected recurring items posted", status: "pass|action_needed", evidence: recurringEvidence},
{id: 9, label: "Large/Unusual Entries", verify: "No unexplained amounts > 2x average", status: "pass|action_needed", evidence: unusualEntriesEvidence},
{id: 10, label: "Revenue Recognition", verify: "Revenue in correct period", status: "pass|action_needed", evidence: revenueRecognitionEvidence},
{id: 11, label: "Prepaid Expenses", verify: "Amortization entries recorded", status: "pass|action_needed", evidence: prepaidEvidence},
{id: 12, label: "Depreciation", verify: "Fixed asset depreciation posted", status: "pass|action_needed", evidence: depreciationEvidence},
{id: 13, label: "Accruals", verify: "Known incurred-but-not-billed expenses accrued", status: "pass|action_needed", evidence: accrualEvidence},
{id: 14, label: "Payroll", verify: "All entries recorded and categorized", status: "pass|action_needed", evidence: payrollEvidence},
{id: 15, label: "Sales Tax", verify: "Collected tax accounted for", status: "pass|action_needed", evidence: salesTaxEvidence},
{id: 16, label: "Changes Since Last Close", verify: "No unapproved edits to closed periods", status: "pass|action_needed", evidence: changesSinceLastCloseEvidence}
])`

Branching:
  0 matches → `flagForReview(aiReasoning="closeRun updateChecks returned no result for the 16-point checklist in YYYY-MM; close proposal cannot be trusted.")`
  1 match → Store checklist statuses and evidence.
  >1 matches → Use the checklist update tied to the close run identified in Phase 1 Step 1.

### Step 4: Pull final trial balance
Read: final Trial Balance after all cleanup and proposed entries.

`qbReports(reportType="TrialBalance", period="YYYY-MM")`

Branching:
  0 matches → `flagForReview(aiReasoning="TrialBalance report did not return for YYYY-MM; cannot verify total debits equal total credits for close proposal.")`
  1 match → If total debits equal total credits, mark check 1 pass; if not, investigate one-sided entries or posting errors before approval.
  >1 matches → Use the Trial Balance for `period="YYYY-MM"` and flag conflicting periods with `flagForReview(aiReasoning="Multiple TrialBalance reports were returned for close proposal; cannot verify debits and credits for the intended YYYY-MM period.")`

### Step 5: Present proposal and await approval
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: AP/AR/open items reviewed and reflected in checklist
- Account basis: financial statements, Trial Balance, account health, and proposed adjusting entries summarized
- Mode: interactive

`closeRun(operation="updateStep", step={id: "statements", status: "pass|action_needed", evidence: financialStatementEvidence})`

`closeRun(operation="updateStep", step={id: "trial_balance", status: "pass|action_needed", evidence: trialBalanceEvidence})`

`closeRun(operation="setFinancials", financials={revenue: revenue, expenses: expenses, net_income: net_income, assets: assets, liabilities: liabilities, equity: equity, cash_change: cash_change})`

Branching:
  0 matches → If financials or trial-balance step update fails, retry once; if it fails again, `flagForReview(aiReasoning="closeRun financial statement or trial balance update failed for YYYY-MM close proposal.")`
  1 match → Present the close proposal with proposed entries, 16-point checklist status, financial statement summary, score forecast out of `16`, and action items; ask:
> I prepared the YYYY-MM close proposal. It includes the proposed adjusting entries, financial statement summary, 16-point checklist, and action items. Do you approve posting the proposed entries and completing the close run?
  >1 matches → Use the close run identified in Phase 1 Step 1 and do not update any other close run.

Batch/async handling: never ask mid-batch; use `flagForReview(aiReasoning="Close proposal for YYYY-MM requires approval before posting adjusting entries; batch/async mode cannot request approval mid-run.")`

## Phase 5 — Execute approved close

### Step 1: Verify approval and fetch duplicate coverage for approved entries
Read: approval record from portal, approved adjusting entries, and journal-entry history for duplicate prevention.

`closeRun(operation="get", period="YYYY-MM")`

`qbFetchTransactions(transactionType="JournalEntry", startDate="YYYY-MM-01", endDate="YYYY-MM-DD", memoContains="accrual|deferral|prepaid|depreciation|reversal|month-end close")`

Branching:
  0 matches → Do not post; `flagForReview(aiReasoning="No approved close run or journal-entry duplicate-check data found for YYYY-MM; cannot execute approved close safely.")`
  1 match → Store approval and duplicate-check result; proceed only for entries approved in Phase 4.
  >1 matches → Do not post if approvals conflict; `flagForReview(aiReasoning="Multiple close-run approval records or journal-entry duplicate-check result sets found for YYYY-MM; cannot determine which entries are approved for posting.")`

### Step 2: Record approved adjusting entries
Pre-write evidence:
- Entity: Close run YYYY-MM and approved adjusting entry IDs
- Open-doc: no open AP/AR document is being paid or received by these adjusting entries
- Account basis: approved close proposal and source schedules; journal-entry history duplicate check completed
- Mode: interactive

`qbJournalEntry(date="YYYY-MM-DD", lines=approvedBalancedLines, memo="Month-end close YYYY-MM — approvedEntryMemo")`

Branching:
  0 matches → If no approved adjusting entries exist, skip posting and proceed to completion.
  1 match → Post the approved journal entry only if debits equal credits (JOURNAL ENTRY BALANCE ENFORCER blocks qbJournalEntry here) and approval is present; WRITE SAFETY GUARD and CURRENCY GUARD must allow the write.
  >1 matches → Use `qbBatch` only when all approved operations are JournalEntry type and the duplicate check covers the full date range (BATCH SAFETY GUARD blocks qbBatch here if mixed types or missing duplicate coverage).

`qbBatch(operations=approvedJournalEntryOperations)`

Branching:
  0 matches → Fall back to individual approved `qbJournalEntry` calls.
  1 match → Store posted journal entry IDs.
  >1 matches → Store all posted journal entry IDs and tie each posted entry back to its approved proposal entry.

### Step 3: Update posted entry status
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: approved adjusting entries posted or skipped if none approved
- Account basis: posted journal entry IDs tied to approved proposal entries
- Mode: interactive

`closeRun(operation="addEntries", adjustingEntries=[{id: approvedEntryId, status: "posted", postedTxnId: journalEntryId, date: "YYYY-MM-DD", memo: postedMemo}])`

Branching:
  0 matches → `flagForReview(aiReasoning="Approved adjusting entries were posted in QuickBooks but closeRun addEntries did not update posted status for YYYY-MM.")`
  1 match → Store posted status for the entry.
  >1 matches → Store all posted statuses and verify each posted journal entry maps to one approved proposal entry.

### Step 4: Complete the close run
Pre-write evidence:
- Entity: Close run YYYY-MM
- Open-doc: AP/AR reviewed; unresolved items listed as actionItems
- Account basis: final reports, checklist evidence, and posted approved entries stored
- Mode: interactive

`closeRun(operation="complete", score=N, actionItems=N)`

Branching:
  0 matches → Retry once; if it fails again, `flagForReview(aiReasoning="closeRun complete failed for YYYY-MM after approved entries were posted and checklist score/actionItems were calculated.")`
  1 match → Status becomes `needs_review`; only a CPA can finalize in the portal.
  >1 matches → Use the close run identified in Phase 1 Step 1 and do not complete any other close run.

Audit-ready exit condition: the close run has status `needs_review`, a score from `0-16`, actionItems count, all 16 checklist items marked `pass` or `action_needed` with evidence, financials stored, Trial Balance reviewed for debits equaling credits, Bank/Credit Card account health addressed, AP/AR reviewed, approved adjusting entries posted or explicitly absent, unresolved items documented for CPA review, and no unapproved edits to closed periods left unexplained.

## Troubleshooting

WRITE SAFETY GUARD blocks qbJournalEntry — qbMasterData or qbFetchTransactions was not called before posting → return to Phase 5 Step 1, fetch journal-entry duplicate coverage, verify accounts with `qbMasterData(entityTypes=["account"])`, then retry Phase 5 Step 2.

JOURNAL ENTRY BALANCE ENFORCER blocks qbJournalEntry — approved entry debits and credits do not balance → return to Phase 3 Step 2 or Phase 3 Step 3, correct proposed lines, re-approve in Phase 4 Step 5, then retry Phase 5 Step 2.

BATCH SAFETY GUARD blocks qbBatch — batch contains mixed transaction types, lacks master data, or lacks duplicate coverage for the full date range → split into homogeneous JournalEntry batches, call `qbMasterData(entityTypes=["account"])`, rerun Phase 5 Step 1, then retry Phase 5 Step 2.

CURRENCY GUARD blocks qbJournalEntry — transaction currency differs from source account currency and no exchangeRate or conversion information is present → add explicit exchangeRate or currency conversion handling in the approved entry before retrying Phase 5 Step 2.

FLAG FOR REVIEW QUALITY GUARD blocks flagForReview — aiReasoning is missing, generic, or under 20 characters → rewrite aiReasoning with the exact report, account, period, amount, and reason for CPA review, then retry the flag in the same phase and step.

qbReports returns no TrialBalance — QuickBooks did not provide the final trial balance for the close period → rerun Phase 4 Step 4 with `period="YYYY-MM"` and do not complete Phase 5 Step 4 until Trial Balance evidence is stored.

closeRun complete fails — portal did not mark the close run complete after score/actionItems were calculated → retry Phase 5 Step 4 once; if it fails again, `flagForReview(aiReasoning="Portal closeRun complete failed for YYYY-MM after score and actionItems were calculated; CPA cannot finalize until portal status is corrected.")`