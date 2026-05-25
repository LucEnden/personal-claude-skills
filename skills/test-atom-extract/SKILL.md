---
name: test-atom-extract
description: Scan the test tree for duplicated setup-then-assert patterns and propose extracting them into a shared test helper function. Derives project conventions on first run, verifies with engineer, caches to TEST_CONVENTIONS.md. Read-only proposal first; engineer approves before any file is written.
triggers:
  - "/test-atom-extract"
  - "extract test atom"
  - "find duplicate tests"
---

# test-atom-extract

A *test helper atom* is a small, single-concept shared function that encapsulates a repeated assertion or setup pattern — one behavior, one function, used across multiple test files.

This skill finds test code duplication and proposes extracting it into a shared helper directory.

## Step 0 — Load or derive conventions

Check for `TEST_CONVENTIONS.md` in the project root.

**File exists** → load it and skip to the Process section below.

**File missing** → derive conventions from the project:

1. **Shared helper directory** — scan the test tree for directories that aggregate reusable helper or utility files. Look for directory names such as `_atoms`, `atoms`, `helpers`, `utils`, `support`, `shared`, `common`, `__helpers__`. Also check if any directory is imported by three or more test files (strong signal). If none found, propose `tests/helpers/` as default.

2. **Fixtures directory** — look for directories named `_fixtures`, `fixtures`, `__fixtures__`, `factories`, `mocks`, `__mocks__`, `stubs`. If multiple candidates exist, pick the one most commonly imported.

3. **Test file pattern** — glob for `**/*.test.*` and `**/*.spec.*`. Take the most common extension (`.ts`, `.py`, `.js`, `.go`, etc.). Note the directory depth and naming convention used by existing test files.

4. **Existing behavior domains** — list any subdirectory names already present inside the shared helper directory. These become the known behavior domains (e.g. `validation`, `auth`, `pagination`).

Then **halt and verify**: present all findings to the engineer via `AskUserQuestion`. Show each derived value, ask for corrections. After confirmation, write `TEST_CONVENTIONS.md` using the format below. All subsequent runs load from this file — no re-derivation.

### TEST_CONVENTIONS.md format

```markdown
# Test conventions
<!-- Written by test-atom-extract / test-author. Edit to update. -->

shared_helper_dir: <path>
fixtures_dir: <path>
test_glob: <glob pattern, e.g. **/*.test.ts>
```

## Process

1. **Scan** all files matching `test_glob` for repeated patterns. Look for:
   - Identical sequences of 3+ statements that include at least one assertion.
   - Identical fixture setup followed by identical assertions on the same subject, field, or identifier.
2. **Cluster** matches by behavior, not by SUT. Group "required-field validation" hits even when they appear in different test files.
3. **Filter**: keep clusters with **≥ 2 occurrences across different test files**. Discard same-file duplication (that's a parameterized test candidate, not a shared helper).
4. **Propose** for each surviving cluster:
   - Helper name (behavior-as-verb, e.g. `assertRequiredField`).
   - Target file within `shared_helper_dir`, grouped by behavior domain. Use an existing domain from the conventions file if one fits; otherwise propose a new domain name.
   - Signature (inputs are identifiers and string constants or keys needed to locate and verify the subject; never hardcode values that vary per call site).
   - Diff for each test site showing the rewrite.
5. **Halt and ask** via `AskUserQuestion`: accept / reject / rename / merge with existing helper. Only after explicit approval does the skill write.

## Hard rules

- One helper = one assertion concept. Split if a single function needs "and" in its description.
- Helper signature accepts identifiers and string constants as parameters. It does not hardcode either.
- Helper does not introduce new layers of indirection (no class hierarchies, no `BaseHelper` superclasses).
- If a candidate cluster's body diverges by more than 1-2 parameters, do not extract. The "near-identical" cases stay inline until they actually converge.

## Out of scope

- Complex shared setup sequences or flow helpers (e.g. multi-step navigation, repeated test scaffolding). Those belong in `fixtures_dir` or a flow helper directory, not in a single-concept assertion helper.
- Authoring new tests. This skill only refactors existing ones. New tests are handled by `/test-author`.