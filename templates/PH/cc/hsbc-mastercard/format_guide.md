---
institution: HSBC Mastercard
country: PH
category: CREDIT_CARD_SOA
category_folder: cc
slug: hsbc-mastercard
created: 2026-05-17
source_document: HSBC_20260511_decrypted.pdf
char_count: 2505
validator: warnings
---

# HSBC Mastercard — Format Guide

Natural-language description of how to read this institution's documents. MatchLedger's `PromptBuilder` injects the **Guide** body below as `--- Institution-Specific Format Guide ---` into the extraction system prompt. It describes the input document, never the output schema.

## Guide

This is an HSBC Philippines credit card statement (the HSBC RED Mastercard product line). The issuer prints as "HSBC RED MASTERCARD" in the page header and "The Hongkong and Shanghai Banking Corporation Limited" in the contact block. Record the card issuer's short name as "HSBC MC" so HSBC Mastercard accounts stay distinct from HSBC Visa accounts in downstream account labels.

Statement period appears at the top as "Statement From DD MMM YYYY to DD MMM YYYY".

Card-on-account headings appear above blocks of transactions in the form "Cardholder Name 5447-XXXX-XXXX-NNNN" — the last four digits identify the specific card. A statement may include several supplementary cards in addition to the primary; all rows roll up to the same statement.

Transactions appear in a 4-column block: Post Date | Tran Date | Description | Amount(PHP).
Date format in the table: "DD MMM" (e.g. "28 Apr"). The year is inferred from the statement period.

Sign convention: charges, debits and fees are unsigned. Payments and credits carry a trailing "CR" on the amount (e.g. "15,875.56CR"). Any value ending in "CR" is a payment/credit; all others are charges.

Descriptions wrap across multiple lines — concatenate continuation lines with a single space (e.g. "Netflix.com Los Gatos" + "SGP" → "Netflix.com Los Gatos SGP"; "GOOGLE*WORKSPACE DSWIN" + "CC GOOGLE.COM SGP USD" → "GOOGLE*WORKSPACE DSWIN CC GOOGLE.COM SGP USD").

HSBC-specific shorthand:
- "SVC FEE DCC FCY ... CARD NNNN" = service fee for dynamic currency conversion on a foreign-currency transaction; the trailing four digits identify which card was used.
- "PAID THRU BPI OBP" = payment received via BPI online banking; carries the "CR" suffix.

Header-level boxes (not transactions): Account Summary (Previous Statement Balance, Payments & Credits, Purchases & Debits, Total Account Balance), Payment Summary (Payment Due Date, Minimum Payment), Credit Limit / Available Credit / Monthly Interest Rates block, and Rewards Summary (Points Earned / Adjusted / Redeemed / Balance / Expiring).

Ignore: "Continued on next page" page-break markers; the "PLEASE DETACH AND RETURN..." payment slip at the bottom of page 1; and the full disclosure section on page 2 beginning "IMPORTANT INFORMATION ABOUT YOUR HSBC CREDIT CARD" (Finance Charge, Cash Advance, Where to pay, When to pay, Lost or Stolen Cards, Fees, Late Fee, Overlimit Fee, Foreign Currency Transactions, Instalment Amount Due, Minimum Amount Due paragraphs — regulatory text, not transactions).

## Validator notes

- Character count: 2505 / 3200
- Hard-rule status: pass
- Soft-rule warnings:
  - Mentions output field substring `amount` (substring of the "Amount(PHP)" column header — same warning the gold-standard Capital One example triggers; describing the document column requires the word).
