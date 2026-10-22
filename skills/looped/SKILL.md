---
name: looped
description: >
  This skill should be used when the user asks to do anything in their Looped account:
  log or review time, manage projects, clients and tasks, create, issue or send invoices
  and credit notes, chase who owes them money, import or review a bank statement, capture
  bills and receipts, post journals, run month-end or year-end, prepare the VAT201 or
  provisional tax, compile annual financial statements, draft or finalise payroll, or set
  up a scheduled routine. Trigger on "in Looped", "my books", "invoice", "bank statement",
  "pay run", "VAT201", "month-end", "time entries", and the names Track, Invoices,
  Financials and Payroll.
---

# Looped

Looped is a suite for consultants and small service businesses: time tracking (Track), invoicing (Invoices), team and billing (Manage), double-entry accounting (Financials) and Payroll. Work in it through the Looped MCP server. Every tool acts as the signed-in user, with their role and permissions in the company it runs in.

## Connect

- MCP server: `https://api.looped.sh/mcp` (Streamable HTTP, OAuth). The first call opens a Looped sign-in.
- The user can act in every company (team) they belong to. `list_teams` lists them; every other tool takes `teamId`, required when the user has more than one company. Ask which company when it is unclear, and never carry an action over from one company to another.
- If the Looped tools are missing, tell the user to enable the Looped connector for this plugin and sign in. Do not work around it.

## How far to go

The user sets the dial for each job. Read their request for it and follow it exactly:

- **"Just show me" / "draft" / "don't post anything"**: read, analyse and prepare. Create drafts (invoices, bills, receipts, pay runs, categorisation proposals) and post nothing.
- **"Do it" / "run month-end" / "sort out my books"** or no instruction at all: do everything that can be undone inside Looped, then stop at the irreversible steps and present them for a decision. Reversible means posted journals (undone with `journal_entries_write` action `reverse`), approved statement lines, approved and paid bills, confirmed receipts, categorisation rules, depreciation, loan interest and FX runs, provisional tax estimates (rerun to replace), issued invoices and recorded payments (undone with `invoices_write` action `reverse_payment`, a credit note or `invoices_write` action `refund`), time entries, projects, clients, suppliers, todos and routines.
- **"Do everything" / "send them" / "finalise it"**: also take the irreversible steps the user named. Never take an irreversible step the user did not name.

Irreversible steps, always named back to the user before taking them: `invoices_write` action `send` and `credit_notes_write` action `send` (they email the client), `pay_runs_write` action `process` (the server also demands the confirmed net total), `afs_write` action `finalise`, any `delete_*`, `bills_write` action `void`, `fixed_assets_write` action `dispose`, `disconnect_integration`, `bank_imports_delete` action `delete`, `journal_entries_write` action `dismiss_blocked` and refreshing a pay run draft with `useLatestInputs`.

Some things only a person can do in the Looped web app, whatever the dial: closing or reopening a financial year, finalising or reopening a VAT201, filing any return, changing bank account numbers or holders, finalising a pay run that has changes, reversing a pay run, confirming a payroll take-on and completing an employee's identity, tax and bank details. Prepare the work, then hand over the link the tool returns.

Whatever the dial, work in one pass. Read everything in scope before proposing, present the whole plan as one list the user can approve in one reply, and once they approve, carry out all of it in the same turn rather than stopping after the first group.

## Conventions

