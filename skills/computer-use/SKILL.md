---
name: computer-use
description: Use browser_use or computer_use to interact with QuickBooks Online directly in the browser. Activate when the user needs to do something in QuickBooks that the 31st.ai MCP tools cannot do — such as finalizing a bank reconciliation, connecting bank accounts, payroll, sales tax filing, or any action that returns a "not supported" error from MCP tools.
---

# Computer Use Skill

When an action cannot be completed via MCP tools, use `browser_use` or `computer_use` to operate the QuickBooks Online browser UI directly.

## When to Use This Skill

Activate when the user asks for any of the following — all require browser access because QuickBooks does not expose these operations via API:

| Task | Why browser required |
|------|----------------------|
| **Finalize bank reconciliation** | QB reconciliation screen is UI-only; no API to mark cleared or click "Finish Now" |
| **Connect or disconnect bank accounts** | Bank feed connections require OAuth in the QB UI |
| **Payroll** | QB Payroll is a separate UI product with no public write API |
| **Sales tax filing** | Sales Tax Center is UI-only; filing and payment require browser interaction |
| **User management** | Inviting/removing QB users and setting permissions is UI-only |
| **Budget creation** | QB Budgets module has no write API |
| **Chart of accounts reordering** | Display order changes are UI-only |
| **Any MCP tool returning an error** | If a tool returns "not supported" or "no API available", switch to browser |

## How to Proceed

1. Confirm with the user that you are about to open their QuickBooks Online account in the browser.
2. Use `browser_use` (preferred, faster) or `computer_use` to navigate to the relevant QB section.
3. Perform the steps on the user's behalf, narrating each action clearly.
4. Stop and ask for explicit confirmation before irreversible actions (e.g., clicking "Finish Now" in reconciliation, submitting a payroll run, or filing a tax return).
5. Report the outcome and any confirmation numbers or PDF reports generated.

## Hard Rules

- Never click "Finish Now", "Submit", "File", or "Pay" without explicit user confirmation — even if everything looks correct.
- Do not enter or modify credentials in the browser — if QB asks for a password, stop and tell the user.
- If you lose the browser session mid-task, tell the user before retrying.
