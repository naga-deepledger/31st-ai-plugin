---
name: master-data
description: Look up, create, update, or deactivate QuickBooks chart-of-accounts entries, vendors, customers, items, and classes. Activate when the user mentions adding or updating a vendor, customer, account, service item, inventory item, class, 1099 setup, tax ID, payment terms, chart of accounts, or entity IDs.
---

## Entity routing

| Entity | Create call | Key constraint |
|--------|------------|----------------|
| Account | `qbMasterData(operation="create", entityType="account", name, accountType, ...)` | `accountType` is permanent — cannot change after create |
| Vendor | `qbMasterData(operation="create", entityType="vendor", name, ...)` | `taxIdentifier` (EIN/SSN) required if `vendor1099=true` |
| Customer | `qbMasterData(operation="create", entityType="customer", name, ...)` | — |
| Item – Service / NonInventory | `qbMasterData(operation="create", entityType="item", itemType="Service"\|"NonInventory", incomeAccountId, ...)` | `incomeAccountId` required |
| Item – Inventory | `qbMasterData(operation="create", entityType="item", itemType="Inventory", assetAccountId, ...)` | `assetAccountId` required; QB auto-sets `TrackQtyOnHand=true`, `QtyOnHand=0` |
| Class | `qbMasterData(operation="create", entityType="class", name, parentClassId?, ...)` | `parentClassId` optional for sub-class |
| Tax rate | Read only — managed in QB Tax Center | Never attempt create/update |

## Before every create: duplicate check

`qbMasterData(entityTypes=[entityType], filter=entityName)`

- 0 matches → proceed to create
- 1 match → ask: "I found existing [entity] [name] (ID [id]). Use this one instead of creating a new one?"
- >1 matches → ask user to choose; never pick one (MULTI-VENDOR AMBIGUITY GUARD blocks downstream writes)

In batch/async with >1 matches: `flagForReview(aiReasoning="Duplicate check before [entity] create returned multiple matches for [name]: [candidate list]. Cannot safely create or select without confirmation.")`

## Resolve (lookup only)

`qbMasterData(entityTypes=["account","vendor","customer","item","class"], filter=name)`

- 1 match → store ID; delegate transaction posting to AP/AR/JE skill
- >1 matches (vendor/customer) → interactive: ask user to choose; batch: `flagForReview` with candidate list

## Account create

Confirm `accountType` before creating — it cannot be changed afterward. Ask if uncertain about direction (Income vs Expense). Confirm `parentAccountId` for sub-accounts.

After create: verify with `qbMasterData(filter=accountName)` before using ID in any transaction.

## Vendor create

Collect: display name, address, email, phone, payment terms. Ask: "Is this vendor an independent contractor who should receive a 1099?" If yes, collect `taxIdentifier` (EIN or SSN) and set `vendor1099=true`.

## Item create

- Service or NonInventory → resolve `incomeAccountId` via `qbMasterData(entityTypes=["account"], filter=accountName)` first
- Inventory → resolve `assetAccountId` first; QB handles qty tracking automatically

## Update / deactivate

1. `qbMasterData(detailedInfo=entityType, filter=entityName)` → store `id` and fresh `syncToken`
2. `syncToken` is required for all updates — always fetch fresh; stale token = concurrency error
3. For deactivation: check outstanding docs first
   - Vendor: `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`
   - Customer: `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`
   - Outstanding items exist → warn; QB hides record but retains all history
4. `qbMasterData(operation="update", entityType, id, syncToken, changedFields)` or `active=false` for deactivation

## Gotchas

1. **`accountType` is permanent** — wrong type = delete and recreate; confirm before creating
2. **`syncToken` required for updates** — always fetch the current record first; never use a cached syncToken
3. **Tax rates are read-only** — direct user to QB Tax Center; don't attempt create/update
4. **Deactivation hides but retains** — outstanding bills/invoices remain in AP/AR aging; resolve them first
5. **Verify after create** — QB propagation delay; lookup the ID before passing it to a transaction tool
