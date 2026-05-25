# test-atom-extract

Scans the test tree for duplicated setup-then-assert patterns and proposes extracting them into a shared test helper function. Read-only proposal first — no file is written until you approve.

## Triggers

```
/test-atom-extract
"extract test atom"
"find duplicate tests"
```

## What it does

1. **Bootstraps conventions** (first run only) — scans the project to derive shared helper directory, fixtures directory, and test file pattern. Asks you to confirm findings, then writes `.claude/test-conventions.md`. Subsequent runs load from that file instantly.
2. **Scans** all test files for repeated patterns: identical 3+ statement sequences containing at least one assertion, repeated across different files.
3. **Clusters** matches by behavior (not by file or SUT).
4. **Filters** to clusters with ≥ 2 occurrences across different files. Same-file duplication is a parameterized test candidate, not a shared helper.
5. **Proposes** a helper name, target file (grouped by behavior domain), and signature. Shows a diff for each affected test site.
6. **Waits for approval** before writing anything.

## What it produces

- One or more small helper functions in the project's shared helper directory, grouped by behavior domain (e.g. `validation`, `auth`, `pagination`).
- Rewrites of each call site to use the extracted helper.

## Hard rules

- One helper = one assertion concept. If the description needs "and", split it.
- Helper signature takes identifiers and string constants as parameters — never hardcoded values.
- No class hierarchies or `BaseHelper` superclasses.
- If candidate bodies diverge by more than 1-2 parameters, do not extract.

## Out of scope

- Multi-step setup sequences or navigation flows — those are fixtures or flow helpers, not atoms.
- Writing new tests — use `/test-author` for that.

## Shared conventions

Both `test-atom-extract` and `test-author` read from and write to `.claude/test-conventions.md`. Whichever skill runs first on a project bootstraps the file. The other inherits it at no cost.
