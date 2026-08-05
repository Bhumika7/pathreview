# Journal

## Issue Selected
[Resume section detection fails on text with leading whitespace #147](https://github.com/ascherj/pathreview/issues/147)

## Tier
Tier 1 (labeled `tier-1`, `good first issue`). I'm still getting familiar with this codebase, so I picked a Tier 1 issue on purpose, it's scoped to one method (`_detect_sections()`) in a single file, has a clear reproduction case, and existing tests that define exactly what "fixed" means. That made it a good entry point before taking on anything with broader scope.

## Problem Summary
PathReview tries to detect standard resume section headers ("Experience," "Education," "Skills," etc.) so it can organize parsed resume content. The detection logic only recognizes a header if it sits at the very start of a line with no leading spaces or tabs. Text extracted from PDFs (and some plain-text resumes) is commonly indented, so every line starts with extra whitespace. Because of that mismatch, section detection currently finds nothing on indented text — `detected_sections` comes back empty even when the resume clearly has Education and Skills sections. A successful fix makes detection tolerant of leading whitespace, so indented resumes get their sections recognized correctly.

## Week 8 - Reproduction & solution planning


** I ran the same repro from the issue locally:

```python
from ingestion.parsers.resume_parser import ResumeParser
r = ResumeParser()
res = r.parse('\n    John Smith\n    john@example.com\n\n    Education:\n    - B.S. Computer Science\n\n    Skills: Python\n')
print(res.metadata['detected_sections'])
```

Against the original code, this returned `[]` instead of the expected `['Education', 'Skills']`, confirming the bug. I also ran the three related tests (`test_parse_single_column_resume_text`, `test_parse_resume_no_work_experience`, `test_detect_sections`) and confirmed they failed against the unmodified `_detect_sections()` method, matching what the issue describes.

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:** Completed sub-tasks 1–2 from `PLAN.md`: reproduced the bug locally with the issue's exact repro case (confirmed `detected_sections` returns `[]` on indented text), and ran the three related tests (`test_parse_single_column_resume_text`, `test_parse_resume_no_work_experience`, `test_detect_sections`) against the unmodified code to confirm they fail, establishing a clear before/after baseline.

**Next steps:** Implement sub-tasks 3–5: update the regex patterns in `_detect_sections()` to tolerate leading whitespace, simplify the redundant `\n`-anchored patterns, then add new edge-case tests and re-run the full unit suite to confirm no regressions.

**Blockers:** None.

---

### Check-in 2 (end of week)

**PR link:** [#219 — Fixed section detection for text with leading whitespace](https://github.com/ascherj/pathreview/pull/219)

**Branch:** `fix/147-section-detection-whitespace`

**What you built:** Fixed `_detect_sections()` in `ingestion/parsers/resume_parser.py` so it correctly matches section headers (Experience, Education, Skills, etc.) in text with leading whitespace/indentation, which is common in text extracted from PDFs. The fix adds `[ \t]*` to the anchor patterns and removes two redundant patterns that duplicated what `re.MULTILINE`'s `^` already covers.

**Tests added or updated:** Modified `tests/unit/test_resume_parser.py`, adding 5 new tests: `test_detect_sections_with_leading_whitespace` (indented header followed by a colon), `test_detect_sections_with_leading_whitespace_no_punctuation` (indented header alone on its own line), `test_detect_sections_with_tab_indentation` (tab characters instead of spaces), `test_detect_sections_ignores_keyword_mid_sentence` (confirms a section keyword used mid-sentence, e.g. "years of experience," isn't falsely flagged as a header), and `test_detect_sections_without_leading_whitespace_still_works` (regression guard confirming the original non-indented case still passes). All 8 tests touching `_detect_sections()` (3 from the issue + 5 new) pass.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
(Note: the codebase has pre-existing, unrelated failures — 2 in `test_resume_parser.py` itself from `_strip_markdown()`, and ~48 more across other modules — confirmed unrelated to this change and documented in the PR's Notes for Reviewers. Per the pre-existing-failures policy, these checks reflect that my changes introduce no new failures.)

**Draft PR feedback received from:** None — did not get a chance to request peer/mentor review before the deadline.

