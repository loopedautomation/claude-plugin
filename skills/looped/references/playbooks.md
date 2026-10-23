# Looped playbooks

Each playbook is a job the user can hand over whole. Follow the dial from SKILL.md (draft only, everything reversible, or everything including the named irreversible steps). Within a playbook, do the reading up front, present one plan, and carry out everything approved in the same turn.

## Onboard a new company

Goal: get a new company set up in Looped with as little effort from the user as possible. Run this before any other setup playbook when the user is new or `setup_status` shows the company is not set up.

1. See what is missing first: `setup_status`, `get_company`, `get_team_info` and `list_financial_years`. Don't ask for anything these already answer.
2. Ask one short batch of questions, only for what is still missing: what the business does; its website; its country; its financial year end; whether it is VAT registered; which parts it will use (invoicing, accounting, payroll, inventory); and what it uses today (Xero, Sage, Pastel, spreadsheets, nothing). In the same message, ask for any of these it has to hand: the company registration certificate (in South Africa the CIPC CoR14.3), the VAT registration letter and last year's annual financial statements.
3. Read the documents (read the files the user points you at yourself). The registration certificate gives the legal name, registration number, registration date and registered address; the VAT letter gives the VAT number and the date registration took effect; the financial statements give the year end and most of the rest. A value read from a document can be saved as it is.
4. For anything still missing, research it publicly when you can browse the web: the company's own website (address, phone, contact email, what it does) and public registries. Propose each value with where you found it, and save an inferred value only after the user confirms it. If you can't browse, ask.
5. Save the details: `update_company` with the legal name, registration number, country (it sets the ledger currency), physical and postal address, contact email and phone, and, when VAT registered, vatRegistered and the vatNumber. Check the team currency with `get_team_info` and `set_team_currency` (owners only) before any invoice exists if it is wrong. Never overwrite a value with a blank.
6. Financial years: when the user has no prior financial statements to load, `create_financial_year` for the current year from the year end (label FYyyyy, yyyy being the year it ends). When they do, the setup-from-prior-financials playbook creates the years, so leave them to it.
7. Hand over to the playbook for each part they will use instead of repeating its steps: setup-from-prior-financials when they have last year's statements or a trial balance (any system, Xero, Sage and Pastel exports included); import-bank-statement, then review-bank-statement, once they have a bank statement; setup-invoicing when they will invoice. For payroll, `start_payroll_take_on`; for inventory, `create_item` for each item, then `set_opening_stock`. Ask before starting each one; they can be done later.
8. Finish with a short summary: what was saved and where each value came from (document, website, the user), what was left out and why, and the items `setup_status` still lists, each with the next step.

Never invent a value: no registration number, VAT number, address, date or figure that you didn't read from a document, get from the user, or have them confirm.

## Set up Financials from prior-year financial statements

Goal: set up this team's Financials from a prior period's financial statements, trial balance or management accounts that the user points you at. Read the document yourself; do the whole setup in one go and ask only for a value you truly cannot read from it.

