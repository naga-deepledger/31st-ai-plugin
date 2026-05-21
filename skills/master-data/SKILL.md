---
name: master-data
description: Manage QuickBooks master data by looking up, creating, updating, or deactivating accounts, vendors, customers, items, classes, and reading tax rates. Activate when the user mentions chart of accounts, adding/updating a vendor or customer, creating service/non-inventory/inventory items, tracking classes, 1099 setup, tax IDs, payment terms, or reviewing entity IDs/details.
---

| Situation | Phase |
|---|---|
| Need valid IDs, details, tax rates, or disambiguation before a transaction | Phase 1 |
| Create an account in the Chart of Accounts | Phase 2 |
| Create a vendor, customer, item, or class | Phases 3–6 |
| Update, rename, merge, deactivate, or soft-delete an existing record | Phase 7 |
| Need to post a bill, expense, invoice, payment, receipt, deposit, or journal entry after resolving IDs | Delegate to the relevant transaction skill after Phase 1 |

## Phase 1 — Resolve master data

### Step 1: Fetch entity IDs

`qbMasterData(entityTypes=["account","vendor","customer","item","class"], filter=entityName)`

Branching:
  0 matches → If the user asked to create the entity, proceed to the entity-specific create phase. If the user asked to use an existing entity, ask:
  > I couldn’t find “[entityName]” in QuickBooks. Should I create it, or should I search for a different name?
  1 match → Store `entityType`, `id`, `name`, and any returned account/customer/vendor/item/class metadata for the current workflow.
  >1 matches → For vendor/customer only, do not choose one. Interactive mode: present the full candidate list and ask:
  > I found multiple matching QuickBooks entities for “[entityName]”: [candidate list with names and IDs]. Which one should I use?
  Batch/async mode: `flagForReview(aiReasoning="Multiple possible QuickBooks vendor/customer matches for [entityName]: [candidate list]. Cannot safely select one because the wrong entity corrupts AP/AR reports, audit trail, and historical account precedent.")` (FLAG FOR REVIEW QUALITY GUARD requires specific `aiReasoning`).

### Step 2: Fetch vendor or customer details

`qbMasterData(detailedInfo="vendor", filter=vendorName)`

`qbMasterData(detailedInfo="customer", filter=customerName)`

Branching:
  0 matches → Ask:
  > I couldn’t find “[name]” in QuickBooks. Should I create this vendor/customer, or search for a different name?
  1 match → Review and store `id`, `syncToken` if returned, address, payment terms, 1099 eligibility, tax identifier, email, and phone. Use this for verifying 1099 setup, confirming billing address, and checking payment terms before entering a bill or customer transaction.
  >1 matches → Interactive mode: ask:
  > I found multiple matching vendors/customers for “[name]”: [candidate list with names and IDs]. Which exact entity should I use?
  Batch/async mode: `flagForReview(aiReasoning="Multiple matching vendors/customers returned while fetching details for [name]: [candidate list]. Need CPA/user confirmation before using contact, payment terms, 1099, or tax ID details.")` (FLAG FOR REVIEW QUALITY GUARD requires specific `aiReasoning`).

### Step 3: Read tax rates

`qbMasterData(entityTypes=["taxRate"], filter=taxRateName)`

Branching:
  0 matches → Tell the user:
  > I couldn’t find that tax rate in QuickBooks. Tax rates are read-only from this workflow and are managed inside the QuickBooks Tax Center.
  1 match → Store the tax rate ID/details for use by the relevant transaction skill.
  >1 matches → Ask:
  > I found multiple tax rates matching “[taxRateName]”: [candidate list]. Which one should I use?
  Create/update requested → Do not call create/update. Tell the user:
  > Tax rates are read-only here. QuickBooks tax rates are managed inside the QB Tax Center, not created or updated through this master-data workflow.

### Step 4: Delegate transaction posting

Use this phase only to resolve valid IDs and details. For bills, expenses, bill payments, invoices, sales receipts, receive payments, deposits, refunds, credits, estimates, purchase orders, recurring transactions, transfers, or journal entries, delegate to the appropriate transaction skill after IDs are resolved.

