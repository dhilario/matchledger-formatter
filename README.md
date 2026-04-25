# matchledger-formatter

A standalone tool that takes a sample bank statement, credit card statement, ledger report, or receipt (CSV or PDF) and uses Claude to suggest a **Detection Hints** array and a **Format Guide** string ready to paste into the MatchLedger admin template editor.

## What it solves

When a MatchLedger customer uploads a statement from an institution we don't yet have a template for, extraction falls back to a Generic template and accuracy drops. The fix is to author a per-institution `FormatTemplate` (detection hints + format guide). Today that authoring is manual — an admin reads the document and writes the prompt by hand.

`matchledger-formatter` automates the first draft. Drop a sample document in, get back a JSON object the admin can paste into MatchLedger's template editor at `/admin/templates/new` (or `/edit/:id`).

## Output contract

```json
{
  "category": "BANK_SOA",
  "country": "US",
  "institution": "Capital One",
  "slug": "capital-one",
  "detection_hints": [
    "Capital One",
    "Statement Period",
    "Account Summary"
  ],
  "format_guide": "Columns: Date | Description | Amount | Balance.\nDate format: MM/DD/YYYY.\nDebit/Credit signaled by Amount sign — negative = withdrawal, positive = deposit.\n..."
}
```

The shape mirrors `format_templates` columns in the MatchLedger schema. See [.claude/rules/domain.md](.claude/rules/domain.md) for the full contract.

Alongside the JSON, the tool prints the same data in two paste-ready blocks (one hint per line for the Detection Hints chip input; real-newline prose for the Format Guide textarea) and offers to push either block straight to your system clipboard so you can paste directly into the MatchLedger admin template editor without hand-stripping JSON quotes or `\n` escapes. See [.claude/rules/output-presentation.md](.claude/rules/output-presentation.md).

## Quickstart

> Tech stack TBD. The repository is currently documentation-only — no implementation has been committed yet. See [CLAUDE.md](CLAUDE.md) for stack guidance and the implementation plan.

## Status

Repository scaffolding only. Docs in `.claude/rules/` capture all the prompt-engineering knowledge needed for an implementation agent to build this end-to-end.

## Related

- **MatchLedger** (`../matchledger`) — the SaaS this tool feeds.
- MatchLedger's authoritative format-template rules: `../matchledger/.claude/rules/format-library.md`, `extraction.md`.
