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