1. Read the document: company name, registration number, VAT number; the financial year end and period covered; the balance sheet (assets, liabilities, equity) as at year end, including the notes that break lines down (cost vs accumulated depreciation, related-party balances); the chart of accounts it implies. Statements are in whole currency units — convert to cents.
2. `get_company`, then `update_company` with anything the document gives that is missing or different: the legal name, registration number, VAT registration and number, and the country (it sets the ledger currency). Never overwrite a value with a blank.
3. `list_financial_years`. Create with `create_financial_year` any year the document covers that does not exist (label FYyyyy where yyyy is the year in which it ends), plus the following year if it does not exist, so the team can post current transactions. The document's own year must exist: opening balances are posted into it as the closing position brought forward.
4. `list_ledger_accounts`. For each balance-sheet line in the document, match an existing account by name or code; `create_ledger_account` for the rest, following the existing numbering (1xxx assets, 2xxx liabilities, 3xxx equity, 4xxx income, 5xxx–6xxx expenses). Mark bank accounts with isBankAccount (one per real bank account). Do not create income or expense accounts for prior-period figures — they start the new year at zero.
5. `capture_opening_balances` with asAt = the first day of the year after the document’s year end, one balance per balance-sheet account, retained earnings/owner’s equity from the document, in the ledger currency. The entry lands on the day before (the prior year’s last day), so those balances are the new year’s opening position and not its movement. Contra balances (accumulated depreciation, an overdrawn bank) are negative amounts. The tool balances the entry itself; check its reply and report any difference it had to post. Check the reply’s entryDate is the prior year’s last day; if it is not, stop and report it instead of carrying on.
6. Fixed assets: for property, plant and equipment, `create_fixed_asset` per asset with its cost, useful life and openingAccumulatedDepreciationCents from the notes, so depreciation continues without double-counting what is already in the opening balances.
7. If the document is signed-off annual financial statements for a completed year, tell the user they can close that year in Financials once they have checked the opening position; only a person can close a year, so leave it open.
8. Reply once with a setup summary: company details set, years created, accounts created (count and codes), the opening-balance entry number and total, and anything you could not read from the document.

Never invent figures: if a line is unreadable, leave it out and say so.

## Import a bank statement

Goal: bring a bank statement file into Financials for review. Nothing is posted to the ledger by importing or categorising — only approving a line posts it.

1. `list_ledger_accounts` and find the bank accounts (isBankAccount). One statement upload belongs to exactly one bank account; ask only if you cannot tell which.
2. Read the statement file. If one file holds several accounts (e.g. a combined export with an account or currency column), split it into one CSV per account, keeping the original rows, dates, amounts and running balance untouched. Splitting is the only change you make to the file.
3. Never convert, recalculate or adjust amounts yourself. A statement in another currency (e.g. a EUR account in an AED company) is imported as-is onto a bank account kept in that currency: `create_ledger_account` or `update_ledger_account` with currency (only while nothing is posted to it), and give its opening balance as foreignAmountCents in `capture_opening_balances`. Looped converts each line at the day's rate, and a currency exchange between the company's own accounts at the bank's actual rate.
4. `import_bank_statement` with the statement's raw CSV or OFX text as csv, a fileName and the bankAccountId. Check statementCheck.differenceCents is 0 (the file is internally consistent) and ledgerCheck.differenceCents is 0 (the statement opens where the ledger leaves off, in the account's currency). Report any difference instead of carrying on. When ledgerCheck.suggestion is present (kind rounding_adjustment), the gap is a few cents of rounding on the opening balances: offer its adjusting journal to the user and post it with `create_journal_entry` only if they agree.
5. `categorise_bank_import` on the returned id to get a proposed account for every line.
6. Reply with the date range, transaction count, lines already imported, the opening + net = closing check and the reviewPath. Then follow the bank statement review playbook if the user wants help approving.

## Review a bank statement upload

Goal: help the user review a bank statement upload — check the proposed accounts, turn repeated patterns into bank rules, and approve only what the user agrees to.

