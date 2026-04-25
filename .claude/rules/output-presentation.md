# Output Presentation & Clipboard

`domain.md` defines the JSON contract — the programmatic shape of the formatter's output. This file defines the **human-facing presentation** of that output. The two are separate concerns:

- **JSON** is for machines (saving to a file, piping to another tool, programmatic integration).
- **Presentation** is for humans pasting into MatchLedger's admin template editor at `/admin/templates/new`.

The MatchLedger admin form has two distinct fields the user pastes into:

1. **Detection Hints** — a chip-style input. The admin types or pastes one hint, presses Enter, repeats. Pasting a JSON array literal `["a", "b"]` does not work.
2. **Format Guide** — a `<textarea>`. The admin pastes prose with real line breaks. Pasting a JSON-escaped string with `\n` literal escape sequences does not work.

The formatter must therefore render the same data twice: once as JSON (for machine), once as paste-ready text (for human), and offer to push either block straight to the system clipboard.

---

## Required output sections

After Claude returns the JSON, the formatter prints the result in three blocks, in this order:

### Block 1 — JSON (the canonical contract)
```
=== JSON (canonical) ===
{
  "category": "BANK_SOA",
  ...
}
```
Pretty-printed with 2-space indent for readability, but still a single valid JSON document. This block is what stdout pipes / file output captures.

### Block 2 — Detection Hints (paste-ready)
```
=== Detection Hints (paste-ready) ===
Xendit WHT
PENDING_SETTLEMENT
PH_GCASH
PH_GRABPAY
EWALLET_PAYMENT
Channel Reference
3rd Party WHT
```

Rules for this block:
- One hint per line.
- No quotes, commas, brackets, or JSON syntax.
- Trim whitespace on each hint.
- Empty array → print `(none — Generic template)` so the user isn't confused by silence.

This format is friendly for two paste workflows:
- **Bulk paste** — most browsers + MUI Autocomplete chip inputs accept newline-delimited input and add each line as a chip.
- **One-by-one** — the admin can also drag-select a single line and paste, repeat for each.

### Block 3 — Format Guide (paste-ready)
```
=== Format Guide (paste-ready) ===
This is a Xendit 'Upcoming Transactions Report', a CSV export...

The first row is the column header. Columns in order:
Product ID | Transaction ID | ...

Date format: the Created Date, Payment Date, and Settlement Date columns print as 'DD MMM YYYY, HH:MM:SS' in Philippine Standard Time...
```

Rules for this block:
- Real newlines, not `\n` escapes. The admin's `<textarea>` must receive the same line breaks Claude wrote.
- No surrounding quotes.
- No leading/trailing blank lines.
- Preserve internal blank lines that came from Claude (those are paragraph breaks; the textarea is `whiteSpace: pre-wrap` in MatchLedger).
- Print the character count beneath the block: `(1487 / 3200 chars — within recommended limit)`. If over 3,200, append a warning line: `(over recommended max — MatchLedger will flag a WARNING on save)`.

---

## Clipboard UX

After printing the three blocks, the formatter offers clipboard copy. Two modes:

### Interactive (default when stdin is a TTY)
```
Copy Detection Hints to clipboard? [Y/n]
Copy Format Guide to clipboard? [Y/n]
```

- Default is **yes** on Enter — paste-into-MatchLedger is the most common next step.
- `n` skips that block. The other prompt still runs.
- Two separate prompts on purpose — sometimes the admin only needs to update one field on an existing template.

The hints block goes to the clipboard in the same newline-delimited form as Block 2. The format guide goes in the same prose form as Block 3.

### Non-interactive (when stdin is not a TTY, or `--no-prompt` is passed)
Skip the prompts. Two flags control which block(s) to push to the clipboard:

| Flag | Behavior |
|---|---|
| `--copy hints` | Copy Detection Hints block, leave Format Guide untouched. |
| `--copy guide` | Copy Format Guide block, leave Detection Hints untouched. |
| `--copy both` | Copy guide first, then hints (so the hints land last and are top-of-clipboard). |
| `--copy none` (default) | Copy nothing. |

The "guide first, hints last" ordering for `--copy both` is deliberate. Hints are the more common second-paste action (admin pastes guide, switches focus to hints field, pastes hints). If the user runs `--copy both` they get hints on the top of the clipboard, ready to paste once they've manually pasted the guide from the terminal output.

### Cross-platform clipboard library

Whatever the implementation language:

- **Python**: `pyperclip` (pure Python, falls back to `xclip`/`xsel` on Linux, native on macOS/Windows). Already what most CLI tools use.
- **Node/TypeScript**: `clipboardy`. Same cross-platform model.

Don't roll your own. Both libraries handle the OS quirks (WSL, Wayland vs X11, lack of clipboard on headless Linux).

### Headless environments

If the clipboard library raises (no display server, SSH session without X forwarding, container without xclip), the formatter must NOT fail the run. Catch the error, print:

```
[clipboard unavailable — paste manually from the block above]
```

…and continue. The blocks above are still on stdout for manual selection.

---

## Why JSON-with-escaped-newlines is not paste-friendly

When Claude returns `{"format_guide": "Line one.\nLine two."}`, JSON requires `\n` to be the two-character escape sequence inside the string literal. Pretty-printing the JSON preserves that escape — it does **not** convert to a real newline, because doing so would produce invalid JSON.

If the admin copies the value of the `format_guide` field straight out of the JSON block and pastes it into MatchLedger, MatchLedger's `<textarea>` receives literal `\n` characters and renders them as text, not line breaks. The format guide saves but reads as one giant unbroken paragraph — ugly, hard to maintain, and triggers the validator's WARNING for "excessive length" because the line-break counts collapse.

Same hazard for Detection Hints: pasting `["Xendit WHT", "PENDING_SETTLEMENT"]` into the chip input makes one chip with brackets and commas in it.

Block 2 and Block 3 exist solely to bypass these two sharp edges.

---

## Implementation hooks for the formatter

When generating the JSON-to-presentation conversion:

```python
def render_paste_blocks(result: FormatterResult) -> tuple[str, str]:
    hints_block = "\n".join(result.detection_hints) if result.detection_hints else "(none — Generic template)"
    guide_block = result.format_guide  # already real newlines from Claude
    return hints_block, guide_block
```

Pseudocode-equivalent for any language. The point: never pass through `json.dumps` for the human blocks — that's what introduces the escape characters.

---

## Test cases the implementation must cover

1. **Hints contain special chars** (commas, quotes, parens). The hints block prints them verbatim — no escaping. `Reference (Trans ID)` paste-friendly version is exactly `Reference (Trans ID)`, not `"Reference (Trans ID)"`.
2. **Format guide contains internal blank line** (paragraph break). Block 3 preserves it as a literal blank line.
3. **Generic template, empty hints array.** Block 2 prints `(none — Generic template)`, prompt for "Copy Detection Hints?" is skipped.
4. **Headless environment.** Clipboard prompt-or-flag triggers, library raises, formatter prints the "unavailable" message and exits 0 (not an error).
5. **`--copy both` ordering.** After the run, the system clipboard contains the hints block (last copy wins). The Format Guide was copied first and overwritten — that's intentional, see flag table above.
