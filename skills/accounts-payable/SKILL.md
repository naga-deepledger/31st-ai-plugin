---
name: accounts-payable
description: Manage accounts payable — enter bills, pay vendors, apply vendor credits, monitor aging, and track outstanding obligations. Use when the user mentions bills, vendor payments, AP aging, bill pay, or outstanding payables.
---

# Accounts Payable Skill

Manage the full AP lifecycle: entering vendor bills, scheduling and making payments, applying vendor credits, and monitoring what you owe.

## Trigger

Activate when the user wants to:
- Enter a vendor bill
- Pay a vendor bill
- Apply a vendor credit
- Check outstanding or overdue payables
- Review vendor spending
- Process a batch of bill payments
- Track payment terms and due dates

## The AP Flow

```
Bill → BillPayment → Bank Account
```

- `qbBill` creates the payable (AP goes up, no cash out yet)
- `qbBillPayment` pays the bill from a bank or credit card account (AP goes down, cash goes down)

For immediate-pay purchases with no bill: `qbExpense` records the payment directly.

## When to Use Bill vs Expense

| Situation | Tool | Why |
|-----------|------|-----|
| Vendor sends an invoice to pay later | `qbBill` | Creates AP, tracks the obligation |
| Paid vendor immediately (card swipe, ACH) | `qbExpense` | No AP needed, money already left |
| Paying a previously entered bill | `qbBillPayment` | Clears the AP, records the payment |

**Key rule**: If a `qbBill` already exists for this vendor+amount+date, use `qbBillPayment` — never create a duplicate `qbExpense`.

## Workflow: Enter a Bill

1. **Lookup** — `qbMasterData(detailedInfo="vendor", filter=vendorName)` for vendor ID. If filter returns >1 vendor with similar names, present options to user and confirm before proceeding — do not pick one.
2. **Open-document check (separate call, no date window)** — `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`. Run this BEFORE the history fetch. Outstanding status is not date-bounded; a date-scoped call would miss old open bills. If a matching outstanding bill is found, do NOT create a new bill — switch to the **Pay a Bill** workflow below using the `billId`(s) already returned by this check; skip the "Find outstanding bills" fetch in that workflow since you already have them.
3. **Fetch vendor history (account inference)** — `qbFetchTransactions(entityId=vendorId, entityType="Vendor", lookbackDays=365)`. Apply the CONSISTENCY RULE: use the inferred expense account ONLY IF all five criteria hold — (a) ≥ 3 prior transactions in last 365 days, (b) dominant account ≥ 70%, (c) no second account ≥ 20%, (d) current amount within 5× median of dominant-account transactions, (e) most recent dominant-account transaction < 180 days old. Fails any criterion → flag or confirm with user before proceeding.
4. **Build bill** — Set `vendorId`, `txnDate`, `dueDate`, `lines` with inferred or confirmed account
5. **Confirm** — Show: vendor, total, due date, expense categories
6. **Record** — `qbBill`
7. **Attach** — `qbAttachFile` (entityType = "Bill") — fetch from portal, local file, drive, or user upload; preferred for audit-ready books

## Workflow: Pay a Bill

1. **Find outstanding bills** — `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`
2. **Select bills** — Identify which bill(s) to pay (by amount, date, or vendor)
3. **Choose payment account** — `qbMasterData` for Bank/CC account IDs
4. **Record** — `qbBillPayment` with `vendorId`, `paymentDate`, `bankAccountId`, `bills` array (billId + amount per bill)
5. **Partial payments** — Pay less than the full bill amount; the bill remains partially outstanding
6. **Multiple bills** — Pay several bills to one vendor in a single BillPayment using the `bills` array
7. **Attach** — `qbAttachFile` (entityType = "BillPayment") — remittance advice or bank confirmation; preferred for audit-ready books
8. **Confirm** — Show: vendor, bills being paid, total, payment account

## Workflow: Record an Expense (Immediate Payment)

