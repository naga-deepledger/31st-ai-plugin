---
name: month-end-close
description: Runs the month-end close workflow for QuickBooks Online: pre-close health checks, AP/AR/bank-feed cleanup, proposed adjusting entries, 16-point close proposal, CPA-approved posting, and portal completion. Activate when the user mentions closing the books, month-end close, close period, adjusting entries, accruals, deferrals, depreciation, or preparing financial statements for review.
---

## Routing

| Situation | Phase |
|-----------|-------|
| "Are we ready to close?" or "run close checks" | Phase 1 |
| AP, AR, bank-feed, uncategorized, or reconciliation issues exist | Phase 2 |
| Accruals, depreciation, reversals, or adjusting entries needed | Phase 3 |
| Build close package and await CPA approval | Phase 4 |
| CPA approved — post entries and complete close | Phase 5 |

**Key constraint:** All adjusting entries are proposals in Phase 3. Nothing posts until Phase 5 after CPA approval. Never post entries from Phases 1–4.

## Phase 1 — Pre-close health check

`closeRun(operation="start", period="YYYY-MM", periodLabel="Month Year")` → store `closeRunId`; if close run already exists for period, continue it.

`qbMasterData(entityTypes=["account"])` → filter to Bank and Credit Card accounts.

`qbAccountHealth(accountId, period="YYYY-MM")` per account.
- Score ≥ 90 → ready
- Score < 90 → route to Phase 2

Pull AP/AR/open-item reports:
- `qbReports(reportType="AgedPayables", period="YYYY-MM")`
- `qbReports(reportType="AgedReceivables", period="YYYY-MM")`
- `qbReports(reportType="BalanceSheet", period="YYYY-MM")`
- `qbReports(reportType="TransactionList", period="YYYY-MM", accountNames=["Undeposited Funds","Uncategorized","Ask My Accountant","Suspense","Clearing","Payroll","Sales Tax"])`
- `qbReports(reportType="TransactionList", startDate=priorCloseDate, endDate=periodEnd, filter="changes_since_last_close")`

`closeRun(operation="updateStep", step={id:"reconciliation", status:"pass|action_needed", evidence:...})`
`closeRun(operation="updateChecks", checks=[{id:7, status:"pass|action_needed", evidence:...}])`

## Phase 2 — Clear open items (delegate, don't duplicate)

**Bank-feed / reconciliation issues** → delegate to `bank-feed-processing` skill; do not re-implement recording logic here.
`qbAccountHealth(accountId, period="YYYY-MM")` → if duplicates/uncategorized/outliers → delegate to `bank-feed-processing`; rerun Phase 1 Step 3 after cleanup.

**AP issues** → delegate to `accounts-payable` skill.
`qbReports(reportType="AgedPayables", period="YYYY-MM")` + `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, endDate=periodEnd)` — verify open bills; route incorrect-period bills, missing bills, and past-due AP to AP skill.

**AR issues** → delegate to `accounts-receivable` skill.
`qbReports(reportType="AgedReceivables", period="YYYY-MM")` + `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, endDate=periodEnd)` — AR >90 days past due → route to AR skill as potential bad debt (discuss with CPA before any write-off).

`closeRun(operation="updateStep", step={id:"ap_ar", status:"pass|action_needed", evidence:...})`
`closeRun(operation="updateChecks", checks=[{id:5, status:...}, {id:6, status:...}])`

## Phase 3 — Propose adjusting entries (PROPOSE ONLY — do not post)

**Prior-month reversals first:**
`qbFetchTransactions(transactionType="JournalEntry", startDate=priorMonthStart, endDate=priorMonthEnd, memoContains="accrual|deferral|prepaid|depreciation|reversal")`

Propose reversals of prior accruals/deferrals BEFORE proposing new entries for the same items.

`closeRun(operation="addEntries", adjustingEntries=[{type, status:"proposed", date:"YYYY-MM-DD", lines, memo, evidence}])`

Standard entry patterns:
- Accrued expenses: Dr Expense / Cr Accrued Liabilities
- Prepaid amortization: Dr Expense / Cr Prepaid Asset
- Deferred revenue: Dr Deferred Revenue / Cr Revenue
- Depreciation: Dr Depreciation Expense / Cr Accumulated Depreciation; memo must include asset description, e.g. `Monthly depreciation — Office Equipment`

**Depreciation consistency check:**
`qbFetchTransactions(transactionType="JournalEntry", startDate=prior365, endDate=periodEnd, memoContains="depreciation")`
Compare proposed amount to prior months. Inconsistency without explanation → `flagForReview(aiReasoning="Proposed depreciation for YYYY-MM differs from prior monthly amount and no explanation was provided.")` before adding.

`closeRun(operation="updateStep", step={id:"accruals", status:...})`
`closeRun(operation="updateStep", step={id:"depreciation", status:...})`
`closeRun(operation="updateChecks", checks=[{id:11, status:...}, {id:12, status:...}, {id:13, status:...}])`

