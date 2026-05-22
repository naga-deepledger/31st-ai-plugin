---
name: journal-entries
description: Records, reverses, and corrects QuickBooks Online journal entries for reclassification, accruals, depreciation, intercompany allocations, and closed-period corrections. Activate when the user mentions journal entries, JE, adjusting entries, reclassifying, reversing, accruals, depreciation, intercompany, or GL corrections.
---

## When to use a JE vs delegate

| Purpose | Tool |
|---------|------|
| Reclassification | `qbJournalEntry` |
| Accrual / deferral | `qbJournalEntry` |
| Depreciation | `qbJournalEntry` |
| Intercompany allocation | `qbJournalEntry` |
| Adjusting entry (closed period) | `qbJournalEntry` |
| Reversing entry | `qbJournalEntry` |
| Bank-to-bank transfer | `qbTransfer` — not JE |
| Paying a vendor bill | `qbBillPayment` — not JE |
| Receiving a customer payment | `qbReceivePayment` — not JE |
| Open-period error correction | Void + re-record in correct skill — not JE |

## Standard JE workflow

1. `qbMasterData(entityType="Account", query=accountNames)` — resolve all debit/credit account IDs
2. `qbFetchTransactions(transactionType="JournalEntry", startDate=dateMinus3Days, endDate=datePlus3Days, amount=totalDebits, memo=memo)` — duplicate check
3. Verify debits = credits exactly — JOURNAL ENTRY BALANCE ENFORCER hard-blocks `qbJournalEntry` if not balanced
4. In interactive mode, show proposed entry (date, accounts, amounts, memo) and wait for approval
5. `qbJournalEntry(date=date, lines=lines, memo=memo)`
6. `qbAttachFile(entityType="JournalEntry", entityId=journalEntryId, ...)` — attach supporting schedule when available

## Reversal workflow

1. `qbFetchTransactions(transactionType="JournalEntry", transactionId=originalId)` — fetch original
2. Swap debits and credits from original lines
3. Date = first day of next period (use different date only if user or CPA specifies)
4. Memo = `Reversal of [original ref] from [date]`
5. Duplicate check for reversal period → `qbJournalEntry`

## Correction workflows

**Open-period error:**
- `qbFetchTransactions` to locate original transaction
- VOID TRANSACTION GUARD: target must appear in most recent `qbFetchTransactions` result
- `qbVoidTransaction(transactionType, transactionId)` → re-record correct transaction in the appropriate skill

**Closed-period error:**
- Do NOT modify the original
- Build offsetting JE (swap debits/credits from original lines)
- Date = current open period
- Memo = `Correction of [original ref] from [date]`
- `qbJournalEntry` in current period; then re-record the correct entry if needed

## Intercompany entries

Use Due To / Due From accounts — not ordinary income or expense accounts — unless CPA explicitly instructs otherwise. Each entity must have its own balanced JE. Propose entries; do not post without CPA approval.

1. `qbMasterData(entityType="Account", query=intercompanyAccountNames)` — resolve Due To/Due From IDs
2. Duplicate check for intercompany period
3. Show proposed balanced entries per entity → wait for approval
4. `qbJournalEntry` per entity after approval

## Gotchas

1. **JOURNAL ENTRY BALANCE ENFORCER is a hard block** — debits must exactly equal credits before calling `qbJournalEntry`; prepare lines first, then call
2. **Bank-to-bank transfer → `qbTransfer`** — a JE for an internal transfer creates incorrect reconciliation entries
3. **Closed-period corrections**: never modify the original; offset only in the current open period
4. **Intercompany**: Due To/Due From accounts, not income/expense — using wrong accounts corrupts entity-level financials
5. **Reversal date defaults to next period** — different date must come from user or CPA; don't assume

In batch/async mode: never ask mid-batch; `flagForReview` for any JE requiring approval or with unresolved account IDs.
