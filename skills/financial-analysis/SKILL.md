---
name: financial-analysis
description: Generate QuickBooks financial reports, analyze business health, compute ratios, identify trends, and provide actionable CFO-level insights. Activate when the user mentions P&L, balance sheet, cash flow, AR/AP aging, financial ratios, margins, revenue trends, customer/vendor concentration, cash runway, or a financial health check.
---

| Situation | Phase |
|---|---|
| Profitability, margins, revenue, expenses, customer/product/vendor trends | Phase 1 |
| Assets, liabilities, equity, liquidity data, trial balance support | Phase 2 |
| Cash flow, burn, runway, operating/investing/financing flows | Phase 3 |
| Who owes the business money or collections timing | Phase 4 |
| What the business owes vendors or upcoming obligations | Phase 5 |

## Phase 1 — Analyze Profit & Loss

### Step 1: Load the financial analysis workflow
`getGuide(guideType="financial_analysis")`

Branching:
  0 matches → Continue with this skill’s phases and use the user-specified period, comparison period, and accountingMethod.
  1 match   → Store the guide workflow and follow it for report sequencing.
  >1 matches → Use the most recent `financial_analysis` guide result.

Set one accountingMethod for the full analysis and reuse it in every `qbReports` call. If the requested comparison would mix Accrual and Cash basis or compare a partial month to a full month, do not run the comparison as requested.

Interactive uncertainty prompt:
> I can compare this partial month only to the same elapsed days in the prior period, or wait until month-end. Which comparison should I use?

Batch/async uncertainty action:
`flagForReview(aiReasoning="Requested financial analysis would mix accounting basis or compare a partial month to a full month; comparison period must be normalized before reporting.")` (FLAG FOR REVIEW QUALITY GUARD enforces aiReasoning specificity)

Month-end handling:
Propose month-end adjustments, accruals, or reclasses only; do not post entries from this skill without approval. If the user approves posting, delegate to the journal-entry/month-end workflow; do not duplicate it here.

