# Meta-Prompt Strategy

This file describes how the formatter should ask Claude to produce the `{detection_hints, format_guide, ...}` JSON. It does NOT describe MatchLedger's extraction prompt — that's owned by `../matchledger`. This is the prompt sent by *this tool*, asking Claude to author a template for *that tool*.

## Architecture

```
Sample document (PDF/CSV)
    ↓
Text extraction (PDFBox-equivalent first 2 pages, or first 2000 chars CSV)
    ↓
Build messages:
  system  = META_PROMPT  (large, stable, cache_control: ephemeral)
  user    = {document_text} + per-call instructions
    ↓
Claude API call (claude-sonnet-4-6 default)
    ↓
JSON response → local validator (mirrors TemplatePromptValidator)
    ↓
On ERROR: regenerate with violations echoed back (max 2 retries)
On WARN: include in result for admin to see
On clean: return JSON
```

## Why split system vs user message

The meta-prompt is large (rules + good/bad examples + output contract). The document text varies per call. Putting the meta-prompt in `system` with `cache_control: {"type": "ephemeral"}` lets Anthropic's prompt cache reuse it across calls within the 5-minute TTL. Real cost win when an admin batches several samples.

## System prompt content

Compose the system prompt from these blocks, in order:

1. **Role** — one sentence: *"You author MatchLedger format templates from sample financial documents. You produce a single JSON object describing how to read the document, never what its data should look like in extracted form."*
2. **Output contract** — the JSON shape from `domain.md`, with brief field descriptions.
3. **Hard rules** — the validator rules from `format-guide-rules.md` sections 1–3 (JSON literals, output instructions, role preambles), stated as hard NEVERs.
4. **Style rules** — sections 4–11 from `format-guide-rules.md` (output field names, length, no currency assertion, date format, sign-convention phrasing, institution quirks).
5. **Detection hints rules** — the rules from `detection-hints-rules.md`.
6. **Per-category guidance** — the four sections from `document-categories.md`, each kept tight.
7. **Two anchored examples** — one good, one bad, from `examples.md`. The bad example is critical — it shows Claude what *not* to write.
8. **Final instruction** — *"Return ONLY the JSON object. No prose around it. No markdown fences."*

Total system prompt budget: aim for under 6,000 tokens. The cache makes it cheap on repeat, but tight prompts also generate better output.

## User message content

Each call's user message contains:

1. The extracted document text (PDF first 2 pages OR CSV first 2000 chars), labeled *"--- Document text ---"* and bracketed with delimiters.
2. Optional per-call hints the admin passed in (institution name override, category override, etc.) labeled *"--- Admin hints ---"*.
3. *"Produce the JSON now."*

Do NOT inline the system prompt content here — that defeats caching and bloats every call.

## Retries on validation failure

If the local validator returns ERRORs against the generated `format_guide`:

1. Re-call with the same system prompt (still cached).
2. User message becomes:
   ```
   --- Document text ---
   <same as before>

   --- Previous attempt rejected ---
   <the bad format_guide>

   --- Validator violations ---
   - <error message 1>
   - <error message 2>

   Produce the JSON now. The new format_guide must not contain any of the violations above.
   ```
3. Cap at 2 retries (3 total attempts). On final failure, return the last attempt with `errors` populated so the admin can see what went wrong rather than getting silence.

## Model choice

- **Default:** `claude-sonnet-4-6`. Strong enough for this task, cheaper than Opus.
- **Fallback for thorny documents:** `claude-opus-4-7` (the latest Opus). Trigger Opus only if Sonnet fails validation twice in a row, or if the admin opts in via a `--deep` flag.
- **Never use Haiku** for this. Format-guide authoring is judgment-heavy; Haiku will produce shallower guides.

## Token economy

- System prompt: ~6k tokens, cached after first use.
- User message: PDF text rarely exceeds 4k tokens for 2 pages; CSV preview is ~500 tokens.
- Output: typical generated template runs 800–1500 tokens.
- Per call (cached): ~10k input + 1.5k output. With cache hit, the input cost drops dramatically.

## What NOT to do

- ❌ Don't ask Claude to produce a Markdown report wrapping the JSON. The output must be a bare JSON object.
- ❌ Don't ask Claude to output extra fields beyond the contract in `domain.md`. Excess fields create temptation to put output-schema definitions there.
- ❌ Don't mix the meta-prompt with MatchLedger's extraction base prompt. They're for different stages: this tool authors templates; MatchLedger uses them.
- ❌ Don't run a separate "categorize the document" call followed by a "now write the guide" call. One call with all the rules is cheaper, faster, and avoids drift between the two passes.
