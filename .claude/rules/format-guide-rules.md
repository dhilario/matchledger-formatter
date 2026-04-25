# Format Guide Rules

The Format Guide is the most failure-prone field. MatchLedger's `TemplatePromptValidator` enforces strict separation: templates describe the **input document**, never the output.

## Hard rules — ERROR if violated (MatchLedger rejects on save)

The validator's regexes are case-insensitive and substring-based. Match them exactly:

### 1. No JSON object literals
**Pattern:** `\{[^}]*"[a-z_]+"\s*:`

Curly braces containing quoted snake_case keys followed by `:`. Examples that REJECT:
```
{ "transaction_date": "YYYY-MM-DD", "amount": 0 }
```
```
Output: { "vendor_name": "..." }
```

Even illustrative JSON intended as documentation triggers the rule. Describe the data with prose only.

### 2. No output-format instructions
Forbidden substrings (case-insensitive):
- `return only json`
- `return valid json`
- `return only a valid`
- `no markdown`

The base prompt already tells Claude to return JSON without markdown. Repeating it in the format guide is treated as an attempt to redefine the contract.

### 3. No role preambles
Forbidden substrings (case-insensitive):
- `you are an expert`
- `you are a financial`
- `you are an ai`

The base prompt sets the role. The format guide must not introduce a new persona.

## Soft rules — WARNING (allowed but flagged)

### 4. Avoid output field names
Mentioning these in the format guide is allowed but warned against, because it tempts Claude to remap the contract:
- `transaction_date`, `post_date`, `reference_number`, `amount`, `confidence`, `entry_date`, `vendor_name`

If you need to point at the document column for a date, write *"Date column"* / *"the Trans Date column"* — never `transaction_date`.

### 5. Length cap
Recommended max **3,200 characters** (roughly 800 tokens). Over the limit triggers a WARNING. The frontend shows a progress bar that turns warning-color past 3,200.

## Style rules (formatter must follow)

### 6. Don't assert currency
Since MatchLedger V5 (issue #130), generic format guides must NOT say things like *"Amounts are in USD"* or *"Amounts are likely in PHP"*. Soft hints leaked into the model's reasoning and made it infer currency from merchant names/location when the document had no symbol or ISO code at all.

The base prompt already enforces: *Detect currency ONLY from explicit symbols (`$`, `₱`, `€`) or ISO codes (`USD`, `PHP`, `EUR`) appearing in the document. If absent, return null.*

The formatter should not contradict that. Acceptable patterns:
- Don't mention currency at all (preferred for institution-specific guides — institution context is enough).
- For genuinely Generic templates, append the same disclaimer the V5 generic guides use:
  > Only report a currency if an explicit currency symbol or ISO 4217 code appears in the document; otherwise return null for the currency field. Do not infer currency from merchant names, location, or context.

Never write "Amounts are in USD" / "Amounts are in PHP" / "PHP / ₱" as a positive assertion.

### 7. Date format describes the document, not the output
**Right:** *"Date format: MM/DD/YYYY."* (this is what appears in the document)
**Wrong:** *"Dates must be in YYYY-MM-DD format."* (this is the output contract — owned by base prompt)

### 8. Sign conventions: state how the document signals direction, not the output sign
The base prompt fixes the output sign convention per category (see `document-categories.md`). The format guide should explain how the *document* distinguishes inflows from outflows so Claude can apply that convention correctly.

**Right:** *"The document has separate Debit and Credit columns; values in the Debit column are withdrawals and in the Credit column are deposits."*
**Right:** *"Charges show with a trailing CR for credits and no suffix for debits."*
**Wrong:** *"Charges must be negative numbers."* (redundant with base prompt; this would also be flagged for re-asserting output rules.)

### 9. Be specific about institution-specific quirks
This is the part that justifies a per-institution template existing at all. Capture:
- Column header names exactly as printed (e.g., *"Trans Date"* not *"Transaction Date"*).
- Institution-specific abbreviations and what they mean (*"BIZLINK = online transfer"*, *"CHECK DEP = check deposit"*, *"POS PUR = point-of-sale purchase"*).
- Multi-line transaction handling (e.g., *"FROM:NAME spans 2 lines under the description; concatenate with single space"*).
- Where statement-level metadata lives (e.g., *"Account number printed as the last token in the header below the bank name; statement period below that as 'MM/DD/YYYY to MM/DD/YYYY'"*).
- Footer/page-break artifacts that should be ignored (*"Each page has a 'Page N of M' footer; not a transaction"*).

### 10. Length: aim for 600–1,500 characters
Below 200 characters is usually too thin to be worth a template. Above 2,500 is usually padding. The 3,200 hard ceiling is a backstop, not a target.

### 11. Plain prose, short lines
The format guide is read by Claude, not parsed. Use short sentences and line breaks at natural boundaries. Avoid Markdown headings, bullet decoration is fine but optional.

## Self-check the formatter must run

Before returning the JSON, the implementation runs all rules above on the candidate `format_guide`. If any ERROR fires, the formatter must regenerate (with the rule violation echoed back into the meta-prompt) or fail loudly. WARNINGs can pass through but should be surfaced in the result so the admin sees them.

This logic must mirror `TemplatePromptValidator` exactly. The Java source at `../matchledger/src/main/java/ai/matchledger/extraction/service/TemplatePromptValidator.java` is the authoritative spec — port from there, don't re-derive.
