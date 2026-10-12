# Looped for Claude

Run your business in Looped from Claude. Track time, invoice clients, keep the books, prepare payroll and set up routines that run on a schedule, across every company you belong to.

## What is in the plugin

- **Skill: `looped`**. Teaches Claude how Looped works: companies and permissions, currency units, what posts to the ledger and what only proposes, which steps a person has to take in the web app, and eleven step-by-step playbooks (bank statement import and review, bills and receipts, month-end, invoicing from logged time, VAT201 and provisional tax, year-end and the AFS pack, payroll).
- **MCP server: `looped`**. The Looped MCP server at `https://api.looped.sh/mcp`. It signs you in with your normal Looped account the first time Claude calls it, and every tool then acts as you, with your roles in each company.

## Setup

Once the plugin is listed, you add it from the directory inside Claude. Until then, if you use Claude Code you can run:

```
/plugin install looped --marketplace loopedautomation/claude-plugin
```

1. Install the plugin.
2. Enable the Looped connector when prompted and sign in at the Looped page that opens.
3. Ask Claude something like "Show my Looped companies".

There are no environment variables or API keys to configure.

## How far Claude goes

You decide per request:

- "Show me" or "draft": Claude reads and prepares and posts nothing.
- "Do it" or no instruction: Claude does everything that can be undone inside Looped (posting journals, approving bank lines, approving bills, issuing invoices, running depreciation) and stops at the irreversible steps, which it lists for you.
- "Send it" or "finalise it": Claude also takes the irreversible step you named. It never takes one you did not name.

Some things are always yours to do in the Looped web app: closing a financial year, finalising or filing a VAT201, finalising a pay run with changes, changing bank account numbers, confirming a payroll take-on and completing an employee's identity, tax and bank details. Claude prepares the work and gives you the link.

## Example requests

- "Import this bank statement into Looped Automation and review it."
- "Run month-end for September and give me the income statement."
- "Invoice Acme for the team's time in September at R500 an hour and give me the PDF."
- "Who owes me money, and draft a chaser for anything over 30 days."
- "What do I type into eFiling for the Jul to Aug VAT201?"
- "Draft this month's pay run and explain why Thabo's PAYE changed."
- "Every Monday at 08:00, send me a summary of unreviewed bank lines." (creates a routine)

## Limits

Looped records payments and refunds in the books; it does not move money through a bank. Claude will not ask for bank logins, OTPs, ID numbers or employee bank details, and the server masks payroll identifiers in what it returns.

## Support

https://looped.sh/contact or support@looped.sh
