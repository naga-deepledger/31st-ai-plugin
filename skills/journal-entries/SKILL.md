---
name: journal-entries
description: Records, reverses, and corrects QuickBooks Online journal entries and adjusting entries, including reclasses, accruals, depreciation, intercompany entries, and closed-period corrections. Activate when the user mentions journal entries, JE, adjusting entries, transfers, reclassifying, reversing, accruals, depreciation, intercompany activity, or GL corrections.
---

| Situation | Phase |
|---|---|
| User asks for a JE, accrual, depreciation, reclass, or adjustment | Phase 1 |
| User asks to move cash between internal bank/credit card accounts | Phase 1 Step 2 |
| User wants to correct an existing transaction | Phase 2, then Phase 5 |
| User wants a reversing entry | Phase 4 |
| User mentions intercompany or multiple entities | Phase 6 |

## Phase 1 — Choose the correct GL workflow

### Step 1: Route journal-entry requests
Use a journal entry only for direct general-ledger activity: reclassification, accrual, depreciation, intercompany allocation, adjusting entry, reversing entry, or correction where the original transaction should not be modified.

Branching:
  0 matches → If the request is not GL activity, delegate to the skill that owns the transaction type instead of forcing a JE.
  1 match → Store `journalEntryPurpose`, `requestedDate`, `memo`, and whether this is `reclassification`, `accrual`, `depreciation`, `intercompany`, `correction`, or `reversal`.
  >1 matches → If the request mixes multiple purposes, separate the work by purpose before any write.

### Step 2: Route simple internal transfers
`qbMasterData(entityType="Account", query=sourceAndDestinationAccountNames)`

Branching:
  0 matches → Ask the user:
> I couldn’t find the source and destination accounts for this transfer. Which QuickBooks accounts should money move from and to?
  1 match → If both source and destination account IDs are resolved, continue with `qbFetchTransactions(transactionType="Transfer", startDate=dateMinus3Days, endDate=datePlus3Days, amount=amount)`.
  >1 matches → Ask the user:
> I found multiple possible accounts for this transfer. Which exact source account and destination account should I use?

If the user is moving money between two internal accounts, do not use a Journal Entry for a simple transfer between bank accounts; use a Transfer.

`qbFetchTransactions(transactionType="Transfer", startDate=dateMinus3Days, endDate=datePlus3Days, amount=amount)`

Branching:
  0 matches → In interactive mode, emit pre-write evidence, then call `qbTransfer(fromAccountId=sourceAccountId, toAccountId=destinationAccountId, amount=amount, date=date, memo=memo)`.
  1 match → Ask the user:
> I found an existing transfer for this date and amount. Should I skip creating a new transfer?
  >1 matches → Ask the user:
> I found multiple similar transfers. Please confirm which one, if any, this request relates to before I record anything.

Pre-write evidence:
- Entity: internal transfer, no vendor/customer
- Open-doc: no outstanding bills/invoices involved
- Account basis: source and destination accounts resolved from qbMasterData
- Mode: interactive

`qbTransfer(fromAccountId=sourceAccountId, toAccountId=destinationAccountId, amount=amount, date=date, memo=memo)`

Branching:
  0 matches → Transfer write failed; see Troubleshooting.
  1 match → Store `transferId`.
  >1 matches → Not applicable for write response.

## Phase 2 — Pre-flight existing activity

### Step 1: Fetch the original transaction being corrected
For corrections, reclasses, or reversals tied to an existing transaction, fetch and verify the source transaction before deciding whether to void, offset, reverse, or re-record.

`qbFetchTransactions(transactionType=originalTransactionType, transactionId=originalTransactionId, startDate=startDate, endDate=endDate, amount=amount, entityId=entityId)`

Branching:
  0 matches → Ask the user:
> I couldn’t find the original transaction to correct. Please provide the transaction type, date, amount, vendor/customer, or QuickBooks transaction ID.
  1 match → Store `originalTransactionId`, `originalTransactionType`, `originalDate`, `originalAmount`, `originalLines`, `originalMemo`, `originalEntityId`, and whether the period is open or closed.
  >1 matches → Ask the user:
> I found multiple possible original transactions. Which one should I correct? Please choose by date, amount, vendor/customer, and memo.

### Step 2: Check for duplicate journal entries
`qbFetchTransactions(transactionType="JournalEntry", startDate=dateMinus3Days, endDate=datePlus3Days, amount=totalDebits, memo=memo)`