1. **Lookup** — `qbMasterData(detailedInfo="vendor", filter=vendorName)` for vendor ID; `qbMasterData(entityTypes=["account"])` for source account ID. If filter returns >1 vendor with similar names, present options to user and confirm before proceeding — do not pick one.
2. **Open-document check (separate call, no date window)** — `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`. Run this BEFORE the history fetch. If a matching outstanding bill exists, do NOT record as an Expense — switch to the **Pay a Bill** workflow using the `billId`(s) from this check's response. The EXPENSE TYPE GUARD hook enforces this; the skill must pre-satisfy it, not rely on the hook to catch it.
3. **Fetch vendor history (account inference)** — `qbFetchTransactions(entityId=vendorId, entityType="Vendor", lookbackDays=365)`. Apply the CONSISTENCY RULE: use the inferred expense account ONLY IF all five criteria hold — (a) ≥ 3 prior transactions in last 365 days, (b) dominant account ≥ 70%, (c) no second account ≥ 20%, (d) current amount within 5× median of dominant-account transactions, (e) most recent dominant-account transaction < 180 days old. Fails any criterion → flag for review rather than recording.
4. **Record** — `qbExpense` with `paymentType`, `accountId` (source), `vendorId`, `lines` (category accounts)
5. **Attach** — `qbAttachFile` (entityType = "Purchase") — receipt from portal, local file, drive, or user upload; preferred for audit-ready books
6. **Rule** — Source account (where money comes from) must differ from line accountId (what it was spent on)

## Workflow: Apply Vendor Credit

When a vendor issues a credit or refund:

1. **Create credit** — `qbCredit(creditType="vendor")` with `vendorId`, `txnDate`, `lines` (account + amount)
2. **Apply to payment** — In `qbBillPayment`, include the VendorCredit in the `bills` array with `txnType: "VendorCredit"`
3. Use for: vendor refunds, billing adjustments, returned goods

## Workflow: AP Aging Review

1. **Pull aging** — `qbReports(reportType="AgedPayables")` for summary, `AgedPayablesDetail` for line items
2. **Calculate DPO** — Days Payable Outstanding = AP / (Annual COGS / 365)
3. **Bucket analysis** — Present in aging buckets: Current, 1-30, 31-60, 61-90, 90+
4. **Flag concerns**:
   - Bills past due → late payment penalties, vendor relationship risk
   - AP growing faster than revenue → cash flow pressure
   - Large upcoming payments → cash planning needed
5. **Recommendations** — Bills to prioritize for payment, vendors to negotiate terms with

## Workflow: Vendor Spending Analysis

1. **Pull report** — `qbReports(reportType="VendorExpenses")` for spending by vendor
2. **Identify** — Top vendors by total spend, fast-growing vendors, new vendors
3. **Compare** — Current period vs prior period
4. **Flag** — Vendor spend up 20%+ without corresponding revenue growth

## Safety Checklist

- [ ] `qbMasterData` lookup completed — valid vendor and account IDs; if >1 match on filter, user confirmed which vendor
- [ ] Open-document check run as a **separate call** with `outstandingOnly=true` and **no date window** — before history fetch or account inference
- [ ] Consistency rule applied: all five criteria checked before using history-inferred expense account
- [ ] Vendor history fetched — expense account inferred from past transactions, not guessed
- [ ] Outstanding bills checked before recording an expense — `qbBillPayment` used when a match exists
- [ ] Source account differs from category account on expense lines
- [ ] User confirmation before recording

## Common Mistakes to Avoid

- Recording an `qbExpense` when an outstanding `qbBill` already exists → use `qbBillPayment`
- Forgetting `vendorId` on expenses → breaks vendor spending reports and audit trail
- Source account same as category account → accounting error
- Creating a BillPayment without checking for outstanding bills first
- Wrong `entityType` when attaching documents: Expense = "Purchase" in the QB API
- Paying a bill from the wrong bank account
