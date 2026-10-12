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
- **"Do it" / "run month-end" / "sort out my books"** or no instruction at all: do everything that can be undone inside Looped, then stop at the irreversible steps and present them for a decision. Reversible means posted journals (undone with `reverse_journal_entry`), approved statement lines, approved and paid bills, confirmed receipts, categorisation rules, depreciation, loan interest and FX runs, provisional tax estimates (rerun to replace), issued invoices and recorded payments (undone with `reverse_invoice_payment`, a credit note or `refund_invoice`), time entries, projects, clients, suppliers, todos and routines.
- **"Do everything" / "send them" / "finalise it"**: also take the irreversible steps the user named. Never take an irreversible step the user did not name.

Irreversible steps, always named back to the user before taking them: `send_invoice` and `send_credit_note` (they email the client), `process_pay_run` (the server also demands the confirmed net total), `finalise_afs_pack`, any `delete_*`, `void_bill`, `dispose_fixed_asset`, `disconnect_integration`, `delete_bank_import`, `dismiss_blocked_posting` and refreshing a pay run draft with `useLatestInputs`.

Some things only a person can do in the Looped web app, whatever the dial: closing or reopening a financial year, finalising or reopening a VAT201, filing any return, changing bank account numbers or holders, finalising a pay run that has changes, reversing a pay run, confirming a payroll take-on and completing an employee's identity, tax and bank details. Prepare the work, then hand over the link the tool returns.

Whatever the dial, work in one pass. Read everything in scope before proposing, present the whole plan as one list the user can approve in one reply, and once they approve, carry out all of it in the same turn rather than stopping after the first group.

## Conventions

