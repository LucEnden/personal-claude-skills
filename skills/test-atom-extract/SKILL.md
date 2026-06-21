---
name: test-atom-extract
description: Scan the test tree for duplicated setup-then-assert patterns and propose extracting them into a shared test helper function. Derives project conventions on first run, verifies with engineer, caches to TEST_CONVENTIONS.md. Read-only proposal first; engineer approves before any file is written.
triggers:
  - "/test-atom-extract"
  - "extract test atom"
  - "find duplicate tests"
---

# test-atom-extract

*Test helper atom* = small, single-concept shared function encapsulating repeated assertion or setup pattern — one behavior, one function, multiple test files.

Skill finds test code duplication, proposes extracting into shared helper directory.

## Step 0 — Load or derive conventions

Check for `TEST_CONVENTIONS.md` in project root.

**File exists** → load it, skip to Process below.

**File missing** → derive from project:

1. **Shared helper directory** — scan test tree for directories aggregating reusable helper/utility files. Look for names: `_atoms`, `atoms`, `helpers`, `utils`, `support`, `shared`, `common`, `__helpers__`. Also check if any directory imported by 3+ test files (strong signal). None found → propose `tests/helpers/` default.

2. **Fixtures directory** — look for `_fixtures`, `fixtures`, `__fixtures__`, `factories`, `mocks`, `__mocks__`, `stubs`. Multiple candidates: pick most commonly imported.

3. **Test file pattern** — glob `**/*.test.*` and `**/*.spec.*`. Take most common extension (`.ts`, `.py`, `.js`, `.go`, etc.). Note directory depth and naming convention.

4. **Existing behavior domains** — list subdirectory names inside shared helper directory. These become known behavior domains (e.g. `validation`, `auth`, `pagination`).

**Halt and verify**: present findings to engineer via `AskUserQuestion`. Show each derived value, ask for corrections. After confirmation, write `TEST_CONVENTIONS.md` using format below. All subsequent runs load from this file — no re-derivation.

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
   - Identical sequences of 3+ statements including at least one assertion.
   - Identical fixture setup followed by identical assertions on same subject, field, or identifier.
2. **Cluster** matches by behavior, not by SUT. Group "required-field validation" hits across different test files.
3. **Filter**: keep clusters with **≥ 2 occurrences across different test files**. Discard same-file duplication (parameterized test candidate, not shared helper).
4. **Propose** per surviving cluster:
   - Helper name (behavior-as-verb, e.g. `assertRequiredField`).
   - Target file within `shared_helper_dir`, grouped by behavior domain. Use existing domain from conventions file if fits; else propose new domain name.
   - Signature (inputs = identifiers and string constants/keys to locate and verify subject; never hardcode values varying per call site).
   - Diff per test site showing rewrite.
5. **Halt and ask** via `AskUserQuestion`: accept / reject / rename / merge with existing helper. Skill writes only after explicit approval.

## Hard rules

- One helper = one assertion concept. Split if function needs "and" in description.
- Helper signature accepts identifiers and string constants as parameters. No hardcoded values.
- No new indirection layers (no class hierarchies, no `BaseHelper` superclasses).
- Candidate cluster body diverges by more than 1-2 parameters → don't extract. Near-identical cases stay inline until they actually converge.

## Out of scope

- Complex shared setup sequences or flow helpers (multi-step navigation, repeated test scaffolding) — those belong in `fixtures_dir` or flow helper directory, not single-concept assertion helper.
- Authoring new tests. Skill only refactors existing ones. New tests handled by `/test-author`.