Branching:
  Missing ID → Return to Phase 1 Step 1.
  One verified ID → Hand off the verified `entityType`, `id`, and relevant metadata to the transaction skill.
  Multiple possible IDs → Return to Phase 1 Step 1 and resolve ambiguity first.

## Phase 2 — Manage accounts

### Step 1: Check for duplicate account before create

`qbMasterData(entityTypes=["account"], filter=accountName)`

Branching:
  0 matches → Proceed to Phase 2 Step 2.
  1 match → Ask:
  > I found an existing account “[accountName]” with ID [accountId]. Do you want to use this existing account instead of creating a new one?
  >1 matches → Ask:
  > I found multiple matching accounts: [candidate list with names, account types, account numbers, and IDs]. Which one should I use, or should I create a new account with a different name?

### Step 2: Determine account type and confirm permanence

Confirm the correct `accountType`, `name`, `accountNumber` if any, and `parentAccountId` if this is a sub-account. Confirm the direction of money flow so an `Income` account is not created when the user needs an `Expense` account, or vice versa.

`qbMasterData(entityTypes=["account"], filter=parentAccountName)`

Branching:
  No parent account needed → Proceed to confirmation.
  0 parent matches → Ask:
  > I couldn’t find the parent account “[parentAccountName]”. Should this be a top-level account, or should I search for a different parent account?
  1 parent match → Store `parentAccountId` and proceed.
  >1 parent matches → Ask:
  > I found multiple possible parent accounts for “[parentAccountName]”: [candidate list]. Which parent account should I use?
  Missing or uncertain `accountType` → Ask:
  > What QuickBooks account type should this be? Account type is permanent and cannot be changed after the account is created.
  Month-end mode → Propose the account setup and do not create it until the user approves.

### Step 3: Create account

In interactive mode, emit:

Pre-write evidence:
- Entity: new account “[accountName]” (ID pending)
- Open-doc: not applicable for account create
- Account basis: duplicate check completed in Phase 2 Step 1; `accountType` user-confirmed
- Mode: interactive

`qbMasterData(operation="create", entityType="account", name=accountName, accountType=accountType, accountNumber=accountNumber, parentAccountId=parentAccountId)`

Branching:
  Success → Store returned account ID. If the account will be used frequently, save it to `agentMemory`.
  Duplicate/name exists error → Return to Phase 2 Step 1 and use the existing account or choose a different name.
  Missing required field → Return to Phase 2 Step 2 and collect `accountType`, `name`, `accountNumber`, or `parentAccountId` as needed.

### Step 4: Verify account creation

`qbMasterData(entityTypes=["account"], filter=accountName)`

Branching:
  0 matches → Tell the user:
  > QuickBooks did not return the new account in lookup yet. I’ll need to verify before using it in a transaction.
  1 match → Confirm the new account ID and account type to the user.
  >1 matches → Ask:
  > I found multiple accounts after creation: [candidate list]. Which one is the newly created account?

## Phase 3 — Manage vendors

### Step 1: Check for duplicate vendor before create

`qbMasterData(entityTypes=["vendor"], filter=vendorName)`

Branching:
  0 matches → Proceed to Phase 3 Step 2.
  1 match → Ask:
  > I found an existing vendor “[vendorName]” with ID [vendorId]. Do you want to use this existing vendor instead of creating a new one?
  >1 matches → Interactive mode: ask:
  > I found multiple matching vendors: [candidate list with names and IDs]. Which one should I use, or should I create a new vendor with a different display name?
  Batch/async mode: `flagForReview(aiReasoning="Duplicate check before vendor create returned multiple possible vendors for [vendorName]: [candidate list]. Cannot safely create or select a vendor without confirmation.")` (FLAG FOR REVIEW QUALITY GUARD requires specific `aiReasoning`).

### Step 2: Gather vendor details and 1099 information

Collect name as the QuickBooks display name. Optional fields: address, email, phone, and payment terms. Ask whether the vendor is an independent contractor who should receive a 1099; if yes, collect `taxIdentifier` as EIN or SSN and set `vendor1099=true`.

Branching:
  Missing display name → Ask:
  > What display name should QuickBooks use for this vendor?
  1099 status unknown → Ask:
  > Is this vendor an independent contractor who should receive a 1099? If yes, please provide the EIN or SSN for the vendor tax identifier.
  1099 yes but missing tax ID → Ask:
  > Please provide the vendor’s EIN or SSN so I can set `vendor1099=true` with the required tax identifier.
  All required information present → Proceed to Phase 3 Step 3.