- Look things up before acting (`list_*` and `search_*` tools) and carry ids forward. Never guess an id.
- Call `setup_status` before helping a new company or when the user asks what is missing. Its `blockedPostings` tells you which invoices and payments are not in the books and why; relay each message.
- Amounts: Financials, Payroll and invoice payment amounts are integer cents; invoice line items and project rates are major currency units. Follow each tool's field description. `get_team_info` gives the Track and Invoices currency; the Financials ledger currency follows the company's country (`get_company`).
- Dates go to tools as ISO `YYYY-MM-DD`; durations are in seconds. Tool dates are UTC.
- Looped does the accounting. Never pre-convert currencies, recalculate balances or adjust amounts outside Looped. When Looped cannot do something, say so plainly.
- Posted journal entries are never edited; undo with `reverse_journal_entry`. Journal numbers (#5) are `entryNo` on `list_journal_entries`.
- Importing and categorising a bank statement only proposes accounts. Approving a line posts it. Transfers between the company's own bank accounts go to the Transfers between accounts clearing account from both statements. Excluding is only for duplicates and lines that are not transactions.
- For PDFs, statements and exports (`get_invoice_pdf`, `get_credit_note_pdf`, `get_client_statement`, `export_report`), present the returned HTTPS link and say it expires in 15 minutes. Call again for a fresh one. Never invent a URL and never claim a file was attached unless the host shows it.
- Recording a payment or refund does not move money. Never describe an accounting record as a bank transfer, and never ask for bank logins, OTPs or integration secrets.
- Memory: when the user says "remember", "always" or "my preference is", or when a decision is made or work is left unfinished, `save_memory` in that company (pinned for decisions and open items that must not be missed). `search_memories` before starting work in a company you have not touched this session.
- Routines: `create_routine` runs a plain-language prompt on a schedule with these same tools. Confirm the schedule and the prompt text with the user before creating one.
- Docs: `search_docs` answers how-Looped-works questions from the product documentation. Use it before guessing at a setting that has no tool.

## Sensitive information

Never ask for ID, passport or personal tax numbers, dates of birth or employee bank details. `create_employee` takes basic starter fields only; show its `completeDetailsUrl` so a person completes the rest in Payroll. `update_company_bank_account` changes label, bank, branch, type, currency, default and archive only. Payroll identifiers come back masked; keep them that way. Never try to bypass a server rejection.

## What the tools cover

| Area | Read | Write |
| --- | --- | --- |
| Track | `list_time_entries`, `get_time_entry`, `list_projects`, `search_projects`, `list_project_members`, `list_project_member_rates`, `list_track_teammates`, `get_time_preferences`, `list_tags`, `list_sections`, `list_todos` | `create_time_entry`, `create_time_entry_for_user`, `update_time_entry`, `update_time_entry_tags`, `log_time_from_todo`, `create_project`, `update_project`, `add_project_member`, `set_project_member_rate_period`, `create_todo`, `update_todo`, tags and sections |
| Invoices | `list_invoices`, `get_invoice`, `list_clients`, `get_client`, `get_client_statement`, `list_customer_balances`, `get_invoice_pdf`, `get_credit_note_pdf` | `create_invoice`, `create_invoice_from_time_entries`, `draft_invoice_from_document`, `update_invoice`, `issue_invoice`, `send_invoice`, `mark_invoice_paid`, `reverse_invoice_payment`, `refund_invoice`, `create_credit_note`, `send_credit_note`, `mark_invoice_already_in_books`, clients, company details, bank accounts, VAT and currency |
| Financials: bank | `list_bank_imports`, `get_bank_import`, `list_categorisation_rules` | `import_bank_statement`, `categorise_bank_import`, `create_categorisation_rule`, `apply_categorisation_rules`, `review_statement_line`, `bulk_review_statement_lines`, `split_statement_line`, `reclassify_statement_lines` |
| Financials: purchases | `list_suppliers`, `list_bills`, `get_bill`, `aged_payables`, `list_receipts`, `get_receipt` | `create_supplier`, `create_bill`, `approve_bill`, `pay_bill`, `void_bill`, `capture_receipt`, `confirm_receipt`, `match_receipt`, `convert_bill_to_receipt`, `convert_receipt_to_bill`, `attach_document`, `create_upload_url` |
| Financials: ledger | `list_ledger_accounts`, `get_account_ledger`, `list_journal_entries`, `get_trial_balance`, `get_income_statement`, `get_balance_sheet`, `get_cash_flow_statement`, `get_equity_statement`, `export_report`, `list_audit_log` | `create_ledger_account`, `update_ledger_account`, `create_journal_entry`, `reverse_journal_entry`, `recode_journal_entries`, `capture_opening_balances`, `create_financial_year` |
| Financials: assets, loans, FX | `list_fixed_assets`, `list_loans`, `list_exchange_rates`, `get_exchange_rate` | `create_fixed_asset`, `update_fixed_asset`, `dispose_fixed_asset`, `run_depreciation`, `set_loan_terms`, `run_loan_interest`, `set_exchange_rate`, `run_fx_revaluation` |
| Financials: tax and year-end | `get_vat201`, `get_tax_computation`, `get_deferred_tax`, `list_financial_years`, `list_afs_notes` | `estimate_provisional_tax`, `post_deferred_tax`, `compile_afs_pack`, `sync_afs_notes`, `create_afs_note`, `update_afs_note`, `reorder_afs_notes`, `finalise_afs_pack` |
| Payroll | `list_employees`, `get_employee`, `list_pay_runs`, `get_pay_run`, `get_pay_run_inputs`, `get_payslip`, `list_payroll_runs`, `get_payroll_take_on` | `create_employee`, `set_employee_sdl_exclusion`, `draft_pay_run`, `add_pay_run_item`, `process_pay_run`, `post_payroll_run`, `start_payroll_take_on`, `save_payroll_take_on_employees`, `set_payroll_take_on_opening_ytd`, `set_payroll_take_on_sdl` |
| Platform | `list_teams`, `get_team_info`, `get_company`, `get_company_details`, `setup_status`, `list_integrations`, `list_routines`, `list_memories`, `search_memories`, `search_docs` | `create_team`, `update_company`, `create_routine`, `update_routine`, `save_memory`, `send_feedback`, integrations |

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