1. `list_bank_imports` to find the upload (the newest in review unless they name one), then `get_bank_import` with status OPEN for the lines still to review and their proposals (page through while hasMore is true). Review the whole upload in one pass (every open line, every month), not a month at a time: read every page of open lines before proposing anything. Each proposal's vatTreatment is what approving it will post; check it before approving.
2. Group the lines by what they are (same supplier, same pattern). For each group the user confirms, prefer a bank rule over one-off reviews: `create_categorisation_rule` with descriptionContains (a stable fragment of the description, no reference numbers) and, when it matters, direction or an amount band. After creating rules call `apply_categorisation_rules` on the upload so its open lines pick them up, then `bulk_review_statement_lines` only for what the user asked to approve.
3. Suspense (4999/9999) lines are the ones to resolve. Ask about a group only when the counter-account cannot be inferred from the description or existing accounts; suggest `create_ledger_account` when the chart has no fit. Check the prior year's financial statements for how recurring items were treated (e.g. payments to a shareholder as a related-party receivable rather than a director's loan) and keep the same treatment.
4. Never approve lines the user has not asked you to approve — approving posts to the ledger. Categorising only suggests matches: a line with a suggested match (an invoice payment, bill or payroll run) is checked and confirmed with `confirm_statement_line_match`, never approved to an account, or the money is counted twice. A line that pays an approved bill or matches a waiting receipt is settled with `pay_bill` (statementLineId) or `match_receipt`. Transfers between the team's own bank accounts go to the 'Transfers between accounts' clearing account (both sides, from each statement) so every bank balance stays right; excluding is only for duplicates and lines that are not transactions. When one line pays several things and a document shows the parts (a vehicle finance instalment as capital and interest, a payroll tax payment as PAYE, UIF and SDL), `split_statement_line` into those parts instead of posting the total to one account.
5. Propose the whole upload at once: one list of groups, each with its lines (count, total, date range) and the account or rule you will use, so the user can approve it in one reply. Once they approve, carry out everything they approved in the same turn rather than stopping after the first group.
6. Reply with what was created (rules by name), what was approved (counts and accounts), and what still needs a decision. `list_categorisation_rules` shows the current rules; `update_categorisation_rule` and `delete_categorisation_rule` change them.

## Capture bills and receipts

Goal: get supplier invoices and till slips into Financials from the file the user points you at (`create_upload_url`, then `attach_document`, to keep it on the record), each posted once and matched to the bank once.

1. A supplier invoice with payment terms is a bill. `list_suppliers` or `create_supplier`, then `create_bill` with each line net of VAT in cents, its account and VAT treatment (set vatCents as printed when it differs from the standard rate). `approve_bill` posts it to accounts payable; `pay_bill` records payment, with statementLineId whenever the payment is on an imported statement. `aged_payables` shows what is owed. `void_bill` can't be undone.
2. A till slip or card receipt (spend paid on the spot) is a receipt. `capture_receipt` from the file or the slip's text; Looped reads the merchant, date, total and VAT and suggests an account. Check possibleDuplicateOf. Ask how it was paid when you can't tell, then `confirm_receipt`. A receipt paid from the business account waits for its bank line; `match_receipt` posts it when the line arrives.
3. `convert_bill_to_receipt` and `convert_receipt_to_bill` move a draft between the two. `attach_document` adds a file to an existing bill or invoice.
4. A foreign-currency bill converts at its date's rate on approval; `set_exchange_rate` first when the bank's actual rate should be used.
5. Reply with the bills and receipts captured (supplier, total, account), which are approved, paid or waiting for their bank line, and any possible duplicates.

## Month-end close

Goal: everything for the month is in the books, the reports agree with the bank, and the user gets a short management view.