### Step 3: Confirm and create vendor

Show: name, address, email, phone, payment terms, 1099 status, and masked tax ID.

In interactive mode, emit:

Pre-write evidence:
- Entity: new vendor “[vendorName]” (ID pending)
- Open-doc: not applicable for vendor create
- Account basis: duplicate check completed in Phase 3 Step 1
- Mode: interactive

`qbMasterData(operation="create", entityType="vendor", name=vendorName, address=address, vendor1099=vendor1099, taxIdentifier=taxIdentifier)`

Branching:
  Success → Store returned vendor ID.
  Duplicate/name exists error → Return to Phase 3 Step 1 and use the existing vendor or choose a different display name.
  User rejects confirmation → Ask what field should change, then repeat Phase 3 Step 3.
  Month-end mode → Propose the vendor creation and do not create until approved.

## Phase 4 — Manage customers

### Step 1: Check for duplicate customer before create

`qbMasterData(entityTypes=["customer"], filter=customerName)`

Branching:
  0 matches → Proceed to Phase 4 Step 2.
  1 match → Ask:
  > I found an existing customer “[customerName]” with ID [customerId]. Do you want to use this existing customer instead of creating a new one?
  >1 matches → Interactive mode: ask:
  > I found multiple matching customers: [candidate list with names and IDs]. Which one should I use, or should I create a new customer with a different display name?
  Batch/async mode: `flagForReview(aiReasoning="Duplicate check before customer create returned multiple possible customers for [customerName]: [candidate list]. Cannot safely create or select a customer without confirmation.")` (FLAG FOR REVIEW QUALITY GUARD requires specific `aiReasoning`).

### Step 2: Gather customer details

Collect name as the QuickBooks display name. Optional fields: billing address, email, phone, and payment terms.

Branching:
  Missing display name → Ask:
  > What display name should QuickBooks use for this customer?
  Optional details missing → Continue if the user confirms they are not needed now.
  All required information present → Proceed to Phase 4 Step 3.

### Step 3: Confirm and create customer

Show: name, billing address, email, phone, and payment terms.

In interactive mode, emit:

Pre-write evidence:
- Entity: new customer “[customerName]” (ID pending)
- Open-doc: not applicable for customer create
- Account basis: duplicate check completed in Phase 4 Step 1
- Mode: interactive

`qbMasterData(operation="create", entityType="customer", name=customerName, address=address)`

Branching:
  Success → Store returned customer ID.
  Duplicate/name exists error → Return to Phase 4 Step 1 and use the existing customer or choose a different display name.
  User rejects confirmation → Ask what field should change, then repeat Phase 4 Step 3.
  Month-end mode → Propose the customer creation and do not create until approved.

## Phase 5 — Manage items

### Step 1: Check for duplicate item before create

`qbMasterData(entityTypes=["item"], filter=itemName)`

Branching:
  0 matches → Proceed to Phase 5 Step 2.
  1 match → Ask:
  > I found an existing item “[itemName]” with ID [itemId]. Do you want to use this existing item instead of creating a new one?
  >1 matches → Ask:
  > I found multiple matching items: [candidate list with names, item types, and IDs]. Which one should I use, or should I create a new item with a different name?

### Step 2: Determine item type and required accounts

Classify `itemType`:
- `Service` — labor, consulting, subscription fees; requires `incomeAccountId`
- `NonInventory` — physical goods not tracked by quantity; requires `incomeAccountId`
- `Inventory` — physical goods with quantity tracking; requires `assetAccountId`; QuickBooks auto-sets `TrackQtyOnHand=true` and `QtyOnHand=0`

`qbMasterData(entityTypes=["account"], filter=accountName)`

Branching:
  `Service` or `NonInventory` with 0 income account matches → Ask:
  > Which income account should this item map to?
  `Service` or `NonInventory` with 1 income account match → Store `incomeAccountId`.
  `Inventory` with 0 asset account matches → Ask:
  > Which asset account should this inventory item map to?
  `Inventory` with 1 asset account match → Store `assetAccountId`.
  >1 account matches → Ask:
  > I found multiple matching accounts: [candidate list with names, types, and IDs]. Which account should this item map to?
  Missing `itemType` → Ask:
  > Should this item be `Service`, `NonInventory`, or `Inventory`?

### Step 3: Confirm and create item

Show: item name, `itemType`, mapped income account or asset account, and `expenseAccountId` if provided.

In interactive mode, emit:

Pre-write evidence:
- Entity: new item “[itemName]” (ID pending)
- Open-doc: not applicable for item create
- Account basis: duplicate check completed in Phase 5 Step 1; linked account resolved in Phase 5 Step 2
- Mode: interactive

`qbMasterData(operation="create", entityType="item", name=itemName, itemType=itemType, incomeAccountId=incomeAccountId, assetAccountId=assetAccountId, expenseAccountId=expenseAccountId)`

Branching:
  Success → Store returned item ID.
  Missing `incomeAccountId` for `Service` or `NonInventory` → Return to Phase 5 Step 2.
  Missing `assetAccountId` for `Inventory` → Return to Phase 5 Step 2.
  Duplicate/name exists error → Return to Phase 5 Step 1 and use the existing item or choose a different name.
  Month-end mode → Propose the item creation and do not create until approved.

## Phase 6 — Manage classes

### Step 1: Check for duplicate class before create

`qbMasterData(entityTypes=["class"], filter=className)`

Branching:
  0 matches → Proceed to Phase 6 Step 2.
  1 match → Ask:
  > I found an existing class “[className]” with ID [classId]. Do you want to use this existing class instead of creating a new one?
  >1 matches → Ask:
  > I found multiple matching classes: [candidate list with names, parent classes, and IDs]. Which one should I use, or should I create a new class with a different name?

### Step 2: Determine class hierarchy

If the class is a sub-class, resolve the parent class first.

`qbMasterData(entityTypes=["class"], filter=parentClassName)`

Branching:
  Top-level class → Proceed to Phase 6 Step 3 with no `parentClassId`.
  0 parent matches → Ask:
  > I couldn’t find parent class “[parentClassName]”. Should this be a top-level class, or should I search for a different parent class?
  1 parent match → Store `parentClassId`.
  >1 parent matches → Ask:
  > I found multiple possible parent classes for “[parentClassName]”: [candidate list]. Which parent class should I use?

### Step 3: Confirm and create class

Show: class name and parent class if any.

In interactive mode, emit:

Pre-write evidence:
- Entity: new class “[className]” (ID pending)
- Open-doc: not applicable for class create
- Account basis: duplicate check completed in Phase 6 Step 1; hierarchy resolved in Phase 6 Step 2
- Mode: interactive

`qbMasterData(operation="create", entityType="class", name=className, parentClassId=parentClassId)`

Branching:
  Success → Store returned class ID.
  Duplicate/name exists error → Return to Phase 6 Step 1 and use the existing class or choose a different name.
  User rejects confirmation → Ask what field should change, then repeat Phase 6 Step 3.
  Month-end mode → Propose the class creation and do not create until approved.

## Phase 7 — Update or deactivate records

### Step 1: Retrieve current record and fresh sync token

`qbMasterData(detailedInfo=entityType, filter=entityName)`

Branching:
  0 matches → Ask:
  > I couldn’t find “[entityName]” in QuickBooks. Should I search for a different name?
  1 match → Store `id`, current values, and fresh `syncToken`. `syncToken` is required for all updates and prevents overwriting concurrent changes.
  >1 matches → Interactive mode: ask:
  > I found multiple matching records for “[entityName]”: [candidate list with names and IDs]. Which one should I update?
  Batch/async mode: `flagForReview(aiReasoning="Multiple records matched update/deactivate request for [entityName]: [candidate list]. Need confirmation before updating or deactivating the wrong master-data record.")` (FLAG FOR REVIEW QUALITY GUARD requires specific `aiReasoning`).

### Step 2: Confirm changed fields

Present current values and requested new values. For vendor details, include address, payment terms, 1099 eligibility, tax identifier, email, and phone. For customer details, include terms, address, and contact info. For accounts, items, and classes, show all changed fields, including rename and hierarchy changes.

