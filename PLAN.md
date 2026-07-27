## Solution plan

**Issue:** [Resume section detection fails on text with leading whitespace #147](https://github.com/ascherj/pathreview/issues/147)

### Understand
`_detect_sections()` in `ingestion/parsers/resume_parser.py` builds regex patterns for each keyword in `SECTION_HEADERS` and searches for them using `^{section}` or `\n{section}` anchors under `re.MULTILINE`. These patterns require the header text to begin immediately at the start of a line, with no characters before it. Text extracted from PDFs (via `pypdf`'s `extract_text()`) and hand-indented plain-text resumes commonly preserve leading spaces/tabs on every line. Because the patterns don't account for that whitespace, `re.search()` never matches, so `_detect_sections()` returns an empty list even when the text clearly contains section headers.

Root cause: missing whitespace tolerance in the regex patterns — not a problem with the section keyword list or the overall detection strategy.

Expected: `detected_sections` includes `Education` and `Skills` for the reproduction input.
Actual: `detected_sections` returns `[]`.


Files I expect to touch:
- `ingestion/parsers/resume_parser.py` — the `_detect_sections()` method, where the regex patterns are built and matched. This is the only method that needs to change.
- `tests/unit/test_resume_parser.py` — no new tests needed; three existing tests (`test_parse_single_column_resume_text`, `test_parse_resume_no_work_experience`, `test_detect_sections`) already define the expected passing behavior and currently fail against the original code.
- `ingestion/parsers/base.py` — reference only (defines `ParseResult`/`BaseParser`), not modified.

### Plan
1. Reproduce the bug locally using the exact repro from the issue and confirm `detected_sections` returns `[]`.
2. Run the three related tests and confirm they fail against the original code, to establish a clear "before" baseline.
3. Update the regex patterns in `_detect_sections()` to allow optional leading whitespace (`[ \t]*`) between the start of a line and the section keyword.
4. Simplify the pattern set: since `^` already matches the start of every line under `re.MULTILINE`, drop the redundant `\n`-prefixed patterns instead of duplicating the whitespace fix in four places.
5. Re-run the three related tests plus the full unit suite (`make test-unit`) to confirm the fix resolves the issue without regressing markdown-stripping or PDF-parsing behavior.

### Inputs & outputs
**Function:** `_detect_sections(self, text: str) -> list[str]`
- Input: raw resume text (string), which may or may not be indented.
- Output: list of detected section names in title case (e.g., `["Education", "Skills"]`).

Reproduction case:
- Input: `'\n    John Smith\n    john@example.com\n\n    Education:\n    - B.S. Computer Science\n\n    Skills: Python\n'`
- Before fix: `[]`
- After fix: `['Education', 'Skills']`

### Risks & unknowns
1. Whitespace-tolerant patterns could over-match if a section keyword happened to appear mid-sentence with only whitespace before it earlier in a line. Checked against the existing test fixtures — doesn't occur in current test data, but worth a second look if new tests are added later.
2. PDFs could use non-standard whitespace characters (e.g., non-breaking spaces) that `[ \t]*` wouldn't catch. Not observed in current test fixtures; flagged as a possible future edge case rather than something blocking this fix.
3. Removing the redundant `\n`-anchored patterns changes the method's internals. Confirmed via testing that `^` under `re.MULTILINE` covers the same line starts, but double-checking test coverage stays green after the simplification is part of the plan.

### Edge cases
- Indented header followed immediately by a colon (e.g., `    Skills:`) — must still match.
- Indented header with no trailing punctuation, alone on its own line (e.g., `    Summary`) — must still match.
- Text with no leading whitespace at all (the original, already-working case) — must continue to work unchanged.
- Section keyword appearing inside a sentence rather than as a header (e.g., "years of experience") — should not be falsely flagged; the patterns still require anchoring at the line start plus trailing punctuation/end-of-line, so this stays safe.
