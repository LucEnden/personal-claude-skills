---
name: practice-review
description: Analyzes one or more source files against the project's coding practices defined in CODING_PRACTICES.md and produces a structured Markdown report. Invoke as /lvde:practice-review <file> [file2 ...].
---

# Coding Practice Review

Analyze source files for adherence to the project's coding practices. Write inline `[REVIEW]` markers directly into each source file as violations are found (halt-resilient), then produce a compact summary report.

## When Invoked

The user calls `/lvde:practice-review <file> [file2 file3 ...]` or asks to check files against coding practices.

---

## Step-by-Step Execution

### 1. Resolve inputs

- Parse the args to get target file paths. Relative paths resolve from the project root.
- If no args given, ask the user which file(s) to analyze.

### 2. Read practices

Read `CODING_PRACTICES.md` from the project root. This is the authoritative list of practices and their definitions. Do not invent practices not listed there.

### 3. Get git context

Run:
```bash
git rev-parse HEAD
git remote get-url origin 2>/dev/null || echo ""
```

Store: `COMMIT_HASH` (full), `COMMIT_SHORT` (first 7 chars), `REMOTE_URL` (for GitHub link construction).

To build a GitHub file link:
- Convert `git@github.com:user/repo.git` → `https://github.com/user/repo`
- Link format: `https://github.com/user/repo/blob/<COMMIT_HASH>/<file_path>`

If no remote exists, use a local format: `` `<COMMIT_SHORT>/<filename>` ``

### 4. For each target file (repeat until all done)

Process one file at a time. Complete all sub-steps for a file before moving to the next.

#### 4a. Read the file

Read full contents. Analyze in isolation — do not import context from other files unless the practice definition itself requires it.

#### 4b. Analyze against all practices

For each violation found:
- Identify the line number(s)
- Name the practice violated
- State the violation (1 sentence)
- State the fix (1 sentence or short snippet)
- Assign severity: **High**, **Medium**, or **Low**

Collect all violations for this file before writing any markers.

#### 4c. Write `[REVIEW]` markers into the source file

**Write markers immediately — do not wait until all files are analyzed.**

For each violation, insert a comment on the line immediately above the violating line. Write markers **from the highest line number to the lowest** (bottom-to-top) so earlier insertions do not shift subsequent line numbers.

**Comment syntax:** Derive the correct comment syntax from the file extension and surrounding code context. For mixed-syntax files (e.g. `.tsx` where `//` and `<!-- -->` are both valid depending on location), use whichever syntax is valid at the specific line being annotated. The marker text is always: `[REVIEW-<SEVERITY>]: <violation>. Fix: <suggestion>`

Severity in the marker is `HIGH`, `MEDIUM`, or `LOW` (uppercase).

For a violation spanning multiple non-contiguous lines (e.g., 12, 47, 89), place the marker above the first line of the violation.

For a violation spanning a contiguous range (e.g., 12–18), place the marker above line 12.

Example (TypeScript):
```typescript
// [REVIEW-HIGH]: Triple-nested if/else; only reached after 3 conditions. Fix: extract inner conditions as guard clauses.
if (a) {
  if (b) {
    if (c) { ... }
  }
}
```

If a file has zero violations, skip 4c entirely.

#### 4d. Accumulate summary rows

For each violation written, record:
- File name (short), line(s), severity, practice name, violation summary, suggestion summary

### 5. Output violations

After all files are processed, check for `SPEC.md` at the project root.

#### If SPEC.md exists — append §T task rows

For each violation (High first, then Medium, then Low), append a task row to the §T section of `SPEC.md`.

- Read existing §T rows to find the highest task number `N`; increment for each new row.
- Row format: `T<N>|.|fix <practice>: <violation summary> (<file>:<line>)|`
  - Status `.` = not started
  - Description ≤ 60 chars; truncate if needed
  - Deps: cite relevant §V invariants if any apply, otherwise omit
- If §T section does not exist, add it after the last existing section.

Example rows appended to §T:
```
T12|.|fix ErrorHandling: missing try/catch on fetch (auth.ts:42)|
T13|.|fix NeverNesting: triple-nested if/else (validators.ts:12)|
T14|.|fix NoMagicValues: literal "pending" (validators.ts:89)|V3
```

Tell user: "Added N task(s) to §T in SPEC.md."

#### If SPEC.md absent — write report to `.claude/reports/`

Save the report to `.claude/reports/` (create directory if absent).

Filename: `<first-filename-stem>-practice-review-<YYYY-MM-DD>.md`

Example: `.claude/reports/validators-practice-review-2026-05-25.md`

Report structure:

```markdown
# Coding Practice Review

- **Project:** <project name from package.json or CLAUDE.md>
- **Date:** <ISO 8601 date and time>
- **Analyst:** <model name>

## Files Analyzed

- [<COMMIT_SHORT>/<filename>](<github_link_with_commit>)
- ...

## Violations

| File | Line(s) | Severity | Practice | Violation | Suggestion |
|------|---------|----------|----------|-----------|------------|
| auth.ts | 42 | High | Error Handling | Missing try/catch on fetch | Wrap in try/catch, surface error to caller |
| validators.ts | 12–18 | High | Never Nesting | Triple-nested if/else | Extract as guard clauses |
| validators.ts | 89 | Medium | No Magic Values | Literal `"pending"` | Use `TransactionStatus.PENDING` |

> **Remove `[REVIEW]` markers** from source files after addressing each item. Do not commit markers.
```

Sort Violations table by Severity descending (High → Medium → Low), then by file name.

If a file has no violations, include it in Files Analyzed but omit it from the Violations table. Add a note: "No violations found in `<filename>`."

Tell the user the saved path after writing.

---

## Rules

- Write markers to source files **before** outputting violations. If halted, per-file markers persist.
- Only report practices from `CODING_PRACTICES.md`. Do not invent new ones.
- Violation and Suggestion cells: 1 sentence max each.
- SPEC.md takes priority over `.claude/reports/` — check for it first; only fall back to report file if absent.
- Do NOT wrap the report in a code block — output raw Markdown so it renders correctly.
- Always write output (SPEC.md §T rows or report file). Do not only print to conversation.
