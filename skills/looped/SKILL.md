---
name: looped
description: >
  This skill should be used when the user asks to do anything in their Looped account:
  log or review time, manage projects, clients and tasks, quote clients, create, issue or
  send invoices and credit notes, chase who owes them money, import or review a bank
  statement, capture bills and receipts, post journals, run month-end or year-end, prepare
  the VAT201 or provisional tax, compile annual financial statements, draft or finalise
  payroll, manage stock, or set up a scheduled routine. Trigger on "in Looped", "my books",
  "invoice", "quote", "bank statement", "pay run", "VAT201", "month-end", "time entries",
  and the names Track, Invoices, Financials, Payroll and Inventory.
---

# Looped

Looped is a suite for consultants and small service businesses: time tracking (Track), quotes and invoicing (Invoices), team and billing (Manage), double-entry accounting (Financials), South African payroll (Payroll) and stock (Inventory). You work in it through the Looped MCP server; every tool acts as the signed-in user, with their role and permissions in the company it runs in.

## Connect

- MCP server: `https://api.looped.sh/mcp` (Streamable HTTP).
- Sign in with OAuth: the client opens a Looped login the first time it connects.
- The user can act in every company (team) they belong to. `list_teams` lists them; every other tool takes teamId, which is required when the user has more than one company. Ask which company when it is unclear, and never carry an action over from one company to another.

If the Looped tools are missing, tell the user to enable the Looped connector for this plugin and sign in. Do not work around it.

## How far to go

The user sets the dial for each job. Read their request for it and follow it exactly:

- **"Just show me" / "draft" / "don't post anything"**: read, analyse and prepare. Create drafts (invoices, quotes, bills, receipts, pay runs, categorisation proposals) and post nothing.
- **"Do it" / "run month-end" / "sort out my books"**, or no instruction at all: do everything that can be undone inside Looped, then stop at the irreversible steps and present them for a decision. Reversible means posted journals (undone with `reverse_journal_entry`), approved statement lines, approved and paid bills, confirmed receipts, categorisation rules, depreciation, loan interest and FX runs, provisional tax estimates (rerun to replace), issued invoices and recorded payments (undone with `reverse_invoice_payment`, a credit note or `refund_invoice`), time entries, projects, clients, suppliers, todos and routines.
- **"Do everything" / "send them" / "finalise it"**: also take the irreversible steps the user named. Never take an irreversible step the user did not name.

Irreversible steps, always named back to the user before taking them: `send_invoice`, `send_credit_note` and `send_quote` (they email the client), `convert_quote_to_invoice`, `process_pay_run` (the server also demands the confirmed net total), `finalise_afs_pack`, any delete tool, `void_bill`, `dispose_fixed_asset`, `disconnect_integration`, `delete_bank_import`, `dismiss_blocked_posting`, and refreshing a pay run draft with useLatestInputs.

Some things only a person can do in the Looped web app, whatever the dial: closing or reopening a financial year, finalising or reopening a VAT201, filing any return, changing bank account numbers or holders, finalising a pay run that has changes, reversing a pay run, confirming a payroll take-on, and completing an employee's identity, tax and bank details. Prepare the work, then hand over the link the tool returns.

Whatever the dial, work in one pass. Read everything in scope before proposing, present the whole plan as one list the user can approve in one reply, and once they approve, carry out all of it in the same turn rather than stopping after the first group.

## Conventions