Branching:
  0 matches → Continue to Phase 3.
  1 match → Ask the user:
> I found an existing journal entry with a similar date, amount, and memo. Should I skip creating a new journal entry?
  >1 matches → Ask the user:
> I found multiple similar journal entries. Please confirm whether this is a new entry or one of these already-recorded entries.

## Phase 3 — Build and record the journal entry

### Step 1: Resolve account IDs
`qbMasterData(entityType="Account", query=accountNames)`

Branching:
  0 matches → In interactive mode, ask:
> I couldn’t find one or more GL accounts for this journal entry. Which QuickBooks account names should I use for each debit and credit line?
  1 match → Store every `accountId` used in the debit and credit lines.
  >1 matches → In interactive mode, ask:
> I found multiple matching accounts. Please confirm the exact account for each debit and credit line.

Batch/async: never ask mid-batch; call `flagForReview(aiReasoning="Journal entry account resolution failed or returned multiple account candidates for: [accountNames]. CPA must confirm exact GL accounts before posting.")`.

### Step 2: Verify debits and credits
Prepare lines with exact debit and credit amounts, account IDs, class/location IDs if needed, and a descriptive `memo` detailing the purpose of the entry.

Branching:
  0 matches → If either side is missing, ask:
> A journal entry needs at least one debit and one credit. What account and amount should I use for the missing side?
  1 match → If total debits exactly equal total credits, continue.
  >1 matches → If multiple balancing alternatives exist, ask:
> There are multiple ways to balance this entry. Which accounts should receive the debits and credits?

Before `qbJournalEntry`, the JOURNAL ENTRY BALANCE ENFORCER blocks unbalanced entries.

### Step 3: Confirm or propose the entry
In interactive mode, show the proposed entry before writing: date, accounts, debits, credits, memo, and any class/location IDs.

Pre-write evidence:
- Entity: no vendor/customer unless line-level name is required; account IDs from qbMasterData
- Open-doc: original transaction `originalTransactionId` reviewed / no original transaction needed
- Account basis: account IDs explicitly resolved from qbMasterData
- Mode: interactive

Branching:
  0 matches → If the user does not approve, do not post.
  1 match → If approved, continue to Step 4.
  >1 matches → If approval is conditional or ambiguous, ask:
> Please confirm whether I should post this exact journal entry as shown.

Batch/async: never ask mid-batch; call `flagForReview(aiReasoning="Journal entry requires approval before posting because debit/credit lines affect the general ledger directly: [summary].")`.

Month-end: propose entries, do not post without approval.

### Step 4: Record the journal entry
`qbJournalEntry(date=date, lines=journalEntryLines, memo=memo)`

Branching:
  0 matches → Journal entry write failed; see Troubleshooting.
  1 match → Store `journalEntryId`.
  >1 matches → Not applicable for write response.

### Step 5: Attach support
Attach supporting schedule, approval email, source document from portal, local file, drive, or user upload when available for audit-ready books.

`qbAttachFile(entityType="JournalEntry", entityId=journalEntryId, fileSource=fileSource, fileId=fileId, note=attachmentNote)`

Branching:
  0 matches → Continue without attachment only if no support is available; otherwise ask:
> I couldn’t attach the support file. Should I retry with another file or leave the journal entry without an attachment?
  1 match → Store `attachmentId`.
  >1 matches → Ask the user:
> I found multiple possible support files. Which one should I attach to this journal entry?

## Phase 4 — Create a reversing journal entry

### Step 1: Fetch the entry to reverse
`qbFetchTransactions(transactionType="JournalEntry", transactionId=originalJournalEntryId, startDate=startDate, endDate=endDate)`

Branching:
  0 matches → Ask the user:
> I couldn’t find the original journal entry to reverse. Please provide the JE ID, date, amount, or memo.
  1 match → Store `originalJournalEntryId`, `originalDate`, `originalLines`, `originalMemo`, and `originalAmount`.
  >1 matches → Ask the user:
> I found multiple possible journal entries to reverse. Which exact entry should I reverse?

### Step 2: Build the reversal dated next period
Swap debits and credits from the original entry and date the reversal in the next period unless the user or month-end approval specifies another open-period date.

`qbMasterData(entityType="Account", query=originalAccountNames)`

Branching:
  0 matches → Ask the user:
> I couldn’t verify the account IDs from the original entry. Should I flag this for CPA review?
  1 match → Store reversal account IDs and continue.
  >1 matches → Ask the user:
