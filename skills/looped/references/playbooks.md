# Looped playbooks

Each playbook is a job the user can hand over whole. Follow the dial from SKILL.md (draft only, everything reversible, or everything including the named irreversible steps). Within a playbook, do the reading up front, present one plan, and carry out everything approved in the same turn.

## Set up Financials from prior-year financial statements

Goal: set up this company's Financials from a prior period's financial statements, trial balance or management accounts the user points at. Read the document; do the whole setup in one go and ask only for a value you cannot read from it.

1. Read the document: company name, registration number, VAT number; financial year end and period covered; the balance sheet (assets, liabilities, equity) at year end, including the notes that break lines down (cost vs accumulated depreciation, related-party balances); the chart of accounts it implies. Statements are in whole currency units; convert to cents.
2. `company_read` action `get`, then `company_write` action `update` with anything the document gives that is missing or different: legal name, registration number, VAT registration and number, and the country (it sets the ledger currency). Never overwrite a value with a blank.
3. `financial_years_read` action `list`. `financial_years_write` action `create` for any year the document covers that does not exist (label FYyyyy where yyyy is the year it ends in), plus the following year so the company can post current transactions. The document's own year must exist: opening balances post into it as the closing position brought forward.
4. `accounts_read` action `list`. Match each balance-sheet line to an existing account by name or code; `accounts_write` action `create` for the rest, following the existing numbering (1xxx assets, 2xxx liabilities, 3xxx equity, 4xxx income, 5xxx to 6xxx expenses). Mark bank accounts with `isBankAccount`, one per real bank account. Do not create income or expense accounts for prior-period figures; they start the new year at zero.
5. `financial_years_write` action `opening_balances` with `asAt` = the first day of the year after the document's year end, one balance per balance-sheet account, retained earnings or owner's equity from the document, in the ledger currency. The entry lands on the prior year's last day. Contra balances (accumulated depreciation, an overdrawn bank) are negative. The tool balances the entry itself; check its reply and report any difference it had to post. Check `entryDate` is the prior year's last day; if it is not, stop and report it.
6. Fixed assets: `fixed_assets_write` action `create` per asset with cost, useful life and `openingAccumulatedDepreciationCents` from the notes, so depreciation continues without double-counting the opening balances.
7. If the document is signed-off annual financial statements for a completed year, tell the user they can close that year in Financials once they have checked the opening position. Only a person can close a year.
8. Reply once with: company details set, years created, accounts created (count and codes), the opening-balance entry number and total, and anything you could not read. Never invent figures; leave an unreadable line out and say so.

## Import a bank statement

Goal: bring a bank statement file into Financials for review. Importing and categorising post nothing; approving a line posts it.

1. `accounts_read` action `list` and find the bank accounts (`isBankAccount`). One upload belongs to exactly one bank account; ask only if you cannot tell which.
2. Read the statement file. If one file holds several accounts, split it into one CSV per account, keeping the original rows, dates, amounts and running balance untouched. Splitting is the only change you make to the file.
3. Never convert, recalculate or adjust amounts yourself. A statement in another currency is imported as-is onto a bank account kept in that currency: `accounts_write` action `create` or `accounts_write` action `update` with `currency` (only while nothing is posted to it), and give its opening balance as `foreignAmountCents` in `financial_years_write` action `opening_balances`. Looped converts each line at the day's rate.
4. `bank_imports_write` action `import` with the raw CSV or OFX text as `csv`, a `fileName` and the `bankAccountId`. Check `statementCheck.differenceCents` is 0 (the file is internally consistent) and `ledgerCheck.differenceCents` is 0 (the statement opens where the ledger leaves off). Report any difference instead of carrying on. When `ledgerCheck.suggestion` is present (kind `rounding_adjustment`), the gap is a few cents of rounding on opening balances: post its adjusting journal with `journal_entries_write` action `create` under the reversible dial, otherwise offer it.
5. `bank_imports_write` action `categorise` on the returned id to get a proposed account for every line.
6. Reply with the date range, transaction count, lines already imported, the opening + net = closing check and the `reviewPath`, then move into the review playbook.

## Review a bank statement upload