### Step 2: Pull P&L reports with prior-period context
`qbReports(reportType="ProfitAndLoss", startDate=currentMonthStart, endDate=currentMonthEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="ProfitAndLoss", startDate=priorComparableStart, endDate=priorComparableEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="ProfitAndLoss", startDate=quarterStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="ProfitAndLoss", startDate=priorQuarterComparableStart, endDate=priorQuarterComparableEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="ProfitAndLoss", startDate=yearStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="ProfitAndLoss", startDate=priorYearComparableStart, endDate=priorYearComparableEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

Branching:
  0 matches → State that P&L data is unavailable for the requested period; ask the user for a different period in interactive mode or flag for review in batch/async.
  1 match   → Store revenue, COGS, operating income, net income, expense categories, period totals, and monthly columns.
  >1 matches → Store all returned P&L reports by period label; use only comparable periods in the analysis.

Compute and interpret gross margin, operating margin, and net margin from the P&L data. If revenue is zero, do not compute margins; state that margin analysis is unavailable because there is no revenue denominator. If COGS is zero or missing, do not treat gross margin as automatically healthy; state that COGS is absent and confirm whether the business has no cost of goods sold or whether COGS mapping is incomplete.

Interactive COGS prompt:
> COGS is zero or missing for this period. Is this expected for this business model, or should I treat this as a possible account-mapping issue?

Batch/async COGS action:
`flagForReview(aiReasoning="COGS is zero or missing in the P&L, so gross margin and DPO interpretation may be misleading unless the business truly has no cost of goods sold.")` (FLAG FOR REVIEW QUALITY GUARD enforces aiReasoning specificity)

Alert thresholds:
- Revenue declining for 3+ consecutive months → flag as a revenue trend risk.
- Expense categories growing faster than revenue → flag as operating leverage deterioration.
- Margin compression versus prior periods → flag as profitability deterioration.

### Step 3: Pull P&L detail for drill-down
`qbReports(reportType="ProfitAndLossDetail", startDate=periodStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

Branching:
  0 matches → Continue with summary-level P&L only and state that transaction-level drill-down is unavailable.
  1 match   → Store transaction-level revenue and expense lines for anomaly explanation.
  >1 matches → Store all detail reports and group by account, vendor, customer, product, and month.

Use P&L detail to explain material changes in revenue, COGS, operating expenses, and net income. Do not present an account movement as a trend unless there are 3+ comparable data points.

### Step 4: Analyze customer, product, and vendor concentration
`qbReports(reportType="SalesByCustomer", startDate=periodStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="CustomerIncome", startDate=periodStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="SalesByProduct", startDate=periodStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="VendorExpenses", startDate=periodStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

Branching:
  0 matches → State that concentration and product/vendor drill-down are unavailable for the requested period.
  1 match   → Store customer revenue, customer profitability, product/service revenue, and vendor spending patterns.
  >1 matches → Store all returned drill-down reports and rank customers, products, and vendors by total amount and period-over-period change.

Alert threshold:
Top customer > 30% of revenue → flag as customer concentration risk.

Use `SalesByProduct` to identify which offerings drive revenue and `VendorExpenses` to identify spending pattern changes.

## Phase 2 — Analyze Balance Sheet

### Step 1: Pull the Balance Sheet with prior-month comparison
`qbReports(reportType="BalanceSheet", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

`qbReports(reportType="BalanceSheet", startDate=priorMonthEnd, endDate=priorMonthEnd, accountingMethod=accountingMethod)`

Branching:
  0 matches → State that balance sheet data is unavailable; ask for a valid as-of date in interactive mode or flag for review in batch/async.
  1 match   → Store cash, AR, inventory, current assets, total assets, AP, current liabilities, total liabilities, and equity.
  >1 matches → Store the current and prior-month balance sheets by as-of date; use the latest valid as-of date as the current balance sheet.

Interactive missing-date prompt:
> I could not retrieve a Balance Sheet for that as-of date. What date should I use instead?

Batch/async missing-date action:
`flagForReview(aiReasoning="Balance Sheet report returned no data for the requested as-of date, so assets, liabilities, equity, and liquidity ratios cannot be verified.")` (FLAG FOR REVIEW QUALITY GUARD enforces aiReasoning specificity)

If AR is missing, do not compute DSO until Phase 4 confirms receivables data. If AP is missing, do not compute DPO until Phase 5 confirms payables data. If inventory is missing, use inventory as zero only after stating that no inventory balance was found.

### Step 2: Pull Trial Balance when balance-sheet support is needed
`qbReports(reportType="TrialBalance", startDate=periodStart, endDate=periodEnd, accountingMethod=accountingMethod)`

Branching:
  0 matches → Continue with Balance Sheet only and state that Trial Balance support is unavailable.
  1 match   → Store all account balances and confirm total debits equal total credits.
  >1 matches → Use the Trial Balance matching the Balance Sheet accountingMethod and period.

Use the Trial Balance to support account-level explanations for Balance Sheet movements, debit/credit integrity, and unusual account balances.

## Phase 3 — Analyze Cash Flow

### Step 1: Pull Cash Flow reports
`qbReports(reportType="CashFlow", startDate=currentMonthStart, endDate=currentMonthEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="CashFlow", startDate=yearStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

Branching:
  0 matches → State that Cash Flow data is unavailable and rely on Balance Sheet cash movement only if Phase 2 data exists.
  1 match   → Store operating cash flow, investing cash flow, financing cash flow, net cash change, cash on hand, and monthly burn.
  >1 matches → Store current-month and YTD Cash Flow reports separately; reconcile direction of cash movement across both.

Separate operating, investing, and financing flows in the interpretation. Compute cash runway from cash on hand and average monthly cash burn.

Alert thresholds:
- Operating cash flow negative while P&L shows profit → investigate AR timing in Phase 4.
- Cash runway < 3 months → critical cash alert.

### Step 2: Connect cash flow to AR and AP timing
`qbReports(reportType="AgedReceivables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

`qbReports(reportType="AgedPayables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

Branching:
  0 matches → State that AR/AP timing cannot be used to explain cash movement.
  1 match   → Store overdue AR and upcoming AP as cash timing drivers.
  >1 matches → Store aging reports by as-of date and use the report matching the Cash Flow period end.

Use AR aging for incoming cash timing and AP aging for upcoming obligations.

## Phase 4 — Analyze AR Aging

### Step 1: Pull Aged Receivables
`qbReports(reportType="AgedReceivables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

Branching:
  0 matches → If the Balance Sheet also has no AR, state that AR appears unavailable or not used; otherwise ask for review because Balance Sheet AR exists but aging detail is missing.
  1 match   → Store customer balances, overdue buckets, total AR, and oldest receivables.
  >1 matches → Use the Aged Receivables report matching the Balance Sheet as-of date.

Interactive AR mismatch prompt:
> The Balance Sheet shows AR, but the Aged Receivables report did not return detail. Should I use a different as-of date or treat this as a QuickBooks reporting issue?

Batch/async AR mismatch action:
`flagForReview(aiReasoning="Balance Sheet indicates AR exists but AgedReceivables returned no detail, so receivables quality and DSO cannot be validated.")` (FLAG FOR REVIEW QUALITY GUARD enforces aiReasoning specificity)

Use AR aging to identify who owes the business money and how overdue the balances are.

Alert thresholds:
- Any month-over-month DSO increase → flag as a collections risk.
- More than 10% of total AR in the >90 day bucket → flag as a serious collections risk.

## Phase 5 — Analyze AP Aging

### Step 1: Pull Aged Payables
`qbReports(reportType="AgedPayables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

Branching:
  0 matches → If the Balance Sheet also has no AP, state that AP appears unavailable or not used; otherwise ask for review because Balance Sheet AP exists but aging detail is missing.
  1 match   → Store vendor balances, due-date buckets, total AP, overdue AP, and upcoming obligations.
  >1 matches → Use the Aged Payables report matching the Balance Sheet as-of date.

Interactive AP mismatch prompt:
> The Balance Sheet shows AP, but the Aged Payables report did not return detail. Should I use a different as-of date or treat this as a QuickBooks reporting issue?

Batch/async AP mismatch action:
`flagForReview(aiReasoning="Balance Sheet indicates AP exists but AgedPayables returned no detail, so payables timing and DPO cannot be validated.")` (FLAG FOR REVIEW QUALITY GUARD enforces aiReasoning specificity)

Use AP aging to identify what the business owes and when obligations are due.

Alert thresholds:
- More than 10% of total AP in the >90 day bucket → flag as a vendor-payment risk.
- AP due within 30 days greater than cash on hand from Phase 2 → flag as a near-term liquidity risk.

## Phase 6 — Compute Ratios

### Step 1: Pull ratio source reports when not already available
`qbReports(reportType="ProfitAndLoss", startDate=yearStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

`qbReports(reportType="BalanceSheet", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

`qbReports(reportType="AgedReceivables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

`qbReports(reportType="AgedPayables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

Branching:
  0 matches → Do not compute ratios; state which required source reports are missing.
  1 match   → Store annual revenue, annual COGS, AR, AP, current assets, current liabilities, inventory, total liabilities, and equity.
  >1 matches → Use source reports with the same accountingMethod and period end; discard non-comparable reports.

Compute and interpret current ratio, quick ratio, DSO, DPO, and debt-to-equity using the stored report data. If current liabilities are zero, do not compute current ratio or quick ratio. If AR is missing, do not compute DSO. If annual revenue is zero, do not compute DSO. If AP is missing, do not compute DPO. If annual COGS is zero or missing, do not compute DPO. If equity is zero or negative, do not compute debt-to-equity as a normal ratio; flag leverage as structurally risky.

Alert thresholds:
- Current ratio ≤ 1.5 → flag as liquidity below target.
- Quick ratio < 1.0 → flag as tight liquid coverage.
- Any month-over-month DSO increase → flag as a collections risk.
- Debt-to-equity > 2.0 → flag as elevated leverage.
- Equity ≤ 0 → flag as critical balance-sheet risk.

## Phase 7 — Run Health Check and Present Findings

### Step 1: Run quick health check
`qbAccountHealth(accountTypes=["Bank","CreditCard"])`

`qbReports(reportType="ProfitAndLoss", startDate=currentMonthStart, endDate=currentMonthEnd, accountingMethod=accountingMethod)`

`qbReports(reportType="AgedReceivables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

`qbReports(reportType="AgedPayables", startDate=asOfDate, endDate=asOfDate, accountingMethod=accountingMethod)`

Branching:
  0 matches → State that quick health check cannot be completed because required health, P&L, AR, or AP data is missing.
  1 match   → Present health scores, headline P&L, overdue amounts, and flags.
  >1 matches → Use all returned health signals and summarize by account, report, and severity.

Use `qbAccountHealth` to detect statistical outliers automatically.

### Step 2: Run anomaly scan for full analysis
`qbAccountHealth(accountTypes=["Bank","CreditCard"], startDate=periodStart, endDate=periodEnd)`

Branching:
  0 matches → Continue without automated outlier detection and state that anomaly scan was unavailable.
  1 match   → Store account health scores and statistical outliers.
  >1 matches → Store all account health results and rank outliers by severity.

Use automated outliers to support, not replace, the report-based analysis from Phases 1–6.

### Step 3: Synthesize CFO-level output
`qbReports(reportType="ProfitAndLoss", startDate=periodStart, endDate=periodEnd, summarizeBy="Month", accountingMethod=accountingMethod)`

Branching:
  0 matches → Provide only the findings supported by already-fetched reports and state which requested analysis could not be completed.
  1 match   → Produce the final analysis using all stored report data.
  >1 matches → Use the report matching the final selected period and accountingMethod.

Structure the final output:
1. Headline — one sentence on overall health.
2. Key Metrics Table — numbers with period-over-period comparison.
3. Trends — improving, declining, or stable with percentages.
4. Risks — issues requiring attention.
5. Recommendations — specific, actionable next steps.

Presentation rules:
- Lead with the most impactful finding.
- Never present a number without comparison context.
- Use percentages for changes, not just absolute numbers.
- Need 3+ data points to call something a trend.
- Account for seasonality before flagging declines.

If recommendations require bookkeeping actions, delegate to the relevant transaction, journal-entry, or month-end skill instead of posting from this skill. If that delegated workflow later posts a journal entry, JOURNAL ENTRY BALANCE ENFORCER enforces balanced debits and credits.

## Troubleshooting
FLAG FOR REVIEW QUALITY GUARD blocks `flagForReview` — aiReasoning is missing, under 20 characters, or generic → rewrite the reason with the specific missing report, metric, period, and decision impact, then retry the flag action from the phase step that raised it.

`qbReports` returns no rows — the report type, period, as-of date, or accountingMethod may not have data → retry with a validated date range in the same phase step; if still empty, state the unavailable analysis rather than inventing numbers.

`qbReports` returns non-comparable reports — accountingMethod or period boundaries differ → return to Phase 1 Step 1 and normalize accountingMethod and comparable periods before presenting analysis.

`qbAccountHealth` unavailable — account health scan cannot detect statistical outliers → continue with report-based analysis in Phase 7 Step 2 and disclose that automated outlier detection was unavailable.

Balance Sheet AR/AP does not match aging detail — report timing or QuickBooks aging data may be inconsistent → use Phase 4 Step 1 for AR mismatch or Phase 5 Step 1 for AP mismatch, then ask in interactive mode or flag for review in batch/async.