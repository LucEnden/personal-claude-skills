# practice-review

Audits source files against project coding practices. Writes inline `[REVIEW]` markers into each source file as violations are found (halt-resilient), then produces a structured Markdown report saved to `.claude/reports/`.

## Trigger

```
/lvde:practice-review <file> [file2 file3 ...]
```

Also triggers on: "check coding practices", "review code style", "audit file for violations".

## Requirements

`CODING_PRACTICES.md` must exist at the project root. The skill reads it as the authoritative list of practices — it never invents new ones.

## What It Does

1. Resolves target file paths (relative to project root)
2. Reads `CODING_PRACTICES.md`
3. Grabs git context (`HEAD` hash + remote URL) for anchored GitHub links
4. For each file: reads it, identifies all violations, writes `[REVIEW]` markers into the source file, accumulates summary rows
5. Writes a Markdown report to `.claude/reports/<first-filename-stem>-practice-review-<YYYY-MM-DD>.md`

## Inline Markers

For each violation, a comment is inserted on the line immediately above the violating line. Comment syntax is derived from the file extension and surrounding code context — for mixed-syntax files (e.g. `.tsx`), whichever syntax is valid at that specific line is used.

Marker format: `[REVIEW-<SEVERITY>]: <violation>. Fix: <suggestion>`

Markers are written before the report — if the skill is halted mid-run, per-file markers already written persist.

> Remove `[REVIEW]` markers from source files after addressing each item. Do not commit markers.

## Report Structure

Saved to `.claude/reports/`, filename: `<first-filename-stem>-practice-review-<YYYY-MM-DD>.md`

| Section | Content |
|---------|---------|
| Header | Project, date, analyst/model |
| Files Analyzed | GitHub-linked at exact commit |
| Violations | All violations across all files sorted High → Medium → Low |

Violations table columns: File / Line(s) / Severity / Practice / Violation / Suggestion

## Severity

| Level | Meaning |
|-------|---------|
| High | Correctness risk or major readability break |
| Medium | Style violation with non-trivial impact |
| Low | Minor style / preference |

## Rules

- Only report practices defined in `CODING_PRACTICES.md`
- Analyze each file in isolation — no cross-file blame
- No violations → include file in Files Analyzed, add note "No violations found in `<filename>`.", omit from table
- Violation + Suggestion cells: 1 sentence max each
