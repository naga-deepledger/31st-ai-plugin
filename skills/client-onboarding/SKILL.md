---
name: client-onboarding
description: Bootstrap a newly connected or re-run QuickBooks Online client by checking the environment, pulling 12 months of history, profiling vendor/customer/account patterns, and seeding reviewed agent memory. Use when the user mentions onboarding, bootstrapping, setting up a new client, learning from history, or re-running onboarding analysis.
---

| Situation | Phase |
|---|---|
| New client or re-run request | Phase 1 |
| Need historical learning data | Phase 2 |
| Need vendor/customer/account baselines | Phase 3 |
| Ready for CPA approval and activation | Phase 4 |

## Phase 1 — Check QuickBooks environment

### Step 1: Read bootstrap status

`agentMemory(operation="read", type="bootstrap_status", key=realmId)`

Branching:
  0 matches → Proceed to Step 2 and treat this as first-time setup.
  1 match → If `cpaReviewed=true` and the user did not specifically ask to re-run, stop and report the existing bootstrap date/counts. If interactive and the user asked to re-run, ask:
> Bootstrap was already completed on {bootstrapDate} with {bootstrapCounts}. Do you want me to re-run onboarding and replace or update the bootstrap-derived memory?
  >1 matches → `flagForReview(aiReasoning="Multiple bootstrap_status records found for realm {realmId}; cannot determine which onboarding baseline is active without CPA review.")`

### Step 2: Verify required QuickBooks accounts

`qbMasterData(entityType="Account")`

Required accounts that must exist before autonomous bookkeeping: Accounts Payable (AP), Accounts Receivable (AR), Undeposited Funds, Retained Earnings.

Branching:
  0 matches → `flagForReview(aiReasoning="QuickBooks account master data returned no accounts during onboarding; cannot verify AP, AR, Undeposited Funds, or Retained Earnings before seeding memory.")`
  1 match → If the single result is not the complete account list, `flagForReview(aiReasoning="Account master data lookup returned only one account; cannot verify required onboarding accounts AP, AR, Undeposited Funds, and Retained Earnings.")`
  >1 matches → Store account IDs for AP, AR, Undeposited Funds, and Retained Earnings if each required account has exactly one active match; if any required account is missing or duplicated, `flagForReview(aiReasoning="Required onboarding account check failed for realm {realmId}: missing or duplicate required account(s): {missingOrDuplicateAccounts}.")`

## Phase 2 — Pull 12-month QuickBooks history

### Step 1: Fetch transaction history by type

Use `lookbackStartDate=today-12months` and `lookbackEndDate=today` unless the user provides a configurable lookback period.

`qbFetchTransactions(transactionType="Expense", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="Bill", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="BillPayment", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="Invoice", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="SalesReceipt", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="ReceivePayment", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="Deposit", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="JournalEntry", startDate=lookbackStartDate, endDate=lookbackEndDate)`

`qbFetchTransactions(transactionType="Transfer", startDate=lookbackStartDate, endDate=lookbackEndDate)`

Branching:
  0 matches → In interactive mode ask:
> I found no transactions in the selected lookback period. Should I expand the lookback window or stop onboarding for now?
Batch/async → `flagForReview(aiReasoning="No transactions found for onboarding lookback {lookbackStartDate} to {lookbackEndDate}; cannot build vendor, customer, recurring, or transfer baselines.")`
  1 match → Store the single transaction in `rawTxnSet` and continue, but mark the client as having insufficient history for autonomous mappings.
  >1 matches → Store all returned transactions in `rawTxnSet` and continue.

### Step 2: Extract historical patterns

No tool call; compute from `rawTxnSet`.

Extract:
- Top vendors by count and total amount.
- Vendor mappings: vendor → expense account, transaction type, frequency, amount range.
- Customer mappings: customer → income account, transaction type, frequency, amount range.
- Account distribution by vendor/customer and transaction type.
- Recurring amounts, including repeated JournalEntries for depreciation, accruals, and other month-end patterns.
- Transfer routes: common account-to-account transfer paths.
- BillPayment and ReceivePayment behavior to understand open-document workflows without resolving them during onboarding.

