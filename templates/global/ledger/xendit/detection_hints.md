---
institution: Xendit
country: global
category: LEDGER_REPORT
category_folder: ledger
slug: xendit
created: 2026-04-25
source_document: UPCOMING_TRANSACTIONS_REPORT_20260421_20260422_1776821524335.csv
---

# Xendit — Detection Hints

Substring keywords MatchLedger's `FormatTemplateResolver` scans against raw document text (PDF first 2 pages, CSV first 2,000 chars) to auto-pick this template at extraction time. Match is case-insensitive substring; the first ACTIVE template within `(category, country)` whose hint list has any substring match wins, ordered by `usage_count DESC`.

## Hints

- Xendit WHT
- PENDING_SETTLEMENT
- EWALLET_PAYMENT
- Channel Reference

## Specificity notes

- **Xendit WHT** — column header containing the brand name; appears in every Xendit upcoming transactions report regardless of country, making it the single strongest discriminator.
- **PENDING_SETTLEMENT** — Xendit-shaped Transaction Status enum value; this report by definition only lists transactions awaiting settlement.
- **EWALLET_PAYMENT** — Xendit Transaction Type enum; the uppercase/underscore form is distinctive to Xendit's API shape.
- **Channel Reference** — Xendit-specific column header for the channel-side reference id (often blank for card rows, populated for ewallet/bank rows).