Goal: check the proposed accounts, turn repeated patterns into bank rules, and approve what the dial allows.

1. `bank_imports_read` action `list` to find the upload (the newest in review unless they name one), then `bank_imports_read` action `get` with status OPEN for the lines still to review and their proposals. Page through while `hasMore` is true. Read every open line, every month, before proposing anything. Each proposal's `vatTreatment` is what approving posts; check it, and check it against the company's VAT registration date (lines before it are NO_VAT).
2. Group the lines by what they are (same supplier, same pattern). For each group, prefer a bank rule over one-off reviews: `categorisation_rules_write` action `create` with `descriptionContains` (a stable fragment, no reference numbers) and, when it matters, direction or an amount band. After creating rules call `bank_imports_write` action `apply_rules` on the upload, then `bank_imports_write` action `bulk_review` for the lines being approved.
3. Suspense (4999/9999) lines are the ones to resolve. Ask about a group only when the counter-account cannot be inferred from the description, existing accounts, bills, receipts or saved memories; suggest `accounts_write` action `create` when the chart has no fit. Keep the treatment the prior year's statements used for recurring items.
4. Lines that pay a bill or match a receipt: settle them through `bills_write` action `pay` with `statementLineId` or `receipts_write` action `match`, so the bank is counted once. When one line pays several things and a document shows the parts (a vehicle finance instalment as capital and interest, a payroll tax payment as PAYE, UIF and SDL), `bank_imports_write` action `split` into those parts. Transfers between the company's own accounts go to the Transfers between accounts clearing account from both statements. Exclude only duplicates and non-transactions.
5. Under the draft dial, present the groups and approve nothing. Under the reversible dial, approve every group whose account you can determine with confidence, leave the rest in suspense and list them with your best guess. Under any dial, show one list of groups (count, total, date range, account or rule) in the reply.
6. Reply with rules created (by name), lines approved (counts and accounts) and what still needs a decision. `categorisation_rules_read` action `list`, `categorisation_rules_write` action `update` and `categorisation_rules_delete` action `delete` manage the rules afterwards.

## Capture bills and receipts

- A supplier invoice with payment terms is a bill. `suppliers_read` action `list` or `suppliers_write` action `create`, then `bills_write` action `create` with each line net of VAT in cents, its account and VAT treatment (STANDARD is 15% in ZA; set `vatCents` as printed when it differs). `bills_write` action `approve` posts it to Accounts Payable; `bills_write` action `pay` records payment, with `statementLineId` whenever the payment is on a statement. `bills_read` action `aged_payables` shows what is owed. `bills_write` action `void` is irreversible.
- A till slip or card receipt (spend paid on the spot) is a receipt. `receipts_write` action `capture` from an attachment, an upload key or the slip's text; the server reads merchant, date, total and VAT and suggests an account. Check `possibleDuplicateOf`. Ask how it was paid when you cannot tell, then `receipts_write` action `confirm`. A receipt paid from the business account waits for its bank line; `receipts_write` action `match` posts it when the line arrives.
- `bills_write` action `convert_to_receipt` and `receipts_write` action `convert_to_bill` move a draft between the two. `attach_document` adds a file to an existing bill, line or journal.
- Foreign-currency bills convert at their date's rate on approval; `exchange_rates_write` action `set` when the bank's actual rate should be used.

## Month-end close

Goal: everything for the month is in the books, the reports agree with the bank, and the user gets a short management view.

1. `memories_read` action `search` for open items and decisions in this company. `financial_reports_read` action `setup_status` for blocked postings; relay each.
2. Bank: for each bank account, confirm the month's statement is imported (`bank_imports_read` action `list`). Import what the user attached, then run the review playbook. Report any account with no statement for the month.
3. Purchases: approve and pay bills the user confirmed, match waiting receipts, review `bills_read` action `aged_payables`.
4. Sales: `invoices_read` action `list` for the month. Issue drafts the user approved; record payments that appear on the bank (`invoices_write` action `mark_paid` when the statement line is not already matched to the invoice); list overdue invoices with `accounting_reports_read` action `customer_balances`.
5. Recurring postings: `fixed_assets_write` action `depreciate` for the month; `loans_write` action `run_interest` for loans without an imported lender statement (preview first, use the lender's figures as overrides when the user has them); `journal_entries_write` action `revalue_fx` at month end when any account is in another currency (`dryRun` first). Under the reversible dial, post them and report the amounts; under the draft dial, show the previews.
6. Checks: `financial_reports_read` action `trial_balance` at month end, bank balances against the statements' closing balances, suspense accounts at zero, VAT control movement against `accounting_reports_read` action `vat201` for the period.
7. Reply with the month's income statement (`financial_reports_read` action `income_statement`), the bank and suspense checks, what was posted, what is still open and anything that needs a person. `memories_write` action `save` the open items, pinned.

## Set up invoicing

Goal: currency, who the invoice is from, how to be paid, the clients, and a first invoice checked before anything is sent.

1. Currency first. `get_team_info`. If the team currency is wrong, `set_team_currency` (owners only) before any invoice exists; it cannot change once the team has invoices. Financials keeps its own ledger currency.
2. Who the invoice is from. `company_read` action `get`, then `company_write` action `update` with the physical address (required before any invoice can be created), contact email and phone, and VAT registration and number when registered. The contact person named on invoices is `invoices_write` action `update_contact`.
3. How to get paid. `company_read` action `bank_accounts`, then `company_write` action `create_bank_account` (the first becomes the default printed on invoices) or `company_write` action `update_bank_account`.
4. Clients. `clients_read` action `list`; `clients_write` action `create` for each new client with name, email, address, currency and country.
5. First invoice, when the user wants one. `invoices_write` action `create`, or `invoices_write` action `from_time_entries` when billing from Track. `invoices_read` action `pdf` and show the link. `invoices_write` action `issue` under the reversible dial if the user approved the draft; `invoices_write` action `send` only when the user asked to send it.
6. Logo, invoice numbering and payment terms have no tools: point the user to Invoices settings rather than skipping them silently.
7. Reply once with what is set, the clients created, any invoice drafted (number and total) and what remains in settings. Never guess an address or VAT registration.

## Invoice from logged time and chase what is owed

1. Resolve the client (`clients_read` action `list`) and projects (`projects_read` action `search`). Check rates with `projects_read` action `member_rates`; `invoices_write` action `from_time_entries` fails with `missing_rate` when a project has none, so ask for one and set `hourlyRate`. Set `includeTeam: true` to bill everyone's time (needs a Track role that sees it). Check `taxPercentage` against the company's VAT registration.
2. Call it once per client for the period. Show the hours, line items and total from the reply, with the PDF link.
3. Under the reversible dial, `invoices_write` action `issue` once the user has approved the draft (it counts as owed and posts to the books). `invoices_write` action `send` only on an explicit ask; it emails the client. If the user emailed it themselves, `invoices_write` action `issue` with `sentAt`.
4. Chasing: `accounting_reports_read` action `customer_balances` for who owes what, `invoices_read` action `client_statement` for a statement to send, `invoices_read` action `list` filtered to overdue. Draft the chaser text in the reply; Looped sends statements and invoices, and anything else the user sends from their own mail.
5. Corrections: `invoices_write` action `update` while draft; `credit_notes_write` action `create` (then `credit_notes_write` action `send` on request) against an issued one; `invoices_write` action `reverse_payment` or `invoices_write` action `refund` for payments. A `PERIOD_LOCKED` error means the date falls in a finalised VAT201 or closed year: redate it or have the user reopen the period.

## VAT201 and provisional tax

- `accounting_reports_read` action `vat201` for the calendar year gives each VAT period's output, input and net VAT plus the `efiling` figures to type into eFiling (supply fields in whole rands, VAT fields with cents). Present those figures as a table. Only a person finalises or reopens a VAT201 and files it. Before presenting, check the period's bank lines are all reviewed; unreviewed lines understate the return.
- Zero-rated and exempt supplies must carry their treatment on the posting to appear in fields 2 and 3; reclassify with `bank_imports_write` action `reclassify` or `journal_entries_write` action `recode` when they were posted as standard or no VAT.
- `tax_read` action `computation` for the financial year shows accounting profit, add-backs, allowances (automatic wear and tear), taxable income and tax. `tax_write` action `estimate_provisional` computes and saves an IRP6 estimate for period 1 or 2 from it; set `useSbcRates` when the company is a Small Business Corporation and `alreadyPaidCents` for period 2. Rerunning it replaces the saved estimate, so under the reversible dial compute and save it, then show the computation and the estimate side by side. The user files the IRP6 themselves.

## Year-end and the AFS pack

1. Close the last month with the month-end playbook. Then `fixed_assets_write` action `depreciate` for any remaining period, `journal_entries_write` action `revalue_fx` at year end, and `tax_read` action `computation`.
2. `tax_read` action `deferred`; show it and, under the reversible dial, `tax_write` action `post_deferred` (rerunning posts only the further movement). Set `recogniseAssessedLoss` only when the user says to.
3. `afs_write` action `compile` for the year with `assurance` (ask when it is not set; South Africa's Companies Act requires the statement on the cover). Set `from` and `to` for an interim pack, which is returned without being saved. `afs_write` action `sync_notes` pulls the notes from the ledger; `afs_write` action `create_note`, `afs_write` action `update_note` and `afs_write` action `reorder_notes` for the narrative notes (accounting policies, related parties, events after the reporting date). Recompiling a draft pack is safe.
4. Present the income statement, balance sheet, equity and cash flow from the pack with the comparatives, and the list of notes.
5. Closing the year happens in Financials by a person. `afs_write` action `finalise` freezes the pack of a closed year and is irreversible: only on an explicit ask, after the year is closed.
6. `financial_reports_read` action `export` (trial balance, general ledger or journal as CSV or xlsx) for the accountant; present the link.

## Payroll: draft, explain and finalise a pay run

1. `pay_runs_read` action `list` for the cycle and its runs. `pay_runs_write` action `draft` drafts the current open period (or returns the existing draft) and reads hours from Track and unpaid leave from Leave. Set `cycleId` when the company has several cycles.
2. `pay_runs_read` action `get` and `pay_runs_read` action `inputs` to explain the run: gross, PAYE, UIF, SDL and net per employee, where hours and leave came from, and what changed since last run. `pay_runs_read` action `payslip` for one employee. Distinguish the draft's saved inputs from Track's current figures. Keep identity, tax and bank numbers masked.
3. `pay_runs_write` action `add_item` for once-off items the user asks for (bonus, deduction, reimbursement); each one makes the run unclean.
4. Refreshing the draft with `useLatestInputs: true` replaces the saved inputs with Track's and Leave's current figures. Only when the user asked for current figures, and say what it replaces first.
5. Finalising: only `pay_runs_write` action `process`, only a clean run (`clean: true`), and only after showing the user the employee count, gross, PAYE and net totals and getting an explicit yes in this chat. Set that net as `expectedNetCents`. On `RUN_HAS_CHANGES` give the `reviewUrl` so a person ticks each change in Payroll; on `TOTALS_CHANGED` show the new totals and ask again; on `APPROVAL_REQUIRED` say who has to approve it. Reversing a run, sending payslips and producing filings are done by a person.
6. `payroll_journals_write` action `post` posts a finalised run's journal to Financials when it is not already posted; `payroll_journals_read` action `list` shows which are.
7. New starters: `employees_write` action `create` with the basic fields, then give the `completeDetailsUrl`. `employees_write` action `set_sdl_exclusion` when the company is under the SDL threshold.

## Payroll take-on from Sage

1. `payroll_take_on_write` action `start` with `cutOverMonth` (the first month Looped pays). It returns the open take-on and its `reviewUrl`.
2. Read the Sage reports the user attached. `payroll_take_on_write` action `save_employees` with the allowed fields, `payroll_take_on_write` action `set_opening_ytd` for March-to-cutover year-to-date figures, `payroll_take_on_write` action `set_sdl`. `payroll_take_on_write` action `remove_employee` for someone who should not come across. `payroll_take_on_read` action `get` shows the draft.
3. Nothing in the take-on affects a pay run until a person with Full Payroll access confirms it on the take-on screen. Say so, with the link. Never claim the draft activated employees or changed a run.
