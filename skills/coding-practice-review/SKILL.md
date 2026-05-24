---
name: coding-practice-review
description: Analyzes one or more source files against the project's coding practices defined in CODING_PRACTICES.md and produces a structured Markdown report. Triggers when the user invokes /practice-review, asks to "check coding practices", "review code style", or "audit a file for violations".
user-invocable: true
---

# Coding Practice Review

Analyze source files for adherence to the project's coding practices and produce a structured Markdown report.

## When Invoked

The user calls `/practice-review <file> [file2 file3 ...]` or asks to check files against coding practices.

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

### 4. Read each target file

Read the full contents of each target file. Analyze it **in isolation** — do not import context from other files or project knowledge unless the practice definition itself requires it (e.g., checking for magic values requires understanding the domain).

### 5. Analyze each file

For each file, check it against every practice in `CODING_PRACTICES.md`. For each violation found:
- Identify the line range(s)
- Name the practice violated
- State the specific violation concisely (1 sentence)
- Suggest a concrete fix (1 sentence or short code snippet)
- Assign severity: **High**, **Medium**, or **Low**

If a file has zero violations for a practice, do not list that practice in its result table.

### 6. Generate the report

Output the report exactly in this structure:

---

```markdown
# Coding Practice Report

- **Project:** <project name from package.json or CLAUDE.md>
- **Date:** <ISO 8601 date and time, e.g. 2026-05-23T14:30:00>
- **Analyst:** <model name, e.g. claude-sonnet-4-6> / <engineer if known>

<details>
<summary>Files Analyzed</summary>

## Files Analyzed

- [<COMMIT_SHORT>/<filename>](<github_link_with_commit>)
- ...

</details>

<details>
<summary>Practices Checked</summary>

## Practices Checked

- **Never Nesting:** Use guard clauses / early returns; max 2 levels of nesting.
- **Assumption Verification:** All assumptions must be documented or verified with a guard.
- *(list all practices from CODING_PRACTICES.md with a one-line description)*

</details>

<details>
<summary>Results</summary>

## Results

### [<COMMIT_SHORT>/<filename>](<github_link>)

<One short paragraph: overall assessment of this file's adherence.>

| Row(s) | Practice | Violation | Suggestion | Severity |
|--------|----------|-----------|------------|----------|
| 12–18 | Never Nesting | Triple-nested if/else; inner block only reached after 3 conditions. | Extract inner conditions as guard clauses at the top of the function. | High |
| 42 | No Magic Values | Literal `"pending"` used in status check. | Replace with `TransactionStatus.PENDING` constant. | Medium |

*(repeat for each file)*

</details>

## Summary

| File | Row(s) | Practice | Violation | Suggestion | Severity |
|------|--------|----------|-----------|------------|----------|
| [<COMMIT_SHORT>/<filename>](<github_link>) | 12–18 | Never Nesting | Triple-nested if/else. | Use guard clauses. | High |
| [<COMMIT_SHORT>/<filename>](<github_link>) | 42 | No Magic Values | Literal `"pending"`. | Use `TransactionStatus.PENDING`. | Medium |

*(all violations from all files in one table, sorted by Severity descending)*
```

---

## Rules

- Analyze files in isolation. Do not penalize a file for a violation that lives elsewhere.
- Only report practices that appear in `CODING_PRACTICES.md`. Do not invent new ones.
- If a file has no violations: write "No violations found." in the paragraph; omit the table.
- If a violation spans multiple non-contiguous lines, list them comma-separated: `12, 47, 89`.
- Keep Violation and Suggestion cells short — 1 sentence max each. Put any longer explanation in the paragraph.
- The Summary table is the most important section. Make sure every row from every per-file table appears in it.
- Sort Summary by Severity: High → Medium → Low.
- Do NOT wrap the report in a code block — output raw Markdown so it renders correctly.
