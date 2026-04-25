# Detection Hints Rules

Detection hints are a string array MatchLedger scans against raw document text to auto-pick this template at extraction time. Quality matters: a missed hint means the document falls back to Generic and accuracy drops.

## How MatchLedger uses them at runtime

| Document type | Text source scanned |
|---|---|
| PDF | First **2 pages** extracted via PDFBox |
| CSV | First **2,000 characters** of raw file content |

The scan is case-insensitive substring matching. The template with the **most matching hints** within the right `(category, country)` scope wins resolution.

## Hard rules

### 1. Must literally appear in the document
Read the actual extracted text (PDF first 2 pages, or CSV first 2000 chars). Pick strings that appear there *as written*.

A keyword like *"BPI"* won't fire on a document that prints the institution as *"Bank of the Philippine Islands"*. Pick the form that's actually printed, or include both.

### 2. Pick discriminating strings, not generic ones
- ✅ Good: `"Capital One"`, `"BPI Family Bank"`, `"Statement of Account No."`, `"BIZLINK"`
- ❌ Bad: `"Statement"`, `"Date"`, `"Amount"`, `"Total"` — too generic; matches every document.

If a hint matches Generic-template documents from other institutions, it's noise, not signal.

### 3. 3–7 hints is the sweet spot
Fewer than 2: brittle. The first hint may be missing on a thin sample document.
More than 8: diminishing returns. Resolution doesn't need redundancy.

### 4. Each hint must be unique within the array
Don't repeat the same string in different casings — matching is case-insensitive. Don't include both `"Capital One"` and `"CAPITAL ONE"`.

### 5. Don't include account-specific data
- ❌ Account numbers, customer names, statement dates, monetary amounts.

These vary across documents from the same institution and would tie the template to one customer.

## What makes a strong hint

Strong hints come from these categories. Aim to pick from at least 2–3 different categories for resilience:

### a. Institution name (always include)
- The bank, card issuer, or accounting software brand exactly as printed.
- If the brand has multiple printed forms (logo word + legal name), include both.
  - *"Wells Fargo"* + *"Wells Fargo Bank, N.A."*

### b. Document-type label
The phrase the institution uses to describe the document.
- Bank: *"Statement of Account"*, *"Account Statement"*, *"Bank Statement"*
- Credit card: *"Credit Card Statement"*, *"Account Summary"*, *"Cardmember Statement"*
- Ledger: *"Reconciliation Report"*, *"General Ledger"*, *"Transaction Detail"*

### c. Institution-specific section headers
- *"Daily Available Balance"* (Wells Fargo)
- *"Balance Subject to Finance Charge"* (BPI credit cards)
- *"Reconcile Account"* (QuickBooks ledger)

### d. Institution-specific transaction codes
- *"BIZLINK"* (BPI online transfer code)
- *"CHECK DEP"* (common bank shorthand)
- *"POS PUR"* (point-of-sale purchase, Capital One)

### e. Footer / regulatory identifiers
- *"Member FDIC"* (US banks)
- *"BSP-Regulated"* (PH banks)
- *"BIR Permit No."* (PH receipts)
- *"Equal Housing Lender"*

### f. Software/format markers (CSV only)
For CSV-exported ledger reports, the column header row is often the most reliable hint:
- *"Date,Type,Num,Name,Memo,Account,Split,Debit,Credit"* (QuickBooks export)
- *"Date,Description,Amount,Running Balance"* (generic CSV bank export)

## What NOT to use

- ❌ **Currency symbols and ISO codes** alone (`"$"`, `"USD"`, `"PHP"`) — match too broadly.
- ❌ **Dates** (`"2024"`, `"January"`) — every document has them.
- ❌ **Generic English words** (`"Total"`, `"Balance"`, `"Description"`) — present in every statement format ever.
- ❌ **Customer-specific data** (account holder name, account number, address).

## Generic templates: zero hints

The Generic templates in MatchLedger (`Generic US`, `Generic PH`, `Generic Global`) carry **no detection hints**. They are the fallback when no institution-specific template matches. The formatter must NOT produce hints when proposing a Generic template — leave the array empty.

For non-Generic templates, MatchLedger requires at least 1 detection hint before the template can be activated. The formatter should produce at minimum 3 to give the admin headroom to prune.

## Self-check the formatter must run

Before returning the JSON, validate the candidate hints:
1. Each hint string is non-empty and ≤ 200 characters.
2. No duplicates (case-insensitive).
3. Each hint actually appears (case-insensitive substring) in the source document text the formatter parsed. If a hint doesn't, drop it and log — generating hints that won't fire is worse than producing fewer hints.
4. For non-Generic templates: array length ≥ 3.
