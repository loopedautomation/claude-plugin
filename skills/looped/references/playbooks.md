# Looped playbooks

Each playbook is a job the user can hand over whole. Follow the dial from SKILL.md (draft only, everything reversible, or everything including the named irreversible steps). Within a playbook, do the reading up front, present one plan, and carry out everything approved in the same turn.

## Set up Financials from prior-year financial statements

Goal: set up this company's Financials from a prior period's financial statements, trial balance or management accounts the user points at. Read the document; do the whole setup in one go and ask only for a value you cannot read from it.

1. Read the document: company name, registration number, VAT number; financial year end and period covered; the balance sheet (assets, liabilities, equity) at year end, including the notes that break lines down (cost vs accumulated depreciation, related-party balances); the chart of accounts it implies. Statements are in whole currency units; convert to cents.
2. `get_company`, then `update_company` with anything the document gives that is missing or different: legal name, registration number, VAT registration and number, and the country (it sets the ledger currency). Never overwrite a value with a blank.
3. `list_financial_years`. `create_financial_year` for any year the document covers that does not exist (label FYyyyy where yyyy is the year it ends in), plus the following year so the company can post current transactions. The document's own year must exist: opening balances post into it as the closing position brought forward.
4. `list_ledger_accounts`. Match each balance-sheet line to an existing account by name or code; `create_ledger_account` for the rest, following the existing numbering (1xxx assets, 2xxx liabilities, 3xxx equity, 4xxx income, 5xxx to 6xxx expenses). Mark bank accounts with `isBankAccount`, one per real bank account. Do not create income or expense accounts for prior-period figures; they start the new year at zero.
5. `capture_opening_balances` with `asAt` = the first day of the year after the document's year end, one balance per balance-sheet account, retained earnings or owner's equity from the document, in the ledger currency. The entry lands on the prior year's last day. Contra balances (accumulated depreciation, an overdrawn bank) are negative. The tool balances the entry itself; check its reply and report any difference it had to post. Check `entryDate` is the prior year's last day; if it is not, stop and report it.
6. Fixed assets: `create_fixed_asset` per asset with cost, useful life and `openingAccumulatedDepreciationCents` from the notes, so depreciation continues without double-counting the opening balances.
7. If the document is signed-off annual financial statements for a completed year, tell the user they can close that year in Financials once they have checked the opening position. Only a person can close a year.
8. Reply once with: company details set, years created, accounts created (count and codes), the opening-balance entry number and total, and anything you could not read. Never invent figures; leave an unreadable line out and say so.

## Import a bank statement

Goal: bring a bank statement file into Financials for review. Importing and categorising post nothing; approving a line posts it.

1. `list_ledger_accounts` and find the bank accounts (`isBankAccount`). One upload belongs to exactly one bank account; ask only if you cannot tell which.
2. Read the statement file. If one file holds several accounts, split it into one CSV per account, keeping the original rows, dates, amounts and running balance untouched. Splitting is the only change you make to the file.
3. Never convert, recalculate or adjust amounts yourself. A statement in another currency is imported as-is onto a bank account kept in that currency: `create_ledger_account` or `update_ledger_account` with `currency` (only while nothing is posted to it), and give its opening balance as `foreignAmountCents` in `capture_opening_balances`. Looped converts each line at the day's rate.
4. `import_bank_statement` with the raw CSV or OFX text as `csv`, a `fileName` and the `bankAccountId`. Check `statementCheck.differenceCents` is 0 (the file is internally consistent) and `ledgerCheck.differenceCents` is 0 (the statement opens where the ledger leaves off). Report any difference instead of carrying on. When `ledgerCheck.suggestion` is present (kind `rounding_adjustment`), the gap is a few cents of rounding on opening balances: post its adjusting journal with `create_journal_entry` under the reversible dial, otherwise offer it.
5. `categorise_bank_import` on the returned id to get a proposed account for every line.
6. Reply with the date range, transaction count, lines already imported, the opening + net = closing check and the `reviewPath`, then move into the review playbook.

## Review a bank statement upload

Goal: check the proposed accounts, turn repeated patterns into bank rules, and approve what the dial allows.

1. `list_bank_imports` to find the upload (the newest in review unless they name one), then `get_bank_import` with status OPEN for the lines still to review and their proposals. Page through while `hasMore` is true. Read every open line, every month, before proposing anything. Each proposal's `vatTreatment` is what approving posts; check it, and check it against the company's VAT registration date (lines before it are NO_VAT).
2. Group the lines by what they are (same supplier, same pattern). For each group, prefer a bank rule over one-off reviews: `create_categorisation_rule` with `descriptionContains` (a stable fragment, no reference numbers) and, when it matters, direction or an amount band. After creating rules call `apply_categorisation_rules` on the upload, then `bulk_review_statement_lines` for the lines being approved.
3. Suspense (4999/9999) lines are the ones to resolve. Ask about a group only when the counter-account cannot be inferred from the description, existing accounts, bills, receipts or saved memories; suggest `create_ledger_account` when the chart has no fit. Keep the treatment the prior year's statements used for recurring items.
4. Lines that pay a bill or match a receipt: settle them through `pay_bill` with `statementLineId` or `match_receipt`, so the bank is counted once. When one line pays several things and a document shows the parts (a vehicle finance instalment as capital and interest, a payroll tax payment as PAYE, UIF and SDL), `split_statement_line` into those parts. Transfers between the company's own accounts go to the Transfers between accounts clearing account from both statements. Exclude only duplicates and non-transactions.
5. Under the draft dial, present the groups and approve nothing. Under the reversible dial, approve every group whose account you can determine with confidence, leave the rest in suspense and list them with your best guess. Under any dial, show one list of groups (count, total, date range, account or rule) in the reply.
6. Reply with rules created (by name), lines approved (counts and accounts) and what still needs a decision. `list_categorisation_rules`, `update_categorisation_rule` and `delete_categorisation_rule` manage the rules afterwards.

