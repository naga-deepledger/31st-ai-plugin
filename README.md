# 31st.ai Plugin

Skills and safety hooks for AI-assisted bookkeeping with QuickBooks Online via the 31st.ai MCP server.

## What This Plugin Does

Adds two things to Claude Code:

1. **Skills** — domain expertise files that tell Claude how to handle QuickBooks accounting tasks correctly: which tools to use, in what order, and what to watch out for
2. **Hooks** — pre-tool-call safety guards that block common bookkeeping mistakes before they hit QuickBooks

Connect your own 31st.ai MCP server, and these skills and hooks activate automatically.

## Skills

| Skill | Purpose |
|-------|---------|
| `accounts-payable` | Bills, bill payments, vendor management |
| `accounts-receivable` | Invoices, receive payments, customer management |
| `bank-feed-processing` | Categorize and record unmatched bank transactions |
| `bank-reconciliation` | Prep accounts for reconciliation (health check, feed, duplicates) |
| `client-onboarding` | Bootstrap agent memory from QB history |
| `computer-use` | Browser-based QB tasks the API can't do (reconciliation finalization, payroll, sales tax) |
| `financial-analysis` | P&L, balance sheet, cash flow, ratios, trend analysis |
| `journal-entries` | Adjusting entries, accruals, depreciation, reclassifications |
| `master-data` | Resolve vendor/customer/account IDs before any write |
| `month-end-close` | Structured close workflow: health checks, accruals, depreciation, trial balance |

## Safety Hooks

All QuickBooks write calls are protected by 11 pre-tool-call guards:

| Guard | What It Catches |
|-------|----------------|
| Write Safety | Blocks any transaction without prior `qbMasterData` lookup and `qbFetchTransactions` duplicate check |
| Duplicate Result | Blocks when the duplicate check returned potential matches — forces user confirmation |
| Expense Type | Catches Bill/Expense confusion, flags outstanding bills before allowing an Expense |
| Deposit Type | Blocks Deposits that should be ReceivePayment + Deposit, or SalesReceipt + Deposit |
| Income Type | Prevents SalesReceipt when an Invoice exists, ReceivePayment when no Invoice exists |
| Vendor/Account Resolution | Cross-checks all IDs against the most recent `qbMasterData` result |
| Amount Anomaly | Flags transactions 3x outside the vendor's learned amount range |
| Journal Entry Balance | Hard-blocks unbalanced JEs before QB rejects them |
| Source-Category Collision | Prevents bank account appearing as both source and category |
| Void Guard | Requires fetch-then-verify before any void |
| Batch Safety | Enforces master data lookup, type homogeneity, and duplicate check for batch operations |
| Flag Quality | Rejects vague `flagForReview` calls — requires specific `aiReasoning` |

## Architecture

```
Claude Code
  ├── skills/      → Domain expertise (how to use QB tools correctly)
  └── hooks/       → Safety guards (what to block before each write)
       ↕ (you connect your own MCP server)
31st.ai MCP Server → QuickBooks Online
```

## Setup

Connect the 31st.ai MCP server in your Claude Code settings, then install this plugin:

```bash
claude --plugin-dir ./31st-ai-plugin
```

Skills activate automatically based on what you ask. Hooks fire before every QuickBooks write.

## Agent Memory

The `client-onboarding` skill bootstraps vendor→account mappings from QB history. Once seeded, every successful transaction upvotes the mapping — confidence grows over time and reduces CPA review flags.

## Version

See [CHANGELOG.md](CHANGELOG.md) for release history.
