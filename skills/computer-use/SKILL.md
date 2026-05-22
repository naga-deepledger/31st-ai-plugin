---
name: computer-use
description: Use browser_use or computer_use to interact with QuickBooks Online directly when the requested action cannot be completed through MCP tools — reconciliation finalization, bank connections, payroll, sales tax filing, user management, budgets, chart of accounts reordering, advanced settings, or any MCP tool returning "not supported". Activate when the user asks for any QuickBooks UI-only task or when an MCP tool returns "not supported" or "no API available".
---

## When browser is required

| QB Action | Why API can't do it |
|-----------|---------------------|
| Finalize bank reconciliation | Reconciliation screen is UI-only; no API to mark cleared or click "Finish Now" |
| Connect / disconnect bank accounts | Bank feed connections require OAuth in QB UI |
| Payroll | QB Payroll is a separate UI product with no public write API |
| Sales tax filing | Sales Tax Center is UI-only; filing and payment require browser |
| User management | Inviting/removing users and setting permissions is UI-only |
| Budget creation | QB Budgets module has no write API |
| Chart of accounts reordering | Display order changes are UI-only |
| Advanced settings | Company settings changes are UI-only |
| MCP tool returns "not supported" / "no API available" | Fallback to browser for that action |

If the request does **not** match any row above → delegate to the MCP-backed bookkeeping skill; do not open a browser.

## Browser operation

**1. Confirm before opening** (interactive mode):
> I can't complete this through the 31st.ai API because [reason]. I'm going to open QuickBooks Online in a browser and perform [action]. Please confirm I may proceed.

Batch/async: never ask mid-batch → `flagForReview(aiReasoning="Browser automation for [action] requires user confirmation before opening QuickBooks Online; batch/async mode cannot ask mid-batch.")`

**2. Open QuickBooks:**
`browser_use(task="Open QuickBooks Online at https://qbo.intuit.com/app/home and wait for the company dashboard to load")`

Use `browser_use` preferred; `computer_use` only if `browser_use` is unavailable or cannot control the page.

**3. Navigate** to the section matching the requested action.

**4. Perform reversible setup steps** — stop before any irreversible action (Finish Now, Submit, File, Pay).

**5. Capture screenshot** at the final confirmation screen before any irreversible click.

**6. Irreversible actions** — always ask first:
> I'm at the final confirmation screen for [action]. This may be irreversible. Do you want me to click [Finish Now / Submit / File / Pay] now?

Never click without explicit user confirmation.

**7. Hand back** with confirmation details: action performed, company, date/time, confirmation number, PDF/report name, and remaining next steps.

## Abort and hand back to human

Stop browser automation immediately when:
- QB asks for credentials or password → never enter; tell user to sign in directly
- MFA / security verification required → stop; tell user to complete it
- Legal, tax, payroll certification, or attestation required → stop; human-only
- Visible data conflicts with the request → stop; report the mismatch before anything changes

## Gotchas

1. `browser_use` is preferred — use `computer_use` only as fallback if `browser_use` can't control the page
2. Never enter or modify credentials, ever
3. Never click Finish Now / Submit / File / Pay without explicit user confirmation
4. Legal/payroll/tax attestations are human-only — always abort regardless of user request
5. Session loss mid-task → tell user before retrying; don't silently reopen and continue

After browser handback: any MCP writes triggered by the returned context remain subject to WRITE SAFETY GUARD and all transaction-specific guards.
