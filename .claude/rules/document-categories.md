# Document Categories

MatchLedger's `TemplateCategory` enum has four values. Each has its own base extraction prompt with a fixed output JSON schema. The format guide supplements that prompt — it must align with the category's conventions.

This file summarizes what each category's base prompt expects, so the formatter can write a guide that complements it (and never contradicts it).

---

## BANK_SOA — Bank Statement of Account

**Base output schema (fixed by MatchLedger):** account-level metadata + `transactions[]` with `transaction_date`, `post_date`, `description`, `amount`, `running_balance`, `reference_number`, `category`.

**Sign convention (output):** withdrawals/debits **negative**, deposits/credits **positive**.

**What the format guide should cover:**
- Column layout exactly as printed (e.g., *"Columns: Date | Description | Withdrawal | Deposit | Balance"*).
- How direction is signaled in the document:
  - Two columns (Debit/Credit, Withdrawal/Deposit) → describe which is outflow vs inflow.
  - Single Amount column with sign → describe the sign convention.
  - Suffix markers (e.g., trailing *"CR"* for credits) → describe the marker.
- Date format as printed (`MM/DD/YYYY`, `DD/MM/YYYY`, `Mon DD YYYY`, etc.).
- Where header metadata lives: account number, statement period, opening/closing balance, totals.
- Multi-line description handling (e.g., *"FROM:NAME on second line, concatenate with single space"*).
- Institution-specific transaction codes (*"BIZLINK"*, *"PESONET"*, *"ATM W/D"*).
- Footer/page artifacts to ignore.

**What NOT to include:**
- Output sign convention (already enforced by base prompt).
- Currency assertions ("Amounts are in USD") — see `format-guide-rules.md` rule 6.
- Output field names like `transaction_date` (use *"Date column"* instead).

---

## CREDIT_CARD_SOA — Credit Card Statement

**Base output schema:** issuer metadata, account summary fields (previous_balance, new_balance, minimum_payment_due, payment_due_date, total_charges, total_payments, etc.) + `transactions[]` with `transaction_date`, `post_date`, `description`, `amount`, `reference_number`, `category`, `transaction_type`.

**Sign convention (output):** charges/purchases **negative**, payments/credits/refunds **positive**.

**What the format guide should cover:**
- Issuer name and card network (Visa / Mastercard / Amex / JCB) as printed.
- Account summary block layout: where to find previous balance, payments, charges, new balance, minimum due, due date, credit limit.
- Transaction table column order and how charges vs payments are visually distinguished:
  - Separate Charge / Payment columns.
  - Single Amount column with `(CR)` suffix for credits.
  - Negative sign for credits but positive for charges (some PH issuers).
- Date format (transaction date and posting date are often both shown).
- Foreign-currency transaction lines (e.g., *"FX charges show original currency in parentheses below the description; the USD-converted amount is the charged value"*).
- Cash advance, fee, and interest line distinguishers.
- Reward redemption / annual fee lines (often in a different format).

**What NOT to include:**
- Same exclusions as `BANK_SOA`.

---

## LEDGER_REPORT — Accounting Software Export

**Base output schema:** report metadata (source_software, account_name, company_name, report_type, period dates) + `entries[]` with `entry_date`, `entry_type`, `description`, `amount`, `reference_number`, `memo`, `category`, `reconciled_in_source`.

**Sign convention (output):** deposits/income **positive**, payments/expenses **negative**. If the report has separate Payment and Deposit columns, Payment values are negative and Deposit values are positive.

**Common source software:** QuickBooks (Online + Desktop), Xero, FreshBooks, Wave, Sage, Quicken, MYOB, Zoho Books.

**What the format guide should cover:**
- Source software name as printed in the report header.
- Report type: *Reconciliation Report*, *General Ledger*, *Transaction Detail*, *Trial Balance*, etc.
- Whether amounts are split into Payment/Deposit columns or shown in a single Amount column.
- How `entry_type` is conveyed (a Type column, embedded in the description prefix, etc.).
- Reference number column — what it's labeled (*Num*, *Check No.*, *Ref*, *Doc No.*).
- The Memo column distinct from the Description / Payee column.
- Class / Department / Location columns (QuickBooks Online specifically).
- How the source software marks reconciled rows (a `*` column, an `R` flag, a Cleared column).
- Subtotals and group headers that aren't actual entries (a Type group break in QuickBooks, account headers in General Ledger).

**What NOT to include:**
- Same exclusions as `BANK_SOA`.

**Known QuickBooks PDF Reconciliation Report structure** (most common LEDGER_REPORT input today):
- Header block: company name, report title (*Reconciliation Report*), as-of date, account name (*Checking*, *Wells Fargo Checking*).
- Begin/end balance summary.
- Cleared transactions section: split into *Cleared Checks and Payments* + *Cleared Deposits and Other Credits*.
- Uncleared transactions section.
- All entries have: Date, Type, Num, Name (Payee), Memo, Amount.

---

## MERCHANT_RECEIPT — Receipts and Invoices

**Status:** V2 scope in MatchLedger — extraction is not user-reachable in V1 production. Templates can still be authored.

**Base output schema:** vendor metadata, transaction date, subtotal, tax, tip, total, currency, payment method, last four digits, category, `line_items[]` (description, quantity, unit_price, amount).

**Sign convention:** N/A (positive amounts only — receipts represent a single transaction).

**What the format guide should cover:**
- Vendor / store name location in the document.
- Date and time format and location.
- Receipt / OR / SI / invoice number prefix and location.
- Itemization layout: how line items are stacked (qty × price = amount, or three-column).
- Tax breakdown (US sales tax vs PH 12% VAT — but state how it's labeled, don't assert which it is).
- Tip line (US service industry receipts).
- BIR Permit / TIN identifiers (PH).
- Payment method line: where the card last four appears (often *"VISA *1234"* or *"AMEX 1234"*).

---

## Country guidance

| Marker | Implies |
|---|---|
| `$` only as currency, MM/DD/YYYY dates, *"Member FDIC"*, US state abbreviations in addresses | `country = "US"` |
| `₱` or `PHP` currency, *"BSP"*, *"BIR"*, *"TIN"*, PH addresses, MM/DD/YYYY dates | `country = "PH"` |
| Cross-jurisdictional / no clear country marker | `country = null` (Global) |

The `country` field affects which Generic fallback template MatchLedger uses if institution-specific resolution misses. Don't guess — if the document genuinely lacks markers, choose `null`.