## Phase 4 — Build close proposal

Pull financial statements:
- `qbReports(reportType="ProfitAndLoss", period="YYYY-MM", compareTo=["prior_month","same_month_last_year"])`
- `qbReports(reportType="BalanceSheet", period="YYYY-MM")`
- `qbReports(reportType="CashFlow", period="YYYY-MM")`
- `qbReports(reportType="TrialBalance", period="YYYY-MM")`

Verify: Assets = Liabilities + Equity; Cash Flow ties to BS cash change; no unexplained entry >2× period average; gross margin variance >5% vs prior periods.

**16-point checklist:**
```
closeRun(operation="updateChecks", checks=[
  {id:1,  label:"Trial Balance",              verify:"Debits = Credits"},
  {id:2,  label:"Undeposited Funds",           verify:"$0 or near-zero"},
  {id:3,  label:"Uncategorized Transactions",  verify:"Nothing in Uncategorized/Ask My Accountant"},
  {id:4,  label:"Suspense Accounts",           verify:"Clearing accounts at zero"},
  {id:5,  label:"AP Accuracy",                 verify:"Open bills match obligations"},
  {id:6,  label:"AR Accuracy",                 verify:"Open invoices match expectations"},
  {id:7,  label:"Bank Reconciliation",         verify:"All bank/CC accounts reconciled"},
  {id:8,  label:"Recurring Transactions",      verify:"All expected recurring items posted"},
  {id:9,  label:"Large/Unusual Entries",       verify:"No unexplained amounts >2× average"},
  {id:10, label:"Revenue Recognition",         verify:"Revenue in correct period"},
  {id:11, label:"Prepaid Expenses",            verify:"Amortization entries recorded"},
  {id:12, label:"Depreciation",                verify:"Fixed asset depreciation posted"},
  {id:13, label:"Accruals",                    verify:"Known incurred-but-not-billed expenses accrued"},
  {id:14, label:"Payroll",                     verify:"All entries recorded and categorized"},
  {id:15, label:"Sales Tax",                   verify:"Collected tax accounted for"},
  {id:16, label:"Changes Since Last Close",    verify:"No unapproved edits to closed periods"}
])
```

`closeRun(operation="setFinancials", financials={revenue, expenses, net_income, assets, liabilities, equity, cash_change})`
`closeRun(operation="updateStep", step={id:"statements", status:...})`
`closeRun(operation="updateStep", step={id:"trial_balance", status:...})`

Present proposal with proposed entries, checklist status, and financial summary. Ask:
> I prepared the YYYY-MM close proposal. Do you approve posting the proposed entries and completing the close run?

Batch/async: `flagForReview(aiReasoning="Close proposal for YYYY-MM requires approval before posting adjusting entries; batch/async mode cannot request approval mid-run.")`

## Phase 5 — Execute approved close

`closeRun(operation="get", period="YYYY-MM")` — verify approval exists.

`qbFetchTransactions(transactionType="JournalEntry", startDate=periodStart, endDate=periodEnd, memoContains="accrual|deferral|prepaid|depreciation|reversal|month-end close")` — duplicate coverage before posting.

Post ONLY entries approved in Phase 4:
`qbJournalEntry(date, lines=approvedBalancedLines, memo="Month-end close YYYY-MM — [entry memo]")`
JOURNAL ENTRY BALANCE ENFORCER hard-blocks if debits ≠ credits.

Batch option (only for homogeneous JournalEntry operations with full duplicate coverage):
`qbBatch(operations=approvedJournalEntryOperations)` — BATCH SAFETY GUARD requires homogeneous type + verified IDs.

`closeRun(operation="addEntries", adjustingEntries=[{id:approvedEntryId, status:"posted", postedTxnId:journalEntryId, date, memo}])`

`closeRun(operation="complete", score=N, actionItems=N)` → status becomes `"needs_review"`. Only a CPA can finalize in the portal.

**Audit-ready exit condition:** status `needs_review`, score 0–16, all 16 checklist items marked pass/action_needed with evidence, financials stored, Trial Balance debits=credits verified, bank health addressed, AP/AR reviewed, approved entries posted, unresolved items documented for CPA review, no unapproved edits to closed periods.

## Gotchas

1. **Propose-before-execute**: all entries have `status="proposed"` in Phase 3; nothing posts until Phase 5 after explicit CPA approval
2. **`closeRun complete` → `"needs_review"`**: the agent's role ends here; CPA finalizes in the portal — never call complete if AP/AR cleanup is still pending
3. **Prior-month reversals first**: propose reversals BEFORE new accruals for the same period; posting in wrong order double-counts expenses
4. **Depreciation consistency**: proposed amount ≠ prior months without explanation → flagForReview; don't add the entry until explanation is provided
5. **Delegate to other skills**: never re-implement AP, AR, or bank-feed logic here; this skill orchestrates; transaction recording belongs in the respective skills