Branching:
  0 matches → `flagForReview(aiReasoning="Historical transactions were fetched but no usable vendor, customer, account, recurring, or transfer patterns could be extracted for onboarding.")`
  1 match → Mark the extracted pattern as low-confidence and continue to CPA presentation.
  >1 matches → Continue to Phase 3 with extracted pattern tables.

## Phase 3 — Profile vendors, customers, recurring entries, and transfers

### Step 1: Build vendor expense profiles

No tool call; compute one profile per vendor from Expense, Bill, and BillPayment history.

For each vendor profile, store:
- `vendorId`
- vendor name
- dominant expense account candidate
- all observed expense accounts and account distribution
- transaction types used: Expense, Bill, BillPayment
- frequency
- amount range: min, max, avg
- typical memo/description patterns
- whether the vendor is low-frequency with 1-2 occurrences

Branching:
  0 matches → Continue without vendor mappings and note that autonomous vendor categorization is not ready.
  1 match → Create one vendor profile and mark confidence according to its account distribution.
  >1 matches → Create vendor profiles for all vendors; do not merge vendors with similar names unless they share the same QuickBooks `vendorId` (MULTI-VENDOR AMBIGUITY GUARD blocks downstream writes if multiple candidate vendors are picked without resolution).

### Step 2: Build customer income profiles

No tool call; compute one profile per customer from Invoice, SalesReceipt, ReceivePayment, and Deposit history.

For each customer profile, store:
- `customerId`
- customer name
- dominant income account candidate
- all observed income accounts and account distribution
- transaction types used: Invoice, SalesReceipt, ReceivePayment, Deposit
- frequency
- amount range: min, max, avg
- typical memo/description patterns
- whether the customer is low-frequency with 1-2 occurrences

Branching:
  0 matches → Continue without customer mappings and note that autonomous customer income classification is not ready.
  1 match → Create one customer profile and mark confidence according to its account distribution.
  >1 matches → Create customer profiles for all customers; do not merge customers with similar names unless they share the same QuickBooks `customerId` (MULTI-VENDOR AMBIGUITY GUARD blocks downstream writes if multiple candidate customers are picked without resolution).

### Step 3: Apply account-inference consistency criteria

No tool call; evaluate each proposed history-inferred account baseline.

A vendor/customer account baseline is eligible for autonomous use only when all 5 criteria pass:
  (a) ≥ 3 prior transactions in the last 365 days,
  (b) dominant account ≥ 70% of those transactions,
  (c) no second account ≥ 20% of those transactions,
  (d) current amount within 5× the median of dominant-account transactions at transaction time,
  (e) most recent dominant-account transaction < 180 days old.

Branching:
  0 matches → Mark all account baselines as CPA-review-only; downstream transaction skills must call `flagForReview` when no eligible baseline exists.
  1 match → Store the one eligible baseline with its supporting transaction count, dominant percentage, second-account percentage, median, and most recent date.
  >1 matches → Store all eligible baselines with their supporting transaction count, dominant percentage, second-account percentage, median, and most recent date. If any criterion fails for a profile, keep the profile for reference but mark it not autonomous (CONSISTENCY RULE GUARD blocks downstream writes when a history-inferred account has not passed these criteria).

### Step 4: Identify recurring JournalEntries and transfer routes

No tool call; compute from JournalEntry and Transfer history.

For recurring JournalEntries, identify repeated descriptions, repeated line structures, and consistent amounts for depreciation, accruals, and other month-end patterns. For transfer routes, identify common source account → destination account paths and frequency.

If a user asks to create or recreate month-end entries during onboarding, delegate to the month-end workflow; propose entries only and do not post without approval.

Branching:
  0 matches → Continue without recurring JournalEntry or transfer-route memory.
  1 match → Store the single recurring or transfer pattern as CPA-review-only until approved.
  >1 matches → Summarize all recurring JournalEntry patterns and transfer routes for CPA confirmation. In interactive mode, if posting is requested, ask:
