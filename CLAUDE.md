# matchledger-formatter

## What This Is
A companion tool to **MatchLedger** that, given a sample bank statement / credit card statement / ledger report / receipt (PDF or CSV), uses Claude to draft:

1. **Detection Hints** — keywords the MatchLedger extraction pipeline will scan against raw document text to auto-pick this template.
2. **Format Guide** — natural-language description of the document layout that MatchLedger injects as `--- Institution-Specific Format Guide ---` into the system prompt for Claude extraction calls.

The output is a JSON object the admin pastes into the MatchLedger admin template editor at `/admin/templates/new`.

## Critical Constraint — read this first

The Format Guide describes **how to READ the input document**. It must NEVER define the output JSON schema or override MatchLedger's base extraction prompt. MatchLedger owns the output contract. Templates are supplementary.

If the formatter produces a Format Guide containing JSON snippets, output field names, role preambles ("you are an expert..."), or instructions like "return only JSON" — MatchLedger's `TemplatePromptValidator` rejects it on save with an ERROR.

Full validator rules + style guide:
@.claude/rules/format-guide-rules.md

## Domain & Output Contract
@.claude/rules/domain.md

## Detection Hints Rules
@.claude/rules/detection-hints-rules.md

## Document Categories (per-category guidance)
@.claude/rules/document-categories.md

## Meta-Prompt Strategy (how to ask Claude)
@.claude/rules/meta-prompt.md

## Gold-Standard Examples
@.claude/rules/examples.md

---

## Tech Stack — TBD (no implementation committed yet)

The repository is documentation-only at this point. The choice of stack is open and not yet locked in. Reasonable options for an implementation agent to propose:

- **Python** with `anthropic` SDK + `pypdf` (or `pdfplumber`) + standard `csv` module. Lowest-friction match given the workload (LLM call, light text extraction, JSON in/out). No need to mirror MatchLedger's Java stack — this tool runs offline as a developer/admin utility, not as part of the SaaS runtime.
- **Node/TypeScript** with `@anthropic-ai/sdk` if the team prefers TS tooling.
- **Java** (matching MatchLedger) — possible but heavier than warranted for a single-purpose CLI.

Whichever stack lands, the implementation must:
- Use **prompt caching** on the system prompt (the meta-prompt is large and stable across runs). See `meta-prompt.md`.
- Read PDFs by extracting **first 2 pages of text** (mirrors MatchLedger's `PdfTextExtractor` so detection hints chosen here will actually fire there).
- Read CSVs by sampling the **first 2000 characters** (also mirrors MatchLedger).
- Validate the generated `format_guide` against the same rules MatchLedger's `TemplatePromptValidator` enforces — locally, before returning the result. Catching violations at generation time saves the admin a round trip.
- Default model: latest Claude Sonnet (currently `claude-sonnet-4-6`). Reserve Opus for the test-lab "deep authoring" mode if added later.

## Workflow Rules

- **Never commit or push without explicit user approval.** Suggest commits when ready; wait for the user to run them.
- The MatchLedger sibling repo at `../matchledger` is the source of truth for everything format-template-related. If a rule here disagrees with `../matchledger/.claude/rules/format-library.md`, fix this repo, not MatchLedger's.

## Sibling Repo References (read-only)

- `../matchledger/.claude/rules/format-library.md` — canonical format-template rules
- `../matchledger/.claude/rules/extraction.md` — extraction module (consumes the templates this tool produces)
- `../matchledger/src/main/java/ai/matchledger/extraction/service/TemplatePromptValidator.java` — the validator whose rules this repo must mirror exactly
- `../matchledger/src/main/java/ai/matchledger/extraction/service/PromptBuilder.java` — base prompts the format guide gets injected into
- `../matchledger/src/main/resources/db/migration/V2__seed_data.sql` (Generic templates) — examples of acceptable format guides
