# Domain Model & Output Contract

## What this tool produces

A single JSON object per sample document, ready to paste into MatchLedger's admin template editor:

```json
{
  "category": "BANK_SOA" | "CREDIT_CARD_SOA" | "LEDGER_REPORT" | "MERCHANT_RECEIPT",
  "country": "US" | "PH" | null,
  "institution": "string",
  "slug": "string",
  "detection_hints": ["string", ...],
  "format_guide": "string"
}
```

| Field | Source of value | Notes |
|---|---|---|
| `category` | Inferred by Claude from document structure | One of four enum values, see `document-categories.md` |
| `country` | Inferred from country-specific markers (currency symbols, addresses, language) | `null` allowed for truly global / institution-specific docs |
| `institution` | Bank / card issuer / accounting software name. Use `"Generic"` if no institution-specific markers found | The human-facing label MatchLedger persists as `template_name` |
| `slug` | Lowercase, spaces→hyphens, alphanumeric+hyphens only | E.g. `"Capital One"` → `"capital-one"`, `"BPI"` → `"bpi"` |
| `detection_hints` | Array of unique keyword strings that literally appear in the document and identify the institution | See `detection-hints-rules.md` |
| `format_guide` | Multi-line natural-language description of the document layout | See `format-guide-rules.md` — strict validator |

## Key concepts mirrored from MatchLedger

### Format Guide
A free-form text block injected into Claude's system prompt during extraction as:

```
--- Institution-Specific Format Guide ---
<format_guide content>
```

It supplements (never replaces) MatchLedger's base prompt. The base prompt owns the output JSON schema; the format guide describes the *input* document.

### Detection Hints
An array of keywords. At extraction time, MatchLedger's `FormatTemplateResolver` scans the raw document text (PDF: first 2 pages via PDFBox; CSV: first 2000 chars) for hints from each candidate template. The template with the most matching hints (within the right category + country) wins.

This means: detection hints **must literally appear in the document**. Translations, paraphrases, and "should-be-there" guesses don't fire.

### Category
Four MatchLedger enum values:
- `BANK_SOA` — bank statement of account
- `CREDIT_CARD_SOA` — credit card statement
- `LEDGER_REPORT` — accounting software export (QuickBooks, Xero, FreshBooks, Wave, Sage, etc.)
- `MERCHANT_RECEIPT` — receipts / invoices (V2 scope in MatchLedger; templates can still be authored)

### Country
ISO 3166-1 alpha-2. MatchLedger only ships US and PH today; templates use `null` (= Global) when the document has no country-specific markers and could plausibly belong to either market.

### Slug
The unique identifier within a `(category, country, tenant_id)` scope in MatchLedger. The formatter generates a candidate slug; the admin can override before save.

## Output validation before returning

The implementation MUST run the same checks as MatchLedger's `TemplatePromptValidator` against the generated `format_guide` *before* returning the JSON to the user. If validator returns ERRORs, regenerate (or fail loudly) — do not silently emit a guide that MatchLedger will reject on save. See `format-guide-rules.md` for the rule list.

## What this tool is NOT

- Not a runtime component of MatchLedger. It runs offline, against a sample document, when an admin wants to bootstrap a new template.
- Not an extractor. It does not parse transactions out of the document — that's MatchLedger's job. It only proposes the *prompt* MatchLedger will use for extraction.
- Not authoritative. The admin reviews and edits the output in the MatchLedger admin UI. The formatter is a draft generator.