> Multiple account matches were returned for the reversal. Please confirm the exact accounts before I post it.

Use memo: `Reversal of [original transaction ref] from [date]`.

### Step 3: Duplicate-check the reversal
`qbFetchTransactions(transactionType="JournalEntry", startDate=reversalDateMinus3Days, endDate=reversalDatePlus3Days, amount=originalAmount, memo=reversalMemo)`

Branching:
  0 matches → Continue to Step 4.
  1 match → Ask the user:
> I found an existing reversing journal entry for this original entry. Should I skip creating another reversal?
  >1 matches → Ask the user:
> I found multiple possible reversing entries. Please confirm whether a new reversal is still needed.

### Step 4: Record the reversal
Pre-write evidence:
- Entity: no vendor/customer unless line-level name is required; account IDs from qbMasterData
- Open-doc: original journal entry `originalJournalEntryId` applied
- Account basis: reversal built by swapping original debits and credits
- Mode: interactive

Before `qbJournalEntry`, the JOURNAL ENTRY BALANCE ENFORCER blocks unbalanced entries.

`qbJournalEntry(date=reversalDate, lines=reversalLines, memo=reversalMemo)`

Branching:
  0 matches → Reversal write failed; see Troubleshooting.
  1 match → Store `reversalJournalEntryId`.
  >1 matches → Not applicable for write response.

## Phase 5 — Correct existing transactions

### Step 1: Decide same-period void versus prior-period offset
If the error is in the current open period, void the incorrect transaction and re-record the correct transaction using the appropriate skill. If the error is in a closed period, do not modify the original transaction; record a reversing or offsetting journal entry in the current open period and record the correct current-period entry if applicable.

Branching:
  0 matches → If the period status is unknown, ask:
> Is the original transaction in an open period or a closed/prior period?
  1 match → Store `correctionMethod` as `voidAndReRecord` or `offsetJournalEntry`.
  >1 matches → If both methods appear possible, ask:
> Should I void and re-record this because it is in the current open period, or leave the original untouched and correct it in the current period?

### Step 2: Void current open-period errors
`qbFetchTransactions(transactionType=originalTransactionType, transactionId=originalTransactionId, startDate=startDate, endDate=endDate)`

Branching:
  0 matches → Ask the user:
> I couldn’t refetch the transaction to void. Please provide the exact QuickBooks transaction ID.
  1 match → Store the verified void target.
  >1 matches → Ask the user:
> I found multiple possible transactions. Which exact one should I void?

The VOID TRANSACTION GUARD requires the transaction ID being voided to appear in the most recent `qbFetchTransactions` result.

`qbVoidTransaction(transactionType=originalTransactionType, transactionId=originalTransactionId)`

Branching:
  0 matches → Void failed; see Troubleshooting.
  1 match → Store `voidedTransactionId`, then delegate re-recording to the skill that owns the correct transaction type.
  >1 matches → Not applicable for write response.

### Step 3: Offset prior-period errors
For a closed-period error, build a journal entry that exactly reverses the original accounting impact by swapping debits and credits, record it in the current open period, and add clear memo: `Correction of [original transaction ref] from [date]`.

`qbMasterData(entityType="Account", query=originalAccountNames)`

Branching:
  0 matches → In interactive mode, ask:
> I couldn’t verify the GL accounts for the correction entry. Which accounts should be debited and credited?
  1 match → Store account IDs and continue.
  >1 matches → In interactive mode, ask:
> I found multiple matching accounts for the correction. Please confirm the exact debit and credit accounts.

`qbFetchTransactions(transactionType="JournalEntry", startDate=currentPeriodDateMinus3Days, endDate=currentPeriodDatePlus3Days, amount=originalAmount, memo=correctionMemo)`

Branching:
  0 matches → Continue to correction write.
  1 match → Ask the user:
> I found an existing correction journal entry for this original transaction. Should I skip creating another one?
  >1 matches → Ask the user:
> I found multiple similar correction entries. Please confirm whether a new correction is still needed.

Pre-write evidence:
- Entity: original transaction `originalEntityId` if applicable
- Open-doc: original transaction `originalTransactionId` reviewed; closed-period original left untouched
- Account basis: correction built from original transaction lines
- Mode: interactive

Before `qbJournalEntry`, the JOURNAL ENTRY BALANCE ENFORCER blocks unbalanced entries.

`qbJournalEntry(date=currentOpenPeriodDate, lines=correctionLines, memo=correctionMemo)`