## Capture bills and receipts

- A supplier invoice with payment terms is a bill. `list_suppliers` or `create_supplier`, then `create_bill` with each line net of VAT in cents, its account and VAT treatment (STANDARD is 15% in ZA; set `vatCents` as printed when it differs). `approve_bill` posts it to Accounts Payable; `pay_bill` records payment, with `statementLineId` whenever the payment is on a statement. `aged_payables` shows what is owed. `void_bill` is irreversible.
- A till slip or card receipt (spend paid on the spot) is a receipt. `capture_receipt` from an attachment, an upload key or the slip's text; the server reads merchant, date, total and VAT and suggests an account. Check `possibleDuplicateOf`. Ask how it was paid when you cannot tell, then `confirm_receipt`. A receipt paid from the business account waits for its bank line; `match_receipt` posts it when the line arrives.
- `convert_bill_to_receipt` and `convert_receipt_to_bill` move a draft between the two. `attach_document` adds a file to an existing bill, line or journal.
- Foreign-currency bills convert at their date's rate on approval; `set_exchange_rate` when the bank's actual rate should be used.

## Month-end close

Goal: everything for the month is in the books, the reports agree with the bank, and the user gets a short management view.

1. `search_memories` for open items and decisions in this company. `setup_status` for blocked postings; relay each.
2. Bank: for each bank account, confirm the month's statement is imported (`list_bank_imports`). Import what the user attached, then run the review playbook. Report any account with no statement for the month.
3. Purchases: approve and pay bills the user confirmed, match waiting receipts, review `aged_payables`.
4. Sales: `list_invoices` for the month. Issue drafts the user approved; record payments that appear on the bank (`mark_invoice_paid` when the statement line is not already matched to the invoice); list overdue invoices with `list_customer_balances`.
5. Recurring postings: `run_depreciation` for the month; `run_loan_interest` for loans without an imported lender statement (preview first, use the lender's figures as overrides when the user has them); `run_fx_revaluation` at month end when any account is in another currency (`dryRun` first). Under the reversible dial, post them and report the amounts; under the draft dial, show the previews.
6. Checks: `get_trial_balance` at month end, bank balances against the statements' closing balances, suspense accounts at zero, VAT control movement against `get_vat201` for the period.
7. Reply with the month's income statement (`get_income_statement`), the bank and suspense checks, what was posted, what is still open and anything that needs a person. `save_memory` the open items, pinned.

## Set up invoicing

Goal: currency, who the invoice is from, how to be paid, the clients, and a first invoice checked before anything is sent.

1. Currency first. `get_team_info`. If the team currency is wrong, `set_team_currency` (owners only) before any invoice exists; it cannot change once the team has invoices. Financials keeps its own ledger currency.
2. Who the invoice is from. `get_company`, then `update_company` with the physical address (required before any invoice can be created), contact email and phone, and VAT registration and number when registered. The contact person named on invoices is `update_invoice_contact`.
3. How to get paid. `list_company_bank_accounts`, then `create_company_bank_account` (the first becomes the default printed on invoices) or `update_company_bank_account`.
4. Clients. `list_clients`; `create_client` for each new client with name, email, address, currency and country.
5. First invoice, when the user wants one. `create_invoice`, or `create_invoice_from_time_entries` when billing from Track. `get_invoice_pdf` and show the link. `issue_invoice` under the reversible dial if the user approved the draft; `send_invoice` only when the user asked to send it.
6. Logo, invoice numbering and payment terms have no tools: point the user to Invoices settings rather than skipping them silently.
7. Reply once with what is set, the clients created, any invoice drafted (number and total) and what remains in settings. Never guess an address or VAT registration.

## Invoice from logged time and chase what is owed

1. Resolve the client (`list_clients`) and projects (`search_projects`). Check rates with `list_project_member_rates`; `create_invoice_from_time_entries` fails with `missing_rate` when a project has none, so ask for one and set `hourlyRate`. Set `includeTeam: true` to bill everyone's time (needs a Track role that sees it). Check `taxPercentage` against the company's VAT registration.
2. Call it once per client for the period. Show the hours, line items and total from the reply, with the PDF link.
3. Under the reversible dial, `issue_invoice` once the user has approved the draft (it counts as owed and posts to the books). `send_invoice` only on an explicit ask; it emails the client. If the user emailed it themselves, `issue_invoice` with `sentAt`.
4. Chasing: `list_customer_balances` for who owes what, `get_client_statement` for a statement to send, `list_invoices` filtered to overdue. Draft the chaser text in the reply; Looped sends statements and invoices, and anything else the user sends from their own mail.
5. Corrections: `update_invoice` while draft; `create_credit_note` (then `send_credit_note` on request) against an issued one; `reverse_invoice_payment` or `refund_invoice` for payments. A `PERIOD_LOCKED` error means the date falls in a finalised VAT201 or closed year: redate it or have the user reopen the period.

## VAT201 and provisional tax

- `get_vat201` for the calendar year gives each VAT period's output, input and net VAT plus the `efiling` figures to type into eFiling (supply fields in whole rands, VAT fields with cents). Present those figures as a table. Only a person finalises or reopens a VAT201 and files it. Before presenting, check the period's bank lines are all reviewed; unreviewed lines understate the return.
- Zero-rated and exempt supplies must carry their treatment on the posting to appear in fields 2 and 3; reclassify with `reclassify_statement_lines` or `recode_journal_entries` when they were posted as standard or no VAT.
- `get_tax_computation` for the financial year shows accounting profit, add-backs, allowances (automatic wear and tear), taxable income and tax. `estimate_provisional_tax` computes and saves an IRP6 estimate for period 1 or 2 from it; set `useSbcRates` when the company is a Small Business Corporation and `alreadyPaidCents` for period 2. Rerunning it replaces the saved estimate, so under the reversible dial compute and save it, then show the computation and the estimate side by side. The user files the IRP6 themselves.

## Year-end and the AFS pack

1. Close the last month with the month-end playbook. Then `run_depreciation` for any remaining period, `run_fx_revaluation` at year end, and `get_tax_computation`.
2. `get_deferred_tax`; show it and, under the reversible dial, `post_deferred_tax` (rerunning posts only the further movement). Set `recogniseAssessedLoss` only when the user says to.
3. `compile_afs_pack` for the year with `assurance` (ask when it is not set; South Africa's Companies Act requires the statement on the cover). Set `from` and `to` for an interim pack, which is returned without being saved. `sync_afs_notes` pulls the notes from the ledger; `create_afs_note`, `update_afs_note` and `reorder_afs_notes` for the narrative notes (accounting policies, related parties, events after the reporting date). Recompiling a draft pack is safe.
4. Present the income statement, balance sheet, equity and cash flow from the pack with the comparatives, and the list of notes.
5. Closing the year happens in Financials by a person. `finalise_afs_pack` freezes the pack of a closed year and is irreversible: only on an explicit ask, after the year is closed.
6. `export_report` (trial balance, general ledger or journal as CSV or xlsx) for the accountant; present the link.

## Payroll: draft, explain and finalise a pay run

1. `list_pay_runs` for the cycle and its runs. `draft_pay_run` drafts the current open period (or returns the existing draft) and reads hours from Track and unpaid leave from Leave. Set `cycleId` when the company has several cycles.
2. `get_pay_run` and `get_pay_run_inputs` to explain the run: gross, PAYE, UIF, SDL and net per employee, where hours and leave came from, and what changed since last run. `get_payslip` for one employee. Distinguish the draft's saved inputs from Track's current figures. Keep identity, tax and bank numbers masked.
3. `add_pay_run_item` for once-off items the user asks for (bonus, deduction, reimbursement); each one makes the run unclean.
4. Refreshing the draft with `useLatestInputs: true` replaces the saved inputs with Track's and Leave's current figures. Only when the user asked for current figures, and say what it replaces first.
5. Finalising: only `process_pay_run`, only a clean run (`clean: true`), and only after showing the user the employee count, gross, PAYE and net totals and getting an explicit yes in this chat. Set that net as `expectedNetCents`. On `RUN_HAS_CHANGES` give the `reviewUrl` so a person ticks each change in Payroll; on `TOTALS_CHANGED` show the new totals and ask again; on `APPROVAL_REQUIRED` say who has to approve it. Reversing a run, sending payslips and producing filings are done by a person.
6. `post_payroll_run` posts a finalised run's journal to Financials when it is not already posted; `list_payroll_runs` shows which are.
7. New starters: `create_employee` with the basic fields, then give the `completeDetailsUrl`. `set_employee_sdl_exclusion` when the company is under the SDL threshold.

## Payroll take-on from Sage

1. `start_payroll_take_on` with `cutOverMonth` (the first month Looped pays). It returns the open take-on and its `reviewUrl`.
2. Read the Sage reports the user attached. `save_payroll_take_on_employees` with the allowed fields, `set_payroll_take_on_opening_ytd` for March-to-cutover year-to-date figures, `set_payroll_take_on_sdl`. `remove_payroll_take_on_employee` for someone who should not come across. `get_payroll_take_on` shows the draft.
3. Nothing in the take-on affects a pay run until a person with Full Payroll access confirms it on the take-on screen. Say so, with the link. Never claim the draft activated employees or changed a run.
