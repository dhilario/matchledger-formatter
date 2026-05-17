---
institution: HSBC Mastercard
country: PH
category: CREDIT_CARD_SOA
category_folder: cc
slug: hsbc-mastercard
created: 2026-05-17
source_document: HSBC_20260511_decrypted.pdf
---

# HSBC Mastercard — Detection Hints

Substring keywords MatchLedger's `FormatTemplateResolver` scans against raw document text (PDF first 2 pages, CSV first 2,000 chars) to auto-pick this template at extraction time. Match is case-insensitive substring; the first ACTIVE template within `(category, country)` whose hint list has any substring match wins, ordered by `usage_count DESC`.

## Hints

- HSBC RED MASTERCARD
- Hongkong and Shanghai Banking Corporation Limited
- Card Products Centre
- SVC FEE DCC FCY

## Specificity notes

- **HSBC RED MASTERCARD** — product-line header printed at the top of page 1; ties this template specifically to HSBC PH Mastercard statements rather than any other HSBC product.
- **Hongkong and Shanghai Banking Corporation Limited** — full legal entity name in the contact block; backup if the product-line header isn't picked up cleanly by PDF text extraction.
- **Card Products Centre** — HSBC PH credit-card business-unit address line ("Card Products Centre, PO BOX 1096 Makati Central Post Office"); discriminates from HSBC bank-account statements which use a different department.
- **SVC FEE DCC FCY** — HSBC-specific transaction-code prefix for service fee on dynamic currency conversion (foreign currency); appears on every statement that has any FX transaction and is not used by other Philippine card issuers.