Branching:
  0 matches → Correction JE write failed; see Troubleshooting.
  1 match → Store `correctionJournalEntryId`, then record the correct entry in the current period if applicable by delegating to the appropriate skill.
  >1 matches → Not applicable for write response.

## Phase 6 — Handle intercompany and multi-entity entries

### Step 1: Identify the entity scope
Determine whether the request affects one QuickBooks company file or multiple entities. For intercompany activity, use Due To/Due From or other approved intercompany accounts rather than ordinary income or expense accounts unless the user or CPA explicitly instructs otherwise.

`qbMasterData(entityType="Account", query=intercompanyAccountNames)`

Branching:
  0 matches → In interactive mode, ask:
> I couldn’t find the intercompany Due To/Due From accounts. Which intercompany accounts should I use?
  1 match → Store `dueToAccountId`, `dueFromAccountId`, and any related clearing account IDs.
  >1 matches → In interactive mode, ask:
> I found multiple intercompany account candidates. Please confirm the exact Due To/Due From accounts to use.

Batch/async: never ask mid-batch; call `flagForReview(aiReasoning="Intercompany journal entry needs CPA confirmation because multiple or missing Due To/Due From account candidates were returned for: [intercompanyAccountNames].")`.

### Step 2: Build balanced entries for each entity
For a single QuickBooks company file, build one balanced journal entry. For multiple company files/entities, propose matching balanced entries for each entity; do not post cross-entity entries without approval.

`qbFetchTransactions(transactionType="JournalEntry", startDate=dateMinus3Days, endDate=datePlus3Days, amount=intercompanyAmount, memo=intercompanyMemo)`

Branching:
  0 matches → Continue to approval and write for the active entity only.
  1 match → Ask the user:
> I found an existing intercompany journal entry with a similar date, amount, and memo. Should I skip creating another one?
  >1 matches → Ask the user:
> I found multiple possible intercompany entries. Please confirm whether this is a new entry.

Month-end: propose entries, do not post without approval.

### Step 3: Record approved intercompany entry
Pre-write evidence:
- Entity: active QuickBooks company file; intercompany counterparty noted in memo
- Open-doc: no bill/invoice applied unless delegated to payable/receivable workflow
- Account basis: Due To/Due From accounts resolved from qbMasterData
- Mode: interactive

Before `qbJournalEntry`, the JOURNAL ENTRY BALANCE ENFORCER blocks unbalanced entries.

`qbJournalEntry(date=date, lines=intercompanyLines, memo=intercompanyMemo)`

Branching:
  0 matches → Intercompany JE write failed; see Troubleshooting.
  1 match → Store `intercompanyJournalEntryId`.
  >1 matches → Not applicable for write response.

## Troubleshooting

JOURNAL ENTRY BALANCE ENFORCER blocks `qbJournalEntry` — debits and credits do not exactly match → fix the line amounts in Phase 3 Step 2, Phase 4 Step 4, Phase 5 Step 3, or Phase 6 Step 3 before submitting.

WRITE SAFETY GUARD blocks `qbJournalEntry` or `qbTransfer` — `qbMasterData` or `qbFetchTransactions` was not called earlier in the conversation → run Phase 2 Step 2 and Phase 3 Step 1 for journal entries, or Phase 1 Step 2 for transfers.

VENDOR/ACCOUNT RESOLUTION GUARD blocks `qbJournalEntry` — one or more account IDs were not returned by the most recent `qbMasterData` lookup → re-run Phase 3 Step 1, Phase 4 Step 2, Phase 5 Step 3, or Phase 6 Step 1.

VOID TRANSACTION GUARD blocks `qbVoidTransaction` — the void target was not fetched or does not appear in the most recent `qbFetchTransactions` result → re-run Phase 5 Step 2 and verify the transaction ID before voiding.

CURRENCY GUARD blocks `qbJournalEntry` — transaction currency differs from the source account currency and no exchange rate or currency conversion information is present → add `exchangeRate` or use explicit currency conversion lines before returning to the relevant write step.

`qbFetchTransactions` returns too many matches — search criteria are too broad → narrow by `transactionType`, `transactionId`, `entityId`, `startDate`, `endDate`, `amount`, or `memo`, then repeat the relevant fetch step.

`qbAttachFile` fails — file source, file ID, or journal entry ID is missing or invalid → verify `journalEntryId`, `fileSource`, and `fileId`, then retry Phase 3 Step 5.