> I can propose month-end journal entries based on recurring history, but I will not post them without CPA approval. Should I prepare a proposal instead of posting entries now?

## Phase 4 — Confirm handoff and seed agent memory

### Step 1: Present onboarding summary to CPA

No tool call; present summary tables for:
- Required environment accounts and IDs: AP, AR, Undeposited Funds, Retained Earnings.
- 12-month lookback period and counts by transaction type: Expense, Bill, BillPayment, Invoice, SalesReceipt, ReceivePayment, Deposit, JournalEntry, Transfer.
- Learned vendor mappings.
- Learned customer mappings.
- Low-frequency vendors and customers with 1-2 occurrences as potential one-offs.
- Recurring JournalEntry patterns.
- Transfer routes.
- Profiles not eligible for autonomous use and why.

Branching:
  0 matches → If no mappings are ready, interactive mode ask:
> I did not find enough reliable history to activate autonomous mappings. Should I save the environment check only, or stop without seeding onboarding memory?
Batch/async → `flagForReview(aiReasoning="Onboarding summary has no reliable mappings to activate; CPA must decide whether to save environment-only bootstrap status or rerun with a longer lookback.")`
  1 match → Present the one mapping/pattern and require CPA confirmation before activation.
  >1 matches → Present all mappings/patterns and require CPA confirmation before activation.

### Step 2: Get CPA approval before activation

No tool call.

Interactive mode prompt:
> Please review the onboarding summary. Reply “approve” to activate these bootstrap-derived mappings, or tell me which vendors, customers, recurring entries, or transfer routes to edit or exclude before I seed memory.

Batch/async mode:
`flagForReview(aiReasoning="CPA approval is required before activating bootstrap-derived onboarding mappings for realm {realmId}; batch/async mode cannot ask mid-run.")`

Branching:
  0 matches → Do not seed memory.
  1 match → If CPA approved, proceed to Step 3; if edits were supplied, apply edits and re-present the changed summary once.
  >1 matches → If approval is ambiguous or multiple reviewers conflict, `flagForReview(aiReasoning="Conflicting or ambiguous CPA onboarding approvals received for realm {realmId}; cannot activate bootstrap-derived memory safely.")`

### Step 3: Seed reviewed mappings to agent memory

Before each write in interactive mode, emit:

Pre-write evidence:
- Entity: {entityName} + {entityId}
- Open-doc: no outstanding bills/invoices applied during onboarding memory seed
- Account basis: {supportingTxnCount} txns, {dominantAccountPercent}% dominant
- Mode: interactive

`agentMemory(operation="write", type="vendor_mapping", key=vendorId, value={vendorName: vendorName, expenseAccountId: dominantExpenseAccountId, transactionType: transactionType, frequency: frequency, amountRange: {min: minAmount, max: maxAmount, avg: avgAmount}, upvotes: cappedUpvotes, source: "bootstrap", autonomousEligible: autonomousEligible})`

`agentMemory(operation="write", type="customer_mapping", key=customerId, value={customerName: customerName, incomeAccountId: dominantIncomeAccountId, transactionType: transactionType, frequency: frequency, amountRange: {min: minAmount, max: maxAmount, avg: avgAmount}, upvotes: cappedUpvotes, source: "bootstrap", autonomousEligible: autonomousEligible})`

`agentMemory(operation="write", type="recurring_journal_pattern", key=recurringPatternId, value={descriptionPattern: descriptionPattern, lineStructure: lineStructure, frequency: frequency, amountRange: {min: minAmount, max: maxAmount, avg: avgAmount}, upvotes: cappedUpvotes, source: "bootstrap", autonomousEligible: false})`

`agentMemory(operation="write", type="transfer_route", key=transferRouteId, value={sourceAccountId: sourceAccountId, destinationAccountId: destinationAccountId, frequency: frequency, amountRange: {min: minAmount, max: maxAmount, avg: avgAmount}, upvotes: cappedUpvotes, source: "bootstrap", autonomousEligible: autonomousEligible})`

