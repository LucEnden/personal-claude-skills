# coding-practice-review

Audits source files against project coding practices and outputs a structured Markdown report.

## Trigger

```
/practice-review <file> [file2 file3 ...]
```

Also triggers on: "check coding practices", "review code style", "audit file for violations".

## Requirements

`CODING_PRACTICES.md` must exist at the project root. The skill reads it as the authoritative list of practices — it never invents new ones.

## What It Does

1. Resolves target file paths (relative to project root)
2. Reads `CODING_PRACTICES.md`
3. Grabs git context (`HEAD` hash + remote URL) for anchored GitHub links
4. Reads each target file in isolation
5. Checks every file against every practice → records violations with line range, severity, and fix
6. Outputs a Markdown report (no code fence — renders directly)

## Report Structure

| Section | Content |
|---------|---------|
| Header | Project, date, analyst/model |
| Files Analyzed | GitHub-linked at exact commit |
| Practices Checked | All practices from `CODING_PRACTICES.md` with one-line descriptions |
| Results | Per-file violation table (Row(s) / Practice / Violation / Suggestion / Severity) |
| Summary | All violations across all files, sorted High → Medium → Low |

## Severity

| Level | Meaning |
|-------|---------|
| High | Correctness risk or major readability break |
| Medium | Style violation with non-trivial impact |
| Low | Minor style / preference |

## Rules

- Only report practices defined in `CODING_PRACTICES.md`
- Analyze each file in isolation — no cross-file blame
- No violations → write "No violations found." and omit the table
- Multi-line violations: comma-separate non-contiguous rows (`12, 47, 89`)
- Violation + Suggestion cells: 1 sentence max each
