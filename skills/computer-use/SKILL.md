---
name: computer-use
description: Use browser_use or computer_use to interact with QuickBooks Online directly in the browser when the requested action cannot be completed through the 31st.ai MCP tools. Activate for UI-only QuickBooks tasks such as finalizing bank reconciliation, connecting bank accounts, payroll, sales tax filing, user management, advanced settings, budget creation, chart of accounts reordering, or any MCP tool error stating "not supported" or "no API available".
---

| Situation | Phase |
|---|---|
| User asks for a QuickBooks UI-only action or an MCP tool returned "not supported" / "no API available" | Phase 1 |
| Browser session is needed and user permission is required | Phase 2 |
| QuickBooks requests credentials, legal attestation, or human-only judgment | Phase 2 Step 6 |
| Browser action is complete and agent workflow should continue | Phase 3 |
| Browser session must be closed and work summarized | Phase 4 |

## Phase 1 — Verify browser use is required

### Step 1: Classify the requested QuickBooks action
Compare the requested action or prior MCP error against the API-unavailable classes below.

- Finalize bank reconciliation — QB reconciliation screen is UI-only; no API to mark cleared or click "Finish Now".
- Connect or disconnect bank accounts — bank feed connections require OAuth in the QB UI.
- Payroll — QB Payroll is a separate UI product with no public write API.
- Sales tax filing — Sales Tax Center is UI-only; filing and payment require browser interaction.
- User management — inviting/removing QB users and setting permissions is UI-only.
- Budget creation — QB Budgets module has no write API.
- Chart of accounts reordering — display order changes are UI-only.
- Advanced settings — company settings changes are UI-only.
- Any MCP tool returning an error — if a tool returns "not supported" or "no API available", switch to browser.

Branching:
- 0 matches → Do not open the browser. Delegate to the appropriate MCP-backed bookkeeping skill for the requested transaction or lookup.
- 1 match → Store `uiOnlyReason="[matched class and reason]"` and proceed to Phase 1 Step 2.
- >1 matches → Store `uiOnlyReason="[all matched classes]"`; if the tasks are separable, sequence them one at a time through Phase 2; if order is unclear, ask:

> I found multiple QuickBooks browser-only tasks in your request: [task list]. Which one should I do first?

### Step 2: Confirm browser access with the user
No QuickBooks browser session should be opened in interactive mode until the user confirms.

> I can’t complete this through the 31st.ai MCP API because [uiOnlyReason]. I’m going to open QuickBooks Online in a browser and perform [requested action]. Please confirm I may proceed.

Branching:
- User confirms → Proceed to Phase 2 Step 1.
- User does not confirm → Stop and say:

> Understood — I will not open QuickBooks Online in the browser. Tell me when you want to proceed.

- Batch/async mode → Never ask mid-batch. Call `flagForReview(aiReasoning="Browser automation for [requested action] requires user confirmation before opening QuickBooks Online; batch/async mode cannot ask mid-batch.")`.

## Phase 2 — Operate QuickBooks Online in the browser

### Step 1: Open QuickBooks Online
Use `browser_use` preferred, faster; use `computer_use` only if `browser_use` is unavailable or cannot control the page.

`browser_use(task="Open QuickBooks Online at https://qbo.intuit.com/app/home and wait for the company dashboard to load")`

Fallback:

`computer_use(action="open_url", url="https://qbo.intuit.com/app/home")`

Branching:
- Dashboard loads → Proceed to Phase 2 Step 2.
- QuickBooks asks for a password, credential entry, or credential modification → Do not enter or modify credentials in the browser. Stop and tell the user:

> QuickBooks is asking for credentials. I can’t enter or modify login credentials. Please sign in directly, then tell me when the QuickBooks dashboard is open.

- QuickBooks asks for MFA/security approval → Stop and tell the user:

> QuickBooks is asking for a security verification step. Please complete it directly, then tell me when the QuickBooks dashboard is open.

- Browser session is lost mid-task → Tell the user before retrying:

> The QuickBooks browser session was lost before I finished [requested action]. Do you want me to reopen QuickBooks Online and continue?

### Step 2: Navigate to the relevant QuickBooks section
Navigate to the section that matches the stored `uiOnlyReason`.

`browser_use(task="Navigate in QuickBooks Online to [reconciliation / bank connections / payroll / sales tax center / manage users / budgets / chart of accounts / advanced settings / section required by requested action]")`

Branching:
- Section opens and matches requested company/task → Proceed to Phase 2 Step 3.
- Section is unavailable because the QuickBooks plan lacks the feature → Stop and ask:

> This QuickBooks company does not appear to have access to [feature]. Should I stop here, or would you like instructions for enabling it?

- QuickBooks shows multiple companies or ambiguous company context → Stop and ask:

> QuickBooks is showing multiple companies. Which company should I use for [requested action]?

- Permission denied → Stop and hand back:

> QuickBooks says this user does not have permission to access [section]. Please have an admin complete this or update permissions.

### Step 3: Perform reversible setup steps
Before any browser UI write in interactive mode, emit:

Pre-write evidence:
- Entity: [company/user/tax agency/payroll/reconciliation account + ID if visible, otherwise "visible in UI; no API ID exposed"]
- Open-doc: [N/A for UI-only action / no outstanding docs affected / billId or invoiceId if visible]
- Account basis: [API unsupported: uiOnlyReason / N txns, XX% dominant if delegated from another skill / N/A for user management or settings]
- Mode: interactive

Then perform only reversible setup steps and stop before irreversible actions.

`browser_use(task="Complete the reversible setup steps for [requested action] in QuickBooks Online without clicking Finish Now, Submit, File, or Pay")`

Branching:
- Reversible setup completes → Proceed to Phase 2 Step 4.
- Month-end work reveals proposed entries or adjustments → Propose entries only; do not post without approval. Ask:

> I found a month-end issue that appears to require an entry or adjustment: [details]. I can propose it, but I will not post it without your approval. Do you want me to prepare the proposed entry for review?

- QuickBooks changes the task into an irreversible action earlier than expected → Stop and proceed to Phase 2 Step 4 before clicking.
- Unexpected discrepancy, warning, or mismatch appears → Do not continue. Ask:

> QuickBooks shows this warning before I can continue: [warning text]. Should I stop and hand this back to you, or do you want me to continue?

### Step 4: Capture confirmation evidence before irreversible action
Take a screenshot or capture visible confirmation details before clicking any final action.

`browser_use(task="Capture a screenshot of the final confirmation screen for [requested action], including visible amounts, dates, accounts, filings, payroll totals, reconciliation difference, or user permission changes")`

Branching:
- Screenshot captured and final button is not one of "Finish Now", "Submit", "File", or "Pay" → Proceed only if the action is still reversible; otherwise ask the confirmation prompt below.
- Screenshot captured and final button is "Finish Now", "Submit", "File", or "Pay" → Ask exactly:

> I’m at the final confirmation screen for [requested action]. This may be irreversible. Do you want me to click [Finish Now/Submit/File/Pay] now?

- User confirms → Proceed to Phase 2 Step 5.
- User does not confirm → Do not click; proceed to Phase 3 Step 1 with status `not_submitted`.

### Step 5: Complete the approved irreversible action
Only run this step after explicit user confirmation in Phase 2 Step 4.

`browser_use(task="Click [Finish Now/Submit/File/Pay] for [requested action], then capture the resulting confirmation screen")`

Branching:
- Confirmation screen appears → Store confirmation number, filing status, payroll status, reconciliation status, PDF/report link, or other visible completion evidence; proceed to Phase 3 Step 1.
- QuickBooks generates a PDF or report → Capture or download it, then proceed to Phase 3 Step 1:

`browser_use(task="Open or download the PDF/report generated for [requested action] and capture its title, date, and confirmation details")`

- QuickBooks returns an error after click → Do not retry blindly. Proceed to Phase 3 Step 1 with status `error_after_submission` and include the exact error text.