Set `cappedUpvotes` to no more than 5 regardless of historical frequency.

Branching:
  0 matches → If there are no CPA-approved mappings, skip memory seeding and proceed to Step 4 only if CPA approved environment-only completion.
  1 match → Write the single approved mapping with `source: "bootstrap"`.
  >1 matches → Write each approved mapping with `source: "bootstrap"`; if any individual write fails, continue only for unrelated mappings and `flagForReview(aiReasoning="Onboarding memory seed partially failed for realm {realmId}; failed mapping keys: {failedKeys}. Successful mappings were written with source bootstrap.")`

### Step 4: Mark onboarding complete and define handoff readiness

Before the status write in interactive mode, emit:

Pre-write evidence:
- Entity: {clientName} + {realmId}
- Open-doc: no outstanding bills/invoices applied during onboarding memory seed
- Account basis: {totalTxnCount} txns, bootstrap CPA approval
- Mode: interactive

`agentMemory(operation="write", type="bootstrap_status", key=realmId, value={date: today, counts: bootstrapCounts, cpaReviewed: true})`

The agent is ready for autonomous operation only after it has learned:
- Required account IDs for AP, AR, Undeposited Funds, and Retained Earnings.
- Transaction counts by type for the lookback period.
- CPA-approved vendor mappings and which vendors remain review-only.
- CPA-approved customer mappings and which customers remain review-only.
- Recurring JournalEntry patterns to propose for month-end, not post without approval.
- Transfer routes.
- Bootstrap-derived `amountRange` min/max/avg for anomaly detection; new transactions that are 3x outside the stored range must be flagged for review by the downstream transaction workflow.
- `bootstrap_status` with `cpaReviewed: true`.

Branching:
  0 matches → If no status can be written, `flagForReview(aiReasoning="Onboarding mappings may have been seeded but bootstrap_status could not be written for realm {realmId}; CPA must verify activation state before autonomous use.")`
  1 match → Report onboarding complete with counts and `cpaReviewed: true`.
  >1 matches → `flagForReview(aiReasoning="Multiple bootstrap_status writes or records detected for realm {realmId}; CPA must verify which onboarding status is authoritative.")`

## Troubleshooting

FLAG FOR REVIEW QUALITY GUARD blocks `flagForReview` — `aiReasoning` is missing, too short, or generic → include the specific failed account, transaction type, entity, amount, date range, or duplicate status and retry the recovery action from Phase 1 Step 2, Phase 2 Step 1, Phase 4 Step 2, or Phase 4 Step 4.

CONSISTENCY RULE GUARD blocks downstream transaction writes — a history-inferred account from onboarding was not evaluated against the 5 criteria or failed one of them → return to Phase 3 Step 3, store the profile as not autonomous, and have the downstream skill call `flagForReview` for that transaction.

MULTI-VENDOR AMBIGUITY GUARD blocks downstream transaction writes — multiple vendors or customers match a name and onboarding did not resolve the exact QuickBooks entity ID → return to Phase 3 Step 1 or Phase 3 Step 2 and keep separate profiles by `vendorId` or `customerId`; if unresolved, `flagForReview` with the candidate list.

MATERIALITY GUARD blocks downstream transaction writes — a current transaction amount is more than 5× the median history fetched for that vendor/customer → use the onboarding `amountRange` and transaction history from Phase 3 Step 1 or Phase 3 Step 2, then `flagForReview` with the amount, median, and historical range.

`qbFetchTransactions` timeout or rate limit — the 12-month pull is too large for one request by transaction type → rerun Phase 2 Step 1 in smaller date windows while preserving the same transaction types and final lookback coverage.

`agentMemory` duplicate `bootstrap_status` — prior onboarding was already completed or multiple status records exist → rerun Phase 1 Step 1 and ask the interactive re-run prompt, or `flagForReview` if batch/async mode cannot confirm replacement.