Branching:
  User confirms changes → Proceed to Phase 7 Step 3.
  User rejects changes → Ask:
  > What should I change instead?
  Missing changed fields → Ask:
  > What fields should I update on “[entityName]”?
  Month-end mode → Propose the update and do not apply it until approved.

### Step 3: Check open transactions before vendor or customer deactivation

For vendor deactivation:

`qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`

For customer deactivation:

`qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`

Branching:
  Not a vendor/customer deactivation → Proceed to Phase 7 Step 4.
  0 outstanding bills/invoices → Proceed to Phase 7 Step 4.
  1 outstanding bill/invoice → Ask:
  > This vendor/customer still has an open transaction: [transaction details]. QuickBooks may warn because deactivating hides the record but history is retained. Should I stop so the open transaction can be resolved first?
  >1 outstanding bills/invoices → Ask:
  > This vendor/customer has open transactions: [transaction list]. QuickBooks may warn if we deactivate before resolving them. Should I stop so these can be resolved first?
  Batch/async with any outstanding bills/invoices → `flagForReview(aiReasoning="Cannot safely deactivate [vendor/customer name] because outstanding [Bills/Invoices] exist: [transaction list]. QuickBooks may warn; resolve open transactions first.")` (FLAG FOR REVIEW QUALITY GUARD requires specific `aiReasoning`).

### Step 4: Update or deactivate entity

For update:

`qbMasterData(operation="update", entityType=entityType, id=entityId, syncToken=syncToken, changedFields=changedFields)`

For deactivate:

`qbMasterData(operation="update", entityType=entityType, id=entityId, syncToken=syncToken, active=false)`

In interactive mode, emit:

Pre-write evidence:
- Entity: “[entityName]” + ID [entityId]
- Open-doc: [no outstanding bills/invoices / outstanding transaction review completed / not applicable]
- Account basis: current record fetched with fresh `syncToken`; before/after confirmed
- Mode: interactive

Branching:
  Success update → Confirm the updated fields to the user.
  Success deactivate → Confirm the record is inactive/hidden but retained in history.
  Stale `syncToken` or concurrency error → Return to Phase 7 Step 1 and fetch a fresh record before retrying.
  User requested merge → Fetch both records with Phase 7 Step 1, present both current values, and require explicit confirmation before applying any merge-capable update supported by QuickBooks.
  Create/update requested for `taxRate` → Stop and tell the user:
  > Tax rates are read-only here and must be managed inside the QuickBooks Tax Center.

## Troubleshooting

MULTI-VENDOR AMBIGUITY GUARD blocks transaction tool — qbMasterData returned more than one matching vendor/customer and the agent tried to pick one → Resolve the entity using Phase 1 Step 1; interactive mode asks the user to choose, batch/async calls `flagForReview` with the candidate list.

VENDOR/ACCOUNT RESOLUTION GUARD blocks transaction tool — vendorId, customerId, or accountId was not returned by the most recent `qbMasterData` lookup → Re-call `qbMasterData` in Phase 1 Step 1 for the exact entity or account and retry only with returned IDs.

FLAG FOR REVIEW QUALITY GUARD blocks `flagForReview` — `aiReasoning` is missing, under 20 characters, or generic → Retry the flag with the exact entity name, candidate list, and why the ambiguity blocks safe posting or master-data change.

`qbMasterData` update rejected for stale `syncToken` — the record changed after it was fetched → Return to Phase 7 Step 1, retrieve the current record and fresh `syncToken`, re-confirm changed fields in Phase 7 Step 2, then update in Phase 7 Step 4.

Duplicate/name exists error on create — QuickBooks already has a matching account, vendor, customer, item, or class → Return to the entity duplicate-check step: Phase 2 Step 1, Phase 3 Step 1, Phase 4 Step 1, Phase 5 Step 1, or Phase 6 Step 1.

Missing item linked account error — `Service` or `NonInventory` item lacks `incomeAccountId`, or `Inventory` item lacks `assetAccountId` → Return to Phase 5 Step 2 and resolve the required account before creating.

Tax rate create/update fails — tax rates are read-only in this workflow and are managed by the QuickBooks Tax Center → Use Phase 1 Step 3 only to read tax rates; do not call create/update for `taxRate`.