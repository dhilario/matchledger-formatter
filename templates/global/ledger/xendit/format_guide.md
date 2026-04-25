---
institution: Xendit
country: global
category: LEDGER_REPORT
category_folder: ledger
slug: xendit
created: 2026-04-25
source_document: UPCOMING_TRANSACTIONS_REPORT_20260421_20260422_1776821524335.csv
char_count: 2518
validator: warnings
---

# Xendit — Format Guide

Natural-language description of how to read this institution's documents. MatchLedger's `PromptBuilder` injects the **Guide** body below as `--- Institution-Specific Format Guide ---` into the extraction system prompt. It describes the input document, never the output schema.

## Guide

This is a Xendit 'Upcoming Transactions Report' — a CSV export listing payments collected by the Xendit payment gateway that are pending settlement to the merchant.

The first row is the column header. Columns in order:
Product ID | Transaction ID | Transaction Type | Transaction Status | Payment Channel | Reference | Currency | Amount | Fee Amount | VAT Amount | 3rd Party WHT | Xendit WHT | Net Amount | Created Date ISO | Time Zone | Created Date | Payment Date | Settlement Date | Bank Code | Account Number | Channel Reference | Name | Description | Invoice ID.

Each data row is one payment received by the merchant. From the merchant's ledger perspective these are inflows — the Net Amount is the figure that will be credited on the Settlement Date.

Date formats:
- Created Date ISO is ISO 8601 in UTC (for example '2026-04-21T23:47:00.857Z').
- Created Date, Payment Date, and Settlement Date are printed in the local time zone shown by the Time Zone column (for example '+0800 UTC') in the format 'DD Mon YYYY, HH:MM:SS' (for example '22 Apr 2026, 07:47:00').

Transaction Type values are Xendit enums such as CREDIT_CARD_PAYMENT, EWALLET_PAYMENT, DIRECT_DEBIT_PAYMENT, BANK_TRANSFER, and RETAIL_OUTLET_PAYMENT.
Transaction Status in this report is typically PENDING_SETTLEMENT — the report lists only transactions awaiting settlement.
Payment Channel carries Xendit's channel code (VISA, MASTERCARD, PH_GCASH, PH_GRABPAY, ID_OVO, ID_DANA, and similar).

The Amount column is the gross transaction amount. Fee Amount, VAT Amount, 3rd Party WHT, and Xendit WHT are deductions Xendit takes before settlement. Net Amount equals Amount minus those four deduction columns and is the figure that lands in the merchant's bank account.

Reference is the merchant-supplied external reference (for example a WooCommerce order id like 'woocommerce-xendit-10333'). Channel Reference is the channel-side reference (often blank for card rows, populated for ewallet and bank rows). Invoice ID is Xendit's internal invoice identifier. Name carries the payer's name when the channel exposes it. Description is the merchant statement descriptor (for example 'XDT*DMH STUDIOS').

Bank Code and Account Number are populated only for direct-debit and bank-transfer channels; they are blank for card and ewallet rows.

There is no header metadata block — the file is a flat CSV with no separator rows. The file name itself typically encodes the report window (for example 'UPCOMING_TRANSACTIONS_REPORT_20260421_20260422_*.csv').

## Validator notes

- Character count: 2518 / 3200
- Hard-rule status: pass
- Soft-rule warnings:
  - Format guide mentions output field name `amount` (substring) via the document column names "Amount", "Fee Amount", "VAT Amount", "Net Amount". This is the same pattern used in the gold-standard Capital One example; MatchLedger's `TemplatePromptValidator` surfaces a soft WARNING but does not block save.
