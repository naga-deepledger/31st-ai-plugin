---
name: client-onboarding
description: Bootstrap a new or re-run QuickBooks Online client by verifying the environment, pulling 12 months of transaction history, building vendor and customer account baselines, and seeding agent memory after CPA approval. Activate when the user mentions onboarding, bootstrapping, new client setup, learning from history, or re-running onboarding analysis.
---

## Routing

| Situation | Start at |
|-----------|----------|
| New client or explicit re-run request | Step 1 |
| Bootstrap already completed, no re-run | Stop — report existing bootstrap date/counts |
| Re-run requested on completed bootstrap | Confirm before overwriting |

## Step 1: Check bootstrap status + verify required accounts

`agentMemory(operation="read", type="bootstrap_status", key=realmId)`

- `cpaReviewed=true` and no re-run request → stop and report existing bootstrap date/counts
- Re-run requested → confirm before replacing CPA-approved mappings
- Not found → proceed

`qbMasterData(entityType="Account")` — verify these 4 required accounts exist with exactly one active match each: **Accounts Payable, Accounts Receivable, Undeposited Funds, Retained Earnings**. Missing or duplicate → `flagForReview`.

## Step 2: Fetch 12-month transaction history

`lookbackStartDate = today − 12 months`; run one `qbFetchTransactions` per type:

`Expense`, `Bill`, `BillPayment`, `Invoice`, `SalesReceipt`, `ReceivePayment`, `Deposit`, `JournalEntry`, `Transfer`

0 results for all types → ask to expand window or stop (batch: `flagForReview`).

## Step 3: Build profiles (no tool call — compute from history)

**Vendor expense profiles** (one per `vendorId`):
- Dominant expense account + full account distribution
- Transaction types used (Expense / Bill / BillPayment), frequency, amount range (min / max / avg)
- Mark low-frequency (1-2 occurrences) — not eligible for autonomous inference

**Customer income profiles** (one per `customerId`):
- Dominant income account + full account distribution
- Transaction types, frequency, amount range

**5-criteria autonomous eligibility** — profile is eligible for autonomous inference only if ALL pass:
(a) ≥3 prior transactions in last 365 days
(b) dominant account ≥70%
(c) no second account ≥20%
(d) amounts within 5× median of dominant-account transactions
(e) most recent dominant-account transaction <180 days old

Profiles failing any criterion → mark `autonomousEligible=false`; downstream skills will `flagForReview` for those entities (CONSISTENCY RULE GUARD).

**Recurring JE patterns** — identify repeated description/line structure (depreciation, accruals); always `autonomousEligible=false` — propose only, never auto-post.

**Transfer routes** — common source → destination account paths.

## Step 4: Present summary and get CPA approval

Present to CPA:
- Required account IDs (AP, AR, Undeposited Funds, Retained Earnings)
- 12-month transaction counts by type
- Vendor mappings (eligible and ineligible, with reasons)
- Customer mappings (eligible and ineligible)
- Recurring JE patterns
- Transfer routes

Ask: "Reply 'approve' to activate these mappings, or tell me which to edit or exclude."

Batch/async: never ask mid-run → `flagForReview(aiReasoning="CPA approval required before activating bootstrap-derived mappings for realm [realmId]; batch/async cannot ask mid-run.")`

## Step 5: Seed agent memory (after CPA approval only)

`cappedUpvotes = min(frequency, 5)` — prevents over-confidence in bootstrap data.

```
agentMemory(operation="write", type="vendor_mapping", key=vendorId, value={
  vendorName, expenseAccountId, transactionType, frequency,
  amountRange: {min, max, avg}, upvotes=cappedUpvotes,
  source="bootstrap", autonomousEligible
})

agentMemory(operation="write", type="customer_mapping", key=customerId, value={
  customerName, incomeAccountId, transactionType, frequency,
  amountRange: {min, max, avg}, upvotes=cappedUpvotes,
  source="bootstrap", autonomousEligible
})

agentMemory(operation="write", type="recurring_journal_pattern", key=patternId, value={
  descriptionPattern, lineStructure, frequency,
  amountRange: {min, max, avg}, upvotes=cappedUpvotes,
  source="bootstrap", autonomousEligible=false
})

agentMemory(operation="write", type="transfer_route", key=routeId, value={
  sourceAccountId, destinationAccountId, frequency,
  amountRange: {min, max, avg}, upvotes=cappedUpvotes,
  source="bootstrap", autonomousEligible
})

agentMemory(operation="write", type="bootstrap_status", key=realmId, value={
  date: today, counts: txnCountsByType, cpaReviewed: true
})
```

## Autonomous operation ready when:

- Required account IDs stored ✓
- Transaction counts by type stored ✓
- CPA-approved vendor/customer mappings seeded ✓
- Ineligible profiles documented (downstream skills will `flagForReview` for those entities) ✓
- Recurring JE patterns stored as `autonomousEligible=false` ✓
- Transfer routes stored ✓
- `amountRange` stored per entity (downstream: amount >3× stored range → MATERIALITY GUARD flags) ✓
- `bootstrap_status.cpaReviewed=true` ✓

## Gotchas

1. **`cappedUpvotes=5` max** — prevents bootstrap history from over-weighting historical frequency in autonomous decisions
2. **Recurring JE patterns always `autonomousEligible=false`** — propose at month-end, never auto-post
3. **Ineligible vendor/customer profiles**: don't delete them — they inform the human-in-the-loop path; downstream skills use `flagForReview` for those entities
4. **`amountRange` stored during onboarding enables MATERIALITY GUARD** — transactions >3× the stored range get flagged automatically by downstream skills
5. **Re-run replaces CPA-approved mappings** — confirm explicitly before overwriting; partial re-run (one vendor only) requires targeted `agentMemory` writes