1. `search_memories` for open items and decisions in this company. `setup_status` for blocked postings; relay each one's message.
2. Bank: for each bank account, check the month's statement is imported (`list_bank_imports`). Import any the user gives you, then follow the bank statement review playbook. Report any account with no statement for the month.
3. Purchases: approve and pay the bills the user confirmed, match waiting receipts, and review `aged_payables`.
4. Sales: `list_invoices` for the month. Issue the drafts the user approved; record payments that are on the bank (`mark_invoice_paid` when the statement line isn't already matched to the invoice); list overdue invoices with `list_customer_balances`.
5. Recurring postings: `run_depreciation` for the month; `run_loan_interest` for loans without an imported lender statement (preview first, and use the lender's figures as overrides when the user has them); `run_fx_revaluation` at month end when any account is in another currency (dryRun first). Post them when the user asked you to post; otherwise show the previews.
6. Checks: `get_trial_balance` at month end, each bank balance against its statement's closing balance, suspense accounts at zero, and the VAT control movement against `get_vat201` for the period.
7. Reply with the month's `get_income_statement`, the bank and suspense checks, what was posted, what is still open and anything only a person can do. `save_memory` the open items, pinned.

## Set up invoicing

Goal: get this team ready to invoice: currency, who the invoice is from, how to be paid, the clients, and a first invoice checked before anything is sent.

1. Currency first. `get_team_info`. If the team currency is wrong, `set_team_currency` (owners only) before any invoice exists: it cannot change once the team has invoices. This is the Track and Invoices currency; Financials keeps its own ledger currency from the company's country.
2. Who the invoice is from. `get_company`, then `update_company` with the physical address, the contact email and phone, and, if the company is VAT registered, vatRegistered and its vatNumber. The contact person named on invoices is `update_invoice_contact` (name).
3. How to get paid. `list_company_bank_accounts`, then `create_company_bank_account` (the first becomes the default printed on invoices) or `update_company_bank_account`.
4. Clients. `list_clients`; `create_client` for each new client with its name, email, address, currency and country (an invoice uses the client's currency unless given one).
5. First invoice, only when the user wants one. `create_invoice`, or `create_invoice_from_time_entries` when they bill from Track (the client, the project and the date range). `get_invoice_pdf` and show the returned download link for review. `send_invoice` only after the user confirms: it emails the client.
6. The logo, invoice numbering and payment terms have no tools: point the user to Invoices settings for them rather than skipping them silently.
7. Reply once with what is set (currency, address, contact, VAT, bank details), the clients created, any invoice drafted (number and total) and what the user still has to do in settings.

Never guess an address, bank details or a VAT number: ask for what the user has not given.

## Invoice from logged time and chase what is owed

Goal: bill a client for the time logged in Track, then keep track of who owes what.

1. Resolve the client (`list_clients`) and the projects (`search_projects`). Check rates with `list_project_member_rates`: `create_invoice_from_time_entries` fails with missing_rate when a project has none, so ask for one and pass hourlyRate. Set includeTeam to bill everyone's time (it needs a Track role that sees it). Check taxPercentage against the company's VAT registration.
2. Call it once per client for the period. Show the hours, line items and total from the reply, with the `get_invoice_pdf` link.
3. Once the user approves the draft, `issue_invoice` (it is owed from then and posts to the books). `send_invoice` only when the user asks for it to be sent: it emails the client. If the user emailed it themselves, `issue_invoice` with sentAt.
4. Chasing: `list_customer_balances` for who owes what, `get_client_statement` for a statement to send, `list_invoices` for what is overdue. Draft the reminder text in your reply; Looped sends invoices and statements, and the user sends anything else from their own mail.
5. Corrections: `update_invoice` while it is a draft; `create_credit_note` against an issued one (`send_credit_note` when asked); `reverse_invoice_payment` or `refund_invoice` for payments. A PERIOD_LOCKED error means the date falls in a finalised VAT201 or a closed year: change the date or ask the user to reopen the period.

## Quote, accept and invoice (sales orders)

Goal: quote a client, record their answer and turn an accepted quote into an invoice or a sales order.

1. `list_clients` (`create_client` when new), then `create_quote` as a draft with its lines in major units; the currency follows the client. Show the user `get_quote_pdf` before anything is sent.
2. `send_quote` only when the user asks: it emails the client and marks the quote sent. `update_quote` changes a quote until it is accepted or converted.
3. When the client answers, `set_quote_status` with ACCEPTED or REJECTED.
4. An accepted quote becomes an issued invoice with `convert_quote_to_invoice` (only when the user says to invoice it; it is owed from then and posts to the books), or a sales order with `convert_quote_to_sales_order` when the work is delivered or invoiced in parts. Invoicing a sales order in parts happens in Invoices.
5. `list_quotes` finds quotes by client, status and date; `search_invoice_history` with type QUOTE or SALES_ORDER finds them by number across years. `delete_quote` can't be undone.

## VAT201 and provisional tax

Goal: give the user the figures for the VAT201 and the IRP6 provisional tax return. Looped prepares them; the user files them.

1. Before the VAT201, check every bank line in the period is reviewed: unreviewed lines understate the return.
2. `get_vat201` for the calendar year gives each VAT period's output, input and net VAT and the efiling figures to type into eFiling (supply fields in whole rands, VAT fields with cents). Present them as a table. Only a person finalises or reopens a VAT201 and files it.
3. Zero-rated and exempt supplies must carry that treatment on their posting to appear in fields 2 and 3: fix ones posted as standard or no VAT with `reclassify_statement_lines` or `recode_journal_entries`.
4. `get_tax_computation` for the financial year shows accounting profit, add-backs, allowances, taxable income and tax. `estimate_provisional_tax` computes and saves an IRP6 estimate for period 1 or 2 from it: set useSbcRates when the company is a small business corporation and alreadyPaidCents for period 2. Rerunning it replaces the saved estimate. Show the computation and the estimate side by side.

## Year-end and the AFS pack

Goal: close the books for the year and compile the annual financial statements (AFS) pack.

1. Close the last month with the month-end playbook. Then `run_depreciation` for any remaining period, `run_fx_revaluation` at year end, and `get_tax_computation`.
2. `get_deferred_tax` and show it; `post_deferred_tax` once the user agrees (rerunning posts only the further movement). Set recogniseAssessedLoss only when the user says to.
3. `compile_afs_pack` for the year with assurance (ask when it isn't set: South African packs state on the cover whether they are audited, independently reviewed or compiled). Set from and to for an interim pack, which comes back without being saved. `sync_afs_notes` brings unedited notes in line with the ledger; `create_afs_note`, `update_afs_note` and `reorder_afs_notes` for the narrative notes (accounting policies, related parties, events after the reporting date), in the terms of the company's reporting framework (`get_company_details`) and from its actual figures. Recompiling a draft pack is safe.
4. Present the income statement, balance sheet, statement of changes in equity and cash flow from the pack with the comparatives, the list of notes and any warnings to resolve.
5. A person closes the year in Financials. `finalise_afs_pack` freezes the pack of a closed year and can't be undone: only when the user asks, after the year is closed.
6. `export_report` (trial balance, general ledger or journal as CSV or xlsx) for the accountant; present the link.

## Payroll: draft, explain and finalise a pay run

Goal: draft the month's pay run, explain it, and finalise it only when it is clean and the user has said yes.

1. `list_pay_runs` for the cycle and its runs. `draft_pay_run` drafts the current open period (or returns the existing draft) with hours from Track and unpaid leave. Set cycleId when the company has several cycles.
2. `get_pay_run` and `get_pay_run_inputs` to explain the run: gross, PAYE, UIF, SDL and net per employee, where the hours came from, and what changed since the last run. `get_payslip` for one employee. Keep identity, tax and bank numbers masked.
3. `add_pay_run_item` for once-off items the user asks for (bonus, deduction, reimbursement); each one makes the run unclean. Refresh the draft with useLatestInputs only when the user asks for Track's current figures, and say what it replaces first.
4. Finalise only with `process_pay_run`, only a clean run, and only after showing the employee count and the gross, PAYE and net totals and getting an explicit yes. Pass that net as expectedNetCents. When the run has changes, give the reviewUrl: a person ticks each change in Payroll. On TOTALS_CHANGED show the new totals and ask again. Reversing a run, sending payslips and filing are done by a person.
5. `post_payroll_run` posts a finalised run's journal to Financials when it isn't posted yet; `list_payroll_runs` shows which are.
6. New starters: `create_employee` with the basic fields, then give its completeDetailsUrl. `set_employee_sdl_exclusion` when the company is under the SDL threshold.

## Payroll take-on from Sage

Goal: bring a payroll over from Sage partway through the tax year, so Looped pays from the cut-over month with the right year-to-date figures.

1. `start_payroll_take_on` with cutOverMonth, the first month Looped pays. It returns the take-on and its reviewUrl.
2. Read the payroll reports the user gives you. `save_payroll_take_on_employees` with the allowed fields, `set_payroll_take_on_opening_ytd` for the year-to-date figures before the cut-over, `set_payroll_take_on_sdl`. `remove_payroll_take_on_employee` for someone who shouldn't come across. `get_payroll_take_on` shows the draft.
3. Nothing in the take-on affects a pay run until a person with full Payroll access confirms it on the take-on screen. Say so, with the link, and never claim the draft activated employees or changed a run.