### Step 6: Abort and hand back to a human
Abort browser automation when QuickBooks requires credentials, legal/tax/payroll certification, admin-only permissions, user attestation, human-only judgment, or when the visible data conflicts with the user request.

`browser_use(task="Stop automation and leave the current QuickBooks screen unchanged for user review")`

Branching:
- Credentials, security verification, or password required → Hand back using the prompt from Phase 2 Step 1.
- Legal/tax/payroll certification or attestation required → Stop and ask:

> QuickBooks is asking for a legal, tax, payroll, or certification attestation that must be completed by an authorized human. I’m stopping here so you can review and decide.

- Visible data conflicts with the user request → Stop and ask:

> The QuickBooks screen does not match the requested action: [specific mismatch]. I’m stopping so a human can review before anything changes.

## Phase 3 — Hand back to the agent workflow

### Step 1: Report the browser outcome
Return the browser result to the main agent workflow with enough detail for audit continuity.

`browser_use(task="Read the final QuickBooks confirmation screen for [requested action] and extract confirmation numbers, dates, amounts, statuses, PDF/report names, and any warnings")`

Branching:
- Completed → Report: action performed, company, date/time, confirmation number, PDF/report name if generated, and any remaining next steps.
- Not submitted → Report that setup was performed but final action was not clicked because user did not approve.
- Error after submission → Report exact QuickBooks error text and whether the confirmation was created.
- Additional MCP-supported bookkeeping remains → Delegate to the appropriate MCP-backed skill; do not duplicate transaction logic here. API writes after handoff remain subject to WRITE SAFETY GUARD, DUPLICATE RESULT GUARD, VENDOR/ACCOUNT RESOLUTION GUARD, and the transaction-specific guards.

## Phase 4 — Close browser session and summarize

### Step 1: Close the browser session
Close the browser tab/window used for QuickBooks Online unless the user explicitly asks to keep it open.

`browser_use(task="Close the QuickBooks Online browser tab or window used for this session")`

Fallback:

`computer_use(action="close_window", target="QuickBooks Online browser window")`

Branching:
- Browser closed → Proceed to Phase 4 Step 2.
- User asks to keep browser open → Leave it open and proceed to Phase 4 Step 2.
- Browser cannot be closed by automation → Tell the user:

> I could not close the QuickBooks browser window automatically. Please close it manually when you are finished.

### Step 2: Summarize what was done
Provide a concise audit summary.

Branching:
- Completed action → Summarize `[requested action]`, `[company]`, `[confirmation details]`, `[reports/PDFs]`, and `[remaining follow-up]`.
- Aborted action → Summarize where automation stopped, why it stopped, and what the human should do next.
- Delegated follow-up → State which MCP-backed skill should continue and what evidence was collected from the browser.

## Troubleshooting

WRITE SAFETY GUARD blocks qbExpense|qbBill|qbBillPayment|qbInvoice|qbSalesReceipt|qbReceivePayment|qbDeposit|qbJournalEntry — MCP write attempted without required lookup/duplicate evidence → return to Phase 1 Step 1 and use browser only if the action is API-unavailable; otherwise delegate to the proper MCP-backed skill.

BATCH SAFETY GUARD blocks qbBatch — batch operation cannot proceed safely through MCP → return to Phase 1 Step 1; if the task is UI-only, continue with Phase 1 Step 2, otherwise split/delegate by transaction type.

FLAG FOR REVIEW QUALITY GUARD blocks flagForReview — aiReasoning was missing or generic → retry the call from Phase 1 Step 2 with a specific reason naming the browser-only action and why batch/async cannot ask mid-batch.

browser_use session expired — QuickBooks session ended or lost state → recover at Phase 2 Step 1 and tell the user before reopening.

QuickBooks permission denied — current user lacks access to the selected section → recover at Phase 2 Step 2 by stopping and asking for admin permission or human completion.

QuickBooks asks for password, MFA, legal attestation, tax certification, or payroll certification — human-only step required → recover at Phase 2 Step 6 and hand back without continuing.