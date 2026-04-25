---
name: save-template
description: Persist a MatchLedger format template (the JSON output of the matchledger-formatter — `category`, `country`, `institution`, `slug`, `detection_hints`, `format_guide`) as two self-describing markdown files under `templates/{country}/{cc|bank|ledger|receipt}/{slug}/`. Use when the user asks to save, persist, archive, or write the just-generated formatter result to disk. Default reads the latest inline formatter JSON from the conversation; pass `--from <path>` to read from a file instead. Refuses to overwrite existing files without `--force`.
---

# save-template

Saves a MatchLedger format template — the JSON object the matchledger-formatter produces — into two self-describing markdown files under a structured folder tree. The files are written **for future Claude sessions to consult as context**, not for paste-into-MatchLedger workflows. That is why each file carries YAML frontmatter, a title, and explanatory prose around the raw payload.

## Args

- `--from <path>` — read JSON from a file instead of scanning the conversation.
- `--force` — allow overwriting existing files at the target path.

## Step 1 — Resolve the input JSON

In order:

1. If `--from <path>` is in the args, read that file with the Read tool and parse as JSON.
2. Otherwise, scan the conversation transcript (earlier assistant turns) for the most recent JSON object containing all six required fields: `category`, `country`, `institution`, `slug`, `detection_hints`, `format_guide`. The canonical block in formatter output is introduced by `--- JSON (canonical, ...)` — that's the easiest target. If multiple candidates match, use the latest.
3. If neither yields a JSON, abort with: `no formatter JSON found in conversation; pass --from <path>`.

Validate that all six required fields are present and well-typed. If anything is missing or the wrong type, abort and report which fields.

Also scan the conversation for a **source document filename** (the sample CSV/PDF path the formatter was run against). This is optional metadata — capture it if obvious, omit if not.

## Step 2 — Compute the target path

- `country_folder`:
  - `country` is null → `global`
  - otherwise normalize the ISO 3166-1 alpha-2 code to uppercase (e.g. `ph` → `PH`, `us` → `US`).
- `type_folder` from the `category` enum:
  - `BANK_SOA` → `bank`
  - `CREDIT_CARD_SOA` → `cc`
  - `LEDGER_REPORT` → `ledger`
  - `MERCHANT_RECEIPT` → `receipt`
  - any other value → abort with `unknown category: <value>`.
- Final path: `templates/{country_folder}/{type_folder}/{slug}/`

## Step 3 — Existing-file check

Use Glob (pattern `templates/{country_folder}/{type_folder}/{slug}/*.md`) to detect existing files at the target path.

- If `detection_hints.md` or `format_guide.md` already exists and `--force` was **not** passed, abort with:
  ```
  templates/{country_folder}/{type_folder}/{slug}/{file} already exists; re-run with --force to overwrite.
  ```
  Write nothing.
- If `--force` was passed, proceed and mark each overwritten file in the final summary.

## Step 4 — Re-validate the format guide

Run the format-guide validator rules from `.claude/rules/format-guide-rules.md` against the `format_guide` string. Capture:

- `char_count` — length of the string.
- Hard-rule violations (ERROR): JSON-literal pattern `\{[^}]*"[a-z_]+"\s*:`, output-format substrings (`return only json`, `return valid json`, `return only a valid`, `no markdown`), role-preamble substrings (`you are an expert`, `you are a financial`, `you are an ai`). All case-insensitive.
- Soft-rule WARNINGs: mentions of output field names (`transaction_date`, `post_date`, `reference_number`, `amount`, `confidence`, `entry_date`, `vendor_name`), currency assertions (`amounts are in usd`, `amounts are in php`, etc.), character count over 3,200.

Overall validator status:
- `clean` — no errors, no warnings.
- `warnings` — no errors, one or more warnings.
- `errors` — one or more hard-rule violations.

**Still write the files even if validator returns `errors`.** The user is archiving for context, not saving to MatchLedger directly. But flag the errors prominently in `format_guide.md` and in the closing summary.

## Step 5 — Create the directory and write the files

Run `mkdir -p templates/{country_folder}/{type_folder}/{slug}` via Bash (idempotent; safe if already exists). Then Write both files using the templates below.

### detection_hints.md template

```markdown
---
institution: {institution}
country: {country_folder}
category: {category}
category_folder: {type_folder}
slug: {slug}
created: {today in YYYY-MM-DD}
source_document: {filename if captured, otherwise OMIT this line entirely}
---

# {institution} — Detection Hints

Substring keywords MatchLedger's `FormatTemplateResolver` scans against raw document text (PDF first 2 pages, CSV first 2,000 chars) to auto-pick this template at extraction time. Match is case-insensitive substring; the first ACTIVE template within `(category, country)` whose hint list has any substring match wins, ordered by `usage_count DESC`.

## Hints

- {hint 1}
- {hint 2}
- {...}

## Specificity notes

- **{hint 1}** — {one-line reason from the conversation}
- **{hint 2}** — {one-line reason}
- {...}
```

If `detection_hints` is an empty array (Generic template), replace the `## Hints` section body with a single line:
```
(none — Generic template; resolver falls back to slug-based match.)
```
and omit the `## Specificity notes` section entirely.

If the conversation did not capture per-hint reasoning, omit the `## Specificity notes` section entirely rather than inventing reasons.

### format_guide.md template

```markdown
---
institution: {institution}
country: {country_folder}
category: {category}
category_folder: {type_folder}
slug: {slug}
created: {today in YYYY-MM-DD}
source_document: {filename if captured, otherwise OMIT this line entirely}
char_count: {N}
validator: {clean | warnings | errors}
---

# {institution} — Format Guide

Natural-language description of how to read this institution's documents. MatchLedger's `PromptBuilder` injects the **Guide** body below as `--- Institution-Specific Format Guide ---` into the extraction system prompt. It describes the input document, never the output schema.

## Guide

{format_guide string verbatim — preserve real newlines, no surrounding quotes, no JSON escaping}

## Validator notes

- Character count: {char_count} / 3200
- Hard-rule status: {pass | FAIL — bulleted list of errors}
- Soft-rule warnings: {none | bulleted list of warnings}
```

Pass the `format_guide` string through byte-for-byte — do **not** apply JSON escaping, HTML escaping, or markdown escaping. Real `\n` in the in-memory string becomes a real newline in the file.

## Step 6 — Print the summary

After both files are written, print to the user:

```
Wrote 2 files under templates/{country_folder}/{type_folder}/{slug}/:
  detection_hints.md ({n} hints)
  format_guide.md ({char_count} chars, validator: {status})
```

Prefix any overwritten file's line with `(overwritten) `.

If validator status is `errors`, append this warning on its own line:
```
WARNING: format_guide contains hard-rule violations and would be REJECTED by MatchLedger's TemplatePromptValidator on save. Fix before pasting into the admin UI.
```

## Don'ts

- Don't create a third file (e.g. the canonical JSON). The two markdown files plus their frontmatter carry enough context.
- Don't escape the `format_guide` content. Pass through as written.
- Don't commit or push anything. Persistence is local-disk only.
- Don't silently overwrite on repeated runs — the `--force` gate is deliberate, since a hand-tuned guide is the real thing at risk.