- Look things up before acting (`list_*` and `search_*` tools) and carry ids forward. Never guess an id.
- Call `financial_reports_read` action `setup_status` before helping a new company or when the user asks what is missing. Its `blockedPostings` tells you which invoices and payments are not in the books and why; relay each message.
- Amounts: Financials, Payroll and invoice payment amounts are integer cents; invoice line items and project rates are major currency units. Follow each tool's field description. `get_team_info` gives the Track and Invoices currency; the Financials ledger currency follows the company's country (`company_read` action `get`).
- Dates go to tools as ISO `YYYY-MM-DD`; durations are in seconds. Tool dates are UTC.
- Looped does the accounting. Never pre-convert currencies, recalculate balances or adjust amounts outside Looped. When Looped cannot do something, say so plainly.
- Posted journal entries are never edited; undo with `journal_entries_write` action `reverse`. Journal numbers (#5) are `entryNo` on `journal_entries_read` action `list`.
- Importing and categorising a bank statement only proposes accounts. Approving a line posts it. Transfers between the company's own bank accounts go to the Transfers between accounts clearing account from both statements. Excluding is only for duplicates and lines that are not transactions.
- For PDFs, statements and exports (`invoices_read` action `pdf`, `credit_notes_read` action `pdf`, `invoices_read` action `client_statement`, `financial_reports_read` action `export`), present the returned HTTPS link and say it expires in 15 minutes. Call again for a fresh one. Never invent a URL and never claim a file was attached unless the host shows it.
- Recording a payment or refund does not move money. Never describe an accounting record as a bank transfer, and never ask for bank logins, OTPs or integration secrets.
- Memory: when the user says "remember", "always" or "my preference is", or when a decision is made or work is left unfinished, `memories_write` action `save` in that company (pinned for decisions and open items that must not be missed). `memories_read` action `search` before starting work in a company you have not touched this session.
- Routines: `routines_write` action `create` runs a plain-language prompt on a schedule with these same tools. Confirm the schedule and the prompt text with the user before creating one.
- Docs: `search_docs` answers how-Looped-works questions from the product documentation. Use it before guessing at a setting that has no tool.

## Sensitive information

Never ask for ID, passport or personal tax numbers, dates of birth or employee bank details. `employees_write` action `create` takes basic starter fields only; show its `completeDetailsUrl` so a person completes the rest in Payroll. `company_write` action `update_bank_account` changes label, bank, branch, type, currency, default and archive only. Payroll identifiers come back masked; keep them that way. Never try to bypass a server rejection.

## What the tools cover

| Area | Read | Write |
| --- | --- | --- |
| Track | `time_entries_read` (list, get, teammates, preferences), `projects_read` (list, search, members, member_rates), `tags_read` (list), `sections_read` (list), `todos_read` (list) | `time_entries_write` (create, create_for_user, update, update_tags), `todos_write` (log_time, create, update), `projects_write` (create, update, add_member, set_rate_period), tags and sections |
| Invoices | `invoices_read` (list, get, client_statement, pdf), `clients_read` (list, get), `accounting_reports_read` (customer_balances), `credit_notes_read` (pdf) | `invoices_write` (create, from_time_entries, draft_from_document, update, issue, send, mark_paid, reverse_payment, refund, mark_in_books), `credit_notes_write` (create, send), clients, company details, bank accounts, VAT and currency |
| Financials: bank | `bank_imports_read` (list, get), `categorisation_rules_read` (list) | `bank_imports_write` (import, categorise, apply_rules, review_line, bulk_review, split, reclassify), `categorisation_rules_write` (create) |
| Financials: purchases | `suppliers_read` (list), `bills_read` (list, get, aged_payables), `receipts_read` (list, get) | `suppliers_write` (create), `bills_write` (create, approve, pay, void, convert_to_receipt), `receipts_write` (capture, confirm, match, convert_to_bill), `attach_document`, `create_upload_url` |
| Financials: ledger | `accounts_read` (list), `journal_entries_read` (ledger, list), `financial_reports_read` (trial_balance, income_statement, balance_sheet, cash_flow, equity, export, audit_log) | `accounts_write` (create, update), `journal_entries_write` (create, reverse, recode), `financial_years_write` (opening_balances, create) |
| Financials: assets, loans, FX | `fixed_assets_read` (list), `loans_read` (list), `exchange_rates_read` (list, get) | `fixed_assets_write` (create, update, dispose, depreciate), `loans_write` (set_terms, run_interest), `exchange_rates_write` (set), `journal_entries_write` (revalue_fx) |
| Financials: tax and year-end | `accounting_reports_read` (vat201), `tax_read` (computation, deferred), `financial_years_read` (list), `afs_read` (notes) | `tax_write` (estimate_provisional, post_deferred), `afs_write` (compile, sync_notes, create_note, update_note, reorder_notes, finalise) |
| Payroll | `employees_read` (list, get), `pay_runs_read` (list, get, inputs, payslip), `payroll_journals_read` (list), `payroll_take_on_read` (get) | `employees_write` (create, set_sdl_exclusion), `pay_runs_write` (draft, add_item, process), `payroll_journals_write` (post), `payroll_take_on_write` (start, save_employees, set_opening_ytd, set_sdl) |
| Platform | `list_teams`, `get_team_info`, `company_read` (get), `afs_read` (company_details), `financial_reports_read` (setup_status), `list_integrations`, `routines_read` (list), `memories_read` (list, search), `search_docs` | `company_write` (update), `routines_write` (create, update), `memories_write` (save), `create_team`, `send_feedback`, integrations |

## Playbooks

Read `references/playbooks.md` for the step-by-step playbooks before starting any of these jobs:

- Set up Financials from prior-year financial statements
- Import a bank statement
- Review a bank statement upload
- Capture bills and receipts
- Month-end close
- Set up invoicing
- Invoice from logged time and chase what is owed
- VAT201 and provisional tax
- Year-end and the AFS pack
- Payroll: draft, explain and finalise a pay run
- Payroll take-on from Sage

The server also offers the first four as MCP prompts (`setup-from-prior-financials`, `import-bank-statement`, `review-bank-statement`, `setup-invoicing`). Reply once at the end of each job with what was done, what is still open and any link the user needs to act on.
