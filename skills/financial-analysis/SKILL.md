---
name: financial-analysis
description: Generate QuickBooks financial reports, compute ratios, identify trends, and provide CFO-level insights. Activate when the user mentions P&L, balance sheet, cash flow, AR/AP aging, financial ratios, margins, revenue trends, customer or vendor concentration, cash runway, or a financial health check.
---

## Report routing

| Analysis need | `reportType` | Key params |
|---------------|-------------|------------|
| P&L | `ProfitAndLoss` | `startDate`, `endDate`, `summarizeBy="Month"`, `accountingMethod` |
| P&L drill-down | `ProfitAndLossDetail` | same |
| Balance sheet | `BalanceSheet` | `asOfDate` (not date range) |
| Cash flow | `CashFlow` | `startDate`, `endDate`, `summarizeBy="Month"`, `accountingMethod` |
| AR aging | `AgedReceivables` | `asOfDate` |
| AP aging | `AgedPayables` | `asOfDate` |
| Trial balance | `TrialBalance` | `startDate`, `endDate`, `accountingMethod` |
| Customer concentration | `SalesByCustomer`, `CustomerIncome` | period |
| Product revenue | `SalesByProduct` | period |
| Vendor spend | `VendorExpenses` | period |
| Statistical outliers | `qbAccountHealth` | `accountTypes=["Bank","CreditCard"]`, period |

Pull P&L with current period + prior period + same period last year for valid comparisons.

## Critical constraints

**Accounting basis:** Set one `accountingMethod` for the full analysis and reuse it in every `qbReports` call. Never mix Accrual and Cash basis — mixing invalidates period comparisons.

**Partial month:** Never compare a partial month to a full month without noting the limitation. Offer same-elapsed-days comparison instead.

**Missing data handling:**
- Revenue = 0 → don't compute margins
- COGS = 0 → don't assume healthy gross margin; could be account-mapping issue — ask or flag
- Current liabilities = 0 → don't compute current ratio or quick ratio
- AR missing → don't compute DSO
- AP missing or annual COGS = 0 → don't compute DPO
- Equity ≤ 0 → flag as critical balance-sheet risk, don't compute D/E ratio normally
- Need 3+ comparable data points to call something a trend
- Account for seasonality before flagging a decline

## Alert thresholds

**P&L / Revenue:**
- Revenue declining 3+ consecutive months → revenue trend risk
- Expenses growing faster than revenue → operating leverage deterioration

**Concentration:**
- Top customer > 30% of revenue → concentration risk

**Liquidity / Ratios:**
- Current ratio ≤ 1.5 → liquidity below target
- Quick ratio < 1.0 → tight liquid coverage
- DSO increase month-over-month → collections risk
- AR in >90-day bucket > 10% of total AR → serious collections risk
- AP in >90-day bucket > 10% of total AP → vendor-payment risk
- AP due within 30 days > cash on hand → near-term liquidity risk
- Debt-to-equity > 2.0 → elevated leverage
- Equity ≤ 0 → critical balance-sheet risk

**Cash:**
- Cash runway < 3 months → critical cash alert
- Operating cash flow negative while P&L shows profit → investigate AR timing

**Unusual entries:**
- Any unexplained entry > 2× period average → flag

## Output structure

1. **Headline** — one sentence on overall health
2. **Key Metrics Table** — numbers with period-over-period comparison; never present a number without comparison context
3. **Trends** — improving / declining / stable with percentages (3+ data points required to call a trend)
4. **Risks** — issues matching alert thresholds above
5. **Recommendations** — specific, actionable next steps

Delegate any bookkeeping actions (journal entries, reclassifications, accruals) to the relevant transaction, journal-entry, or month-end skill — never post from this skill.

## Gotchas

1. **One accountingMethod for the full analysis** — mixing Accrual/Cash invalidates every comparison
2. **COGS = 0 is not always healthy** — it may mean expense accounts aren't mapped to COGS; ask first
3. **Denominator protection** — never divide by zero; state the analysis is unavailable instead
4. **Trend requires 3+ points** — two data points is a change, not a trend
5. **flagForReview aiReasoning must be specific** — FLAG QUALITY GUARD blocks generic reasons (<20 chars); include report name, period, metric, and why CPA input is needed
