---
name: bank-reconciliation
description: Reconcile bank and credit card accounts by running health checks, resolving duplicates and uncategorized items, delegating bank-feed processing, then completing the QuickBooks reconciliation via browser. Activate when the user mentions reconciliation, bank matching, uncleared items, account health, bank-feed cleanup, or closing the books.
---

## Routing

| Situation | Phase |
|-----------|-------|
| Check account health, duplicates, uncategorized items | Phase 1 |
| Unrecorded bank-feed transactions exist | Phase 2 |
| Bank-feed review queue has flagged items | Phase 2 Step 3 |
| Ready to finalize reconciliation in QB UI | Phase 3 |

## Phase 1 — Health Check

`qbMasterData(entityTypes=["account"])` → filter to Bank and Credit Card accounts.

`qbAccountHealth(accountId, startDate, endDate)` per account.

Health score thresholds:
- 95–100 → proceed to Phase 2 (or Phase 3 if feed already processed)
- 90–94 → minor cleanup first, then Phase 2
- 80–89 → resolve all flags before touching reconciliation screen
- < 80 → critical; fix before reconciling

Health flags:
- **Duplicates** — same amount + date + vendor
- **Uncategorized** — booked to "Ask My Accountant" or "Uncategorized"
- **Outliers** — amount > 2 standard deviations from mean
- **Past-due** — `dueDate` field older than 7 days (not QB cleared status)

**Resolve duplicates:**
Fetch both candidates: `qbFetchTransactions(transactionId=firstId)` + `qbFetchTransactions(transactionId=secondId)`
Verify true duplicate (same amount + vendor + purpose; two similar charges from one vendor can both be legitimate).

Voidable via tools: `BillPayment`, `Invoice`, `Payment`, `SalesReceipt`, `CreditMemo`, `Purchase`, `RefundReceipt`, `Transfer`
Not voidable via tools: `Bill`, `JournalEntry`, `Deposit`, `Expense`, `VendorCredit` → `flagForReview` with transaction type and matched pair.

VOID TRANSACTION GUARD: void target must appear in the most recent `qbFetchTransactions` result.
`qbVoidTransaction(transactionId, transactionType)` — ask before voiding; voiding preserves audit trail better than deleting.

Re-run `qbAccountHealth` after each void or `flagForReview`. Repeat until no duplicate flags remain.

**Resolve uncategorized:**
`qbFetchTransactions(accountId, filter=["Ask My Accountant","Uncategorized"], startDate, endDate)`

For each uncategorized item:
1. Resolve counterparty: `qbMasterData(detailedInfo="vendor", filter=counterpartyName)` — >1 matches → MULTI-VENDOR AMBIGUITY GUARD blocks; `flagForReview` with candidate list
2. **Open-doc check** — separate call, no date window: `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`. Outstanding status not date-bounded.
   - 1 bill match → the open doc encodes the correct account; route to AP workflow rather than re-categorizing
   - 0 matches → fetch 365-day vendor history; apply 5-criteria consistency rule before any re-categorization: (a) ≥3 txns, (b) dominant ≥70%, (c) no second ≥20%, (d) amount within 5× median, (e) most recent <180 days old — any failure → `flagForReview`

Re-run `qbAccountHealth` after cleanup. Do not open the reconciliation screen until score ≥ 95.

## Phase 2 — Delegate Bank Feed Processing

`bankFeed(action="fetch", accountId, sinceDate=startDate)` — skip items with `alreadyFlagged=true`.

- 0 unrecorded items → proceed to Phase 2 Step 3
- 1+ items → delegate ALL recording decisions to the `bank-feed-processing` skill with `accountId`, `startDate`, `endDate`, and the fetched item list. Do not re-implement recording logic here.

After delegation, items must be marked recorded or flagged via `fetchWorkQueue(source="markRecorded", ...)`.

**Review queue check:**
`fetchWorkQueue(source="bank-feed-processing", accountId, startDate, endDate, status="flagged")`

- 0 flagged items → proceed to Phase 3
- 1+ flagged items → do not finalize. Ask if user wants to wait for CPA resolution or inspect-only mode.
  Batch/async: `flagForReview(aiReasoning="Reconciliation blocked: [N] bank-feed items remain flagged for the period. Do not finalize until the review queue is cleared. Items: [summaries].")`

## Phase 3 — Finalize via Computer Use

**Start browser automation only after:** Phase 1 health score ≥ 95 AND Phase 2 review queue is empty.

Any unresolved item = inspection-only, not finalization.

Activate `computer-use` skill: QBO → Bookkeeping → Reconcile → select account.

**Enter statement details:**
- Beginning balance must match QB's calculated opening balance — **mismatch = STOP**. Prior reconciled transactions may have been changed; do not continue.
- Ending balance from bank statement
- Statement end date

Mark cleared transactions. Investigate suspicious items with `qbFetchTransactions(accountId, startDate, endDate)` while screen stays open.

**Reconciliation difference threshold: $0.00 exactly.**
Any non-zero difference → halt reconciliation. Never force-adjust — a forced adjustment creates a discrepancy that compounds every month.

Ask before "Finish Now":
> The reconciliation difference is $0.00. Do you want me to click "Finish Now"?

Never click "Finish Now" without explicit user confirmation, even with $0.00 difference.

After completion: download/save the QB reconciliation report PDF for audit records.

## Gotchas

1. **Health score < 95 = don't finalize** — open the screen for inspection only; no "Finish Now"
2. **Reconciliation difference must be exactly $0.00** — no force-adjust; non-zero means missing or duplicated transactions
3. **Beginning balance mismatch = full stop** — prior reconciled transactions were changed; never proceed past this point
4. **VOID TRANSACTION GUARD**: fetch both candidates before voiding; not all transaction types are voidable via tools
5. **Delegate bank-feed recording to `bank-feed-processing`** — this skill must not re-implement categorization or inference logic
