# Detection Hints Rules

Detection hints are a string array MatchLedger scans against raw document text to auto-pick this template at extraction time. Quality matters: a missed hint means the document falls back to Generic and accuracy drops.

## How MatchLedger uses them at runtime

| Document type | Text source scanned |
|---|---|
| PDF | First **2 pages** extracted via PDFBox |
| CSV | First **2,000 characters** of raw file content |

### The matching algorithm — read this carefully

The match is **case-insensitive substring**, **OR within a template**, **first-matching-template wins**, with templates ordered by `usage_count DESC`.

Pseudocode (port of `FormatTemplateService.matchByDetectionHints` + `FormatTemplateRepository.findActiveWithDetectionHints`):

```
templates = SELECT * FROM format_templates
            WHERE category = :category
              AND status = ACTIVE
              AND detection_hints IS NOT NULL
            ORDER BY usage_count DESC

lowerText = documentText.toLowerCase()

for template in templates:
    for hint in template.detection_hints:
        if lowerText.contains(hint.toLowerCase()):
            return template       # short-circuits — done
return None                       # falls back to slug → Generic
```

Key consequences:
- **No scoring, no AND, no fuzzy.** A template either matches (any hint is a substring) or it doesn't.
- **Order of hints inside a template is irrelevant** to which template wins. It only changes which hint string gets logged at DEBUG level.
- **Order of templates is `usage_count DESC`** — more-used templates are checked first and short-circuit later candidates.
- **Cross-template hint collisions favor the higher-`usage_count` template.** This is the sharp edge — see "Specificity is critical" below.

### Specificity is critical (the cross-template collision trap)

Worked example. Three active `BANK_SOA` templates exist:

| Template | usage_count | hints |
|---|---|---|
| Capital One | 142 | `["Capital One", "Member FDIC", "Statement Period"]` |
| Wells Fargo | 23 | `["Wells Fargo", "Member FDIC", "Equal Housing Lender"]` |

A real **Wells Fargo** statement is uploaded. Its text contains "Wells Fargo" and "Member FDIC".
- Capital One is checked first (higher usage_count). "capital one" not found → continue. "member fdic" found → **return Capital One**. ❌

Wells Fargo's statement matches Capital One's template because both list `"Member FDIC"` and Capital One was more-used. Wells Fargo's template is never even checked.

The rule that follows: **every hint must be specific enough that a false-positive match against a different institution's document is unlikely.** Generic regulatory phrases ("Member FDIC", "BSP-Regulated") are dangerous as standalone hints — they only work as redundancy alongside an institution-name hint that's much more discriminating.

## Hard rules

### 1. Must literally appear in the document
Read the actual extracted text (PDF first 2 pages, or CSV first 2000 chars). Pick strings that appear there *as written*.

A keyword like *"BPI"* won't fire on a document that prints the institution as *"Bank of the Philippine Islands"*. Pick the form that's actually printed, or include both.

### 2. Pick discriminating strings, not generic ones
- ✅ Good: `"Capital One"`, `"BPI Family Bank"`, `"Statement of Account No."`, `"BIZLINK"`
- ❌ Bad: `"Statement"`, `"Date"`, `"Amount"`, `"Total"` — too generic; matches every document.

If a hint matches Generic-template documents from other institutions, it's noise, not signal.

### 3. 3–5 hints. Floor at 3, ceiling at 5. Prefer fewer if each is strong.

The asymmetry that sets the band:

| Failure mode | What happens | How visible |
|---|---|---|
| Too few hints (or hints absent from extracted text) | Document falls back to Generic | **Loud** — accuracy drops, admin opens a format request, admin investigates. |
| Too many hints, one is non-specific | Template steals another institution's document via false-positive substring match | **Silent** — wrong format guide injected, extraction degrades, no log says "you stole their document." |

Silent failures are worse than loud ones. The dial sits toward **fewer hints, each highly specific** — not "more hints, hope one matches."

The floor of 3 exists because PDF text extraction is unreliable at the edges:
- Institution names sometimes appear only in image-rendered logos PDFBox can't capture.
- The first 2 pages might be cover/disclosure, with the institution name on page 3.
- Scanned-then-OCR'd statements can mangle the institution name.

So you carry **2–3 backup variants of identifying strings** as redundancy against bad text extraction — not as confidence-stacking. The ceiling of 5 caps the false-positive surface area; adding a sixth marginally-discriminating hint increases collision risk without buying meaningful coverage.

### 4. Each hint must be unique within the array
Don't repeat the same string in different casings — matching is case-insensitive. Don't include both `"Capital One"` and `"CAPITAL ONE"`.

### 5. Don't include account-specific data
- ❌ Account numbers, customer names, statement dates, monetary amounts.

These vary across documents from the same institution and would tie the template to one customer.

## Specificity test (apply to every hint)

Before adding hint #N, ask: *"Could this string appear on a document from a different institution?"*

- ✅ **Strong (specific to one institution):** `"Capital One, N.A."`, `"BPI Family Savings Bank"`, `"Wells Fargo Bank, N.A."`, `"Daily Available Balance"`.
- ⚠️ **Weak (specific to a class of institutions):** `"Member FDIC"` (every US bank), `"BSP-Regulated"` (every PH bank). Use only as defensive backup, never as primary discriminator. Two weak hints together still don't make one strong hint.
- ❌ **Noise:** `"Statement"`, `"Account Number"`, `"Total"`, `"USD"`, `"Date"`. Match every document. Drop on sight.

If a hint can't pass the specificity test, it's not redundancy — it's a collision waiting to happen with whichever institution acquires the higher `usage_count`.

## Practical recipe (default shape)

For a typical institution-specific template, produce exactly this shape:

1. **Hint 1** — institution name in its most-discriminating printed form (with legal suffix or product line: `"Capital One, N.A."`, `"BPI Credit Cards"`).
2. **Hint 2** — alternate printed form of the institution name (logo word, parent company, common abbreviation): `"Capital One"`, `"Bank of the Philippine Islands"`.
3. **Hint 3** — institution-specific section header or transaction code that doesn't appear on competitor documents: `"BIZLINK"`, `"Daily Available Balance"`, `"Cardmember Statement"`.

Stop there unless there's a concrete reason for a 4th or 5th. *"I have more good hints"* is not a reason.

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

### e. Footer / regulatory identifiers — supporting hints only, never standalone
- *"Member FDIC"* (US banks)
- *"BSP-Regulated"* (PH banks)
- *"BIR Permit No."* (PH receipts)
- *"Equal Housing Lender"*

These appear on documents from **many** institutions in the same jurisdiction. Listing one as a hint on Bank A's template will steal documents from Bank B and Bank C if Bank A has the highest `usage_count`. Use them as defense-in-depth alongside the institution-name hint, never as the only or first match-target.

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

For non-Generic templates, MatchLedger requires at least 1 detection hint before the template can be activated. The formatter should target 3–5 (see rule 3).

## Self-check the formatter must run

Before returning the JSON, validate the candidate hints:
1. Each hint string is non-empty and ≤ 200 characters.
2. No duplicates (case-insensitive).
3. Each hint actually appears (case-insensitive substring) in the source document text the formatter parsed. If a hint doesn't, drop it and log — generating hints that won't fire is worse than producing fewer hints.
4. For non-Generic templates: array length is 3, 4, or 5. Below 3 → ask Claude to propose more variants. Above 5 → drop the weakest by the specificity test until at 5.