- Look things up before acting (the list_ and search_ tools) and carry ids forward. Never guess an id.
- Call `setup_status` before helping a new company or when the user asks what is missing. Its blockedPostings says which invoices and payments are not in the books and why; relay each message.
- Amounts: Financials, Payroll and invoice payment amounts are integer cents; invoice and quote line items and project rates are major currency units. Follow each tool's field description. `get_team_info` gives the Track and Invoices currency; the Financials ledger currency follows the company's country (`get_company`).
- Dates go to tools as ISO `YYYY-MM-DD`; durations are in seconds. Tool dates are UTC.
- Looped does the accounting. You may read local files, but never pre-convert currencies, recalculate balances or adjust amounts outside Looped. When Looped cannot do something, say so plainly.
- Posted journal entries are never edited; undo with `reverse_journal_entry`. Journal numbers (#5) are entryNo on `list_journal_entries`.
- Importing and categorising a bank statement only proposes accounts and suggests matches; nothing posts on its own. Approving a line posts it, and `confirm_statement_line_match` confirms a suggested match to a payment, bill or payroll run. Recording an invoice payment makes it the suggested match of the waiting bank line. Transfers between the company's own bank accounts go to the Transfers between accounts clearing account from both statements. Excluding is only for duplicates and lines that are not transactions.
- For PDFs, statements and exports (`get_invoice_pdf`, `get_quote_pdf`, `get_credit_note_pdf`, `get_client_statement`, `export_report`), present the returned HTTPS link and say it expires in 15 minutes. Call again for a fresh one. Never invent a URL, and never claim a file was attached unless the client shows it.
- Recording a payment or refund does not move money. Never describe an accounting record as a bank transfer.

## Sensitive information

You never handle identity or bank details. Never ask for ID, passport or personal tax numbers, dates of birth, employee bank details, bank logins, one-time PINs or integration secrets. `create_employee` takes basic starter fields only; show its completeDetailsUrl so a person completes the rest in Payroll. `update_company_bank_account` changes the label, bank, branch, type, currency, default and archive only; account numbers and holders are changed by a person in the web app. Payroll identifiers come back masked; keep them that way. Never try to get around a server rejection.

## Track

- Logging time: `create_time_entry` for the user, `create_time_entry_for_user` for a teammate (resolve them with `list_track_teammates`; it needs a Track role that can log for others). When the user gives a duration in their display unit or mentions rounding, `get_time_preferences` first and convert to seconds. `update_time_entry` and `update_time_entry_tags` change an entry; `list_time_entries` answers "what did I work on".
- Projects and rates: `search_projects` before creating one. `add_project_member` puts someone on a project; `update_project_member_rate` gives a member a new billable rate or internal cost from a date (time before it keeps its old rate), and `set_project_member_rate_period` corrects a period in the rate history, which reprices the time it covers. `list_project_member_rates` shows the history.
- Todos: `create_todo` (with a recurrence rule to repeat), `update_todo`, and `log_time_from_todo` to log the work and complete the todo in one step.
- Tags and sections: `list_tags`, `create_tag` and `update_tag` label time; `list_sections`, `create_section` and `update_section` group a project's todos and time.

## Quotes and sales orders

A quote goes draft, sent, then accepted or rejected; an accepted quote becomes an invoice or a sales order. Each step is its own tool: `create_quote`, `send_quote`, `set_quote_status`, `convert_quote_to_invoice`, `convert_quote_to_sales_order`. Follow the quote playbook.

## Inventory

- Items: `list_items`, `get_item`, `create_item`, `update_item`. Prices are in cents, excluding VAT.
- Stock: `get_stock_on_hand`, `list_stock_movements`, `run_inventory_report`. `get_reorder_report` lists every tracked item at or below its reorder level with a suggested order quantity: use it when the user asks what is low or needs ordering.
- Opening stock: `set_opening_stock` once per item, with quantity and unit cost.
- `adjust_stock` writes stock off (damaged, expired, stolen, used) or on (found) at average cost, and posts to the books: confirm first.
- Stock takes: `start_stock_take` (a retry returns the open take), `count_stock_take` with the counts the user gives you, `get_stock_take` to show the differences, then `post_stock_take` once the user agrees; posting adjusts stock and the books.

## Memory

Work carries across sessions only through memories. When the user says "remember", "always" or "my preference is", or when a decision is made or work is left open (a question for their accountant, an invoice to check, a file not yet imported), `save_memory` in that company, pinned for decisions and open items that must not be missed. Update or `delete_memory` it once settled. `search_memories` before asking the user something they may have answered before, and before starting work in a company you have not touched this session.

## Routines

`create_routine` runs a plain-language prompt on a schedule with these same tools. Confirm the schedule and the exact prompt text with the user before creating one. `update_routine` pauses or resumes one; resuming re-enables whatever its prompt does, so say what that is. `delete_routine` removes it and its run history.

## Documents and attachments

To keep a supplier invoice or a source document on a record, `create_upload_url` gives a short-lived upload URL and a ready command to upload the local file; then `attach_document` links it to a bill or an invoice. `capture_receipt` and `draft_invoice_from_document` read a document into a draft.

## Docs

`search_docs` answers how-Looped-works questions from the product documentation. Use it before guessing at a setting that has no tool, and point the user to the page it returns.

## Integrations

`list_integrations` and `get_integration` show what is connected. Event sources (Google and Microsoft calendars, Todoist) answer "what's on my calendar" or "what did I finish this week": `list_event_source_containers`, then `list_event_source_events`. `disconnect_integration` only after the user confirms it in plain words; only admins and owners can disconnect. Connecting an integration is done by a person in Manage.

## What the tools cover

| Area | Read | Write |
| --- | --- | --- |
| Track | `list_time_entries`, `get_time_entry`, `list_track_teammates`, `get_time_preferences`, `list_projects`, `search_projects`, `list_project_members`, `list_project_member_rates`, `list_tags`, `list_sections`, `list_todos` | `create_time_entry`, `create_time_entry_for_user`, `update_time_entry`, `update_time_entry_tags`, `delete_time_entry`, `log_time_from_todo`, `create_todo`, `update_todo`, `delete_todo`, `create_project`, `update_project`, `add_project_member`, `remove_project_member`, `update_project_member_rate`, `set_project_member_rate_period`, `create_tag`, `update_tag`, `create_section`, `update_section` |
| Quotes | `list_quotes`, `get_quote`, `get_quote_pdf`, `search_invoice_history` | `create_quote`, `update_quote`, `send_quote`, `set_quote_status`, `convert_quote_to_invoice`, `convert_quote_to_sales_order`, `delete_quote` |
| Invoices | `list_invoices`, `get_invoice`, `get_invoice_pdf`, `get_client_statement`, `list_clients`, `get_client`, `list_customer_balances`, `get_credit_note_pdf`, `get_archived_invoice` | `create_invoice`, `create_invoice_from_time_entries`, `draft_invoice_from_document`, `update_invoice`, `issue_invoice`, `send_invoice`, `mark_invoice_paid`, `reverse_invoice_payment`, `refund_invoice`, `mark_invoice_already_in_books`, `create_credit_note`, `send_credit_note`, `create_client`, `update_client`, `update_invoice_contact`, `create_company_bank_account`, `update_company_bank_account`, `set_team_currency` |
| Financials: bank | `list_bank_imports`, `get_bank_import`, `list_categorisation_rules` | `import_bank_statement`, `categorise_bank_import`, `apply_categorisation_rules`, `review_statement_line`, `bulk_review_statement_lines`, `confirm_statement_line_match`, `split_statement_line`, `reclassify_statement_lines`, `create_categorisation_rule`, `update_categorisation_rule` |
| Financials: purchases | `list_suppliers`, `list_bills`, `get_bill`, `aged_payables`, `list_receipts`, `get_receipt` | `create_supplier`, `create_bill`, `approve_bill`, `pay_bill`, `void_bill`, `convert_bill_to_receipt`, `capture_receipt`, `confirm_receipt`, `match_receipt`, `convert_receipt_to_bill` |
| Financials: ledger | `list_ledger_accounts`, `get_account_ledger`, `list_journal_entries`, `get_trial_balance`, `get_income_statement`, `get_balance_sheet`, `get_cash_flow_statement`, `get_equity_statement`, `export_report`, `list_audit_log`, `list_financial_years` | `create_ledger_account`, `update_ledger_account`, `create_journal_entry`, `reverse_journal_entry`, `recode_journal_entries`, `capture_opening_balances`, `create_financial_year` |
| Financials: assets, loans, FX | `list_fixed_assets`, `list_loans`, `list_exchange_rates`, `get_exchange_rate` | `create_fixed_asset`, `update_fixed_asset`, `dispose_fixed_asset`, `run_depreciation`, `set_loan_terms`, `run_loan_interest`, `set_exchange_rate`, `run_fx_revaluation` |
| Financials: tax and year-end | `get_vat201`, `get_tax_computation`, `get_deferred_tax`, `list_afs_notes`, `get_company_details` | `estimate_provisional_tax`, `post_deferred_tax`, `compile_afs_pack`, `sync_afs_notes`, `create_afs_note`, `update_afs_note`, `reorder_afs_notes`, `finalise_afs_pack`, `update_company_details` |
| Payroll | `list_employees`, `get_employee`, `list_pay_runs`, `get_pay_run`, `get_pay_run_inputs`, `get_payslip`, `list_payroll_runs`, `get_payroll_take_on` | `create_employee`, `set_employee_sdl_exclusion`, `draft_pay_run`, `add_pay_run_item`, `process_pay_run`, `post_payroll_run`, `start_payroll_take_on`, `save_payroll_take_on_employees`, `set_payroll_take_on_opening_ytd`, `set_payroll_take_on_sdl`, `remove_payroll_take_on_employee` |
| Inventory | `list_items`, `get_item`, `get_stock_on_hand`, `list_stock_movements`, `get_reorder_report`, `run_inventory_report`, `list_stock_takes`, `get_stock_take` | `create_item`, `update_item`, `set_opening_stock`, `adjust_stock`, `start_stock_take`, `count_stock_take`, `post_stock_take` |
| Documents |  | `create_upload_url`, `attach_document` |
| Platform | `list_teams`, `get_team_info`, `list_team_members`, `get_company`, `setup_status`, `list_routines`, `get_routine`, `list_memories`, `search_memories`, `get_memory`, `search_docs`, `list_integrations`, `get_integration`, `list_event_source_containers`, `list_event_source_events` | `update_company`, `create_team`, `create_routine`, `update_routine`, `delete_routine`, `save_memory`, `delete_memory`, `disconnect_integration`, `send_feedback` |

## Playbooks

Read `references/playbooks.md` for the step-by-step playbooks before starting any of these jobs. Start with onboarding when the company is new or `setup_status` shows it is not set up:

- Onboard a new company
- Set up Financials from prior-year financial statements
- Import a bank statement
- Review a bank statement upload
- Capture bills and receipts
- Month-end close
- Set up invoicing
- Invoice from logged time and chase what is owed
- Quote, accept and invoice (sales orders)
- VAT201 and provisional tax
- Year-end and the AFS pack
- Payroll: draft, explain and finalise a pay run
- Payroll take-on from Sage

The server also offers playbooks as MCP prompts (onboard-company, setup-from-prior-financials, import-bank-statement, review-bank-statement, setup-invoicing, capture-bills-and-receipts, month-end-close, invoice-from-time, quote-to-invoice, vat-and-provisional-tax, year-end-afs, run-payroll, payroll-take-on). Reply once at the end of each job with what was done, what is still open and any link the user needs to act on.
