---
name: test-author
description: Author a test for a source file or feature. Requires SPEC.md. Derives project conventions on first run, verifies with engineer, caches to .claude/test-conventions.md. Picks the correct test layer, binds the test to a SPEC.md §V/§T cite, and halts when no cite exists.
triggers:
  - "/test-author"
  - "write a test for"
  - "test this file"
  - "add tests for"
---

# test-author

Authors a single test (or test file) for a given source file or feature.

The skill is **always SPEC-bound**: no cite, no test. If the SPEC has no relevant invariant, the skill **stops and converges with the engineer** before writing anything.

## Inputs

- A file path (`/test-author src/lib/validation.py`) or
- A feature description (`/test-author "checkout form must show error when card is expired"`).

If neither given, ask the engineer.

## Step 0 — Load or derive conventions

Check for `.claude/test-conventions.md` in the project root.

**File exists** → load it and proceed to Step 1.

**File missing** → derive conventions from the project:

1. **Shared helper directory** — scan the test tree for directories that aggregate reusable helper or utility files. Look for directory names such as `_atoms`, `atoms`, `helpers`, `utils`, `support`, `shared`, `common`, `__helpers__`. Also check if any directory is imported by three or more test files (strong signal). If none found, propose `tests/helpers/` as default.

2. **Fixtures directory** — look for directories named `_fixtures`, `fixtures`, `__fixtures__`, `factories`, `mocks`, `__mocks__`, `stubs`. If multiple candidates exist, pick the one most commonly imported.

3. **Test file pattern** — glob for `**/*.test.*` and `**/*.spec.*`. Take the most common extension (`.ts`, `.py`, `.js`, `.go`, etc.). Note the directory depth and naming convention used by existing test files.

4. **Test runner commands** — check in order: `package.json` scripts for `test`, `test:unit`, `test:integration`, `test:e2e`; `Makefile` for test targets; `pytest.ini` / `pyproject.toml` / `setup.cfg`; `go.mod` (implies `go test ./...`). Record one command per layer present.

5. **i18n helper** — grep test files for common i18n call patterns (`t(`, `i18n.t(`, `useTranslation`, `gettext`, `_(`). If found, identify the import path and function name. If not found, record as none.

6. **Module-group map** — read `SPEC.md` §V section headings and match them to `src/` subdirectory names. Record each match as `<path-prefix> → <§V-prefix>`.

Then **halt and verify**: present all findings to the engineer via `AskUserQuestion`. Show each derived value, ask for corrections. After confirmation, write `.claude/test-conventions.md` using the format below. All subsequent runs load from this file — no re-derivation.

### .claude/test-conventions.md format

```markdown
# Test conventions
<!-- Written by test-author / test-atom-extract. Edit to update. -->

shared_helper_dir: <path>
fixtures_dir: <path>
test_glob: <glob pattern, e.g. **/*.test.ts>
runner_unit: <command>
runner_integration: <command>
runner_e2e: <command>
i18n_helper: <import path>::<function> | none
spec_map: |
  src/orders/ → V.orders
  src/auth/ → V.auth
```

## Step 1 — Identify the SUT

Read the file. Note:
- Exports / public surface (functions, classes, components).
- Imports (signals the layer: database client → likely backend, UI framework → likely component).
- Where it's used (`grep -rn <export-name>`). Knowing the call sites shapes which layer is right.

## Step 2 — Find the SPEC cite

Search `SPEC.md` for relevant `§V.*` invariants and `§T*` tasks. Match by:
- Module-group prefix — use `spec_map` from the conventions file to map the file's path segment to its §V group.
- Behavior keywords — scan §V lines for verbs that match what the SUT does: validation, pagination, access control / route guard, error state, i18n, selection, sorting, etc.
- Cross-cutting concerns in `§V.cross.*` (or your project's equivalent section) — behaviors that apply app-wide. Every module touches at least one.

A test can cite **multiple** ids (e.g. an integration test for a bulk-delete action cites both `V.items.4` and `V.items.5`). Cite all that genuinely apply; do not pad.

## Step 3 — No cite? Halt and converge.

Use `AskUserQuestion` with these four options (order = recommended first):

1. **Add a new §V invariant.** Draft the proposed invariant text and section, then update `SPEC.md` directly or invoke your project's spec skill (e.g., `/spec amend §V.<section>`).
2. **Tie to an existing related invariant.** List 1-3 candidates from `§V` with rationale ("This is a special case of V.cross.3 because …").
3. **Promote to §V.cross.** If the behavior applies app-wide (loading state shape, error state shape, auth/permission checks, logging behavior, i18n key usage), the right place is a new `§V.cross.*` line.
4. **Decline the test.** Document the rationale ("not worth covering because …"). No test is written. The rationale gets returned to the engineer.

If the SUT exists because the engineer is fixing a bug, propose `/spec bug:` instead — the bug entry replaces the cite.

Never write a test with a fabricated or stretched cite. The cite relation must actually make sense to a reader who only sees the SPEC line and the test.

## Step 4 — Pick the layer

| SUT shape | Layer | File convention |
|-----------|-------|-----------------|
| Pure function with no UI or DB dependencies | unit | `<name>.unit.test.<ext>` |
| UI component or reactive primitive (renders, no navigation) | component or integration | `<name>.component.test.<ext>` or `<name>.integration.test.<ext>` |
| Service or repository function (business logic, data access) | unit or integration | `<name>.unit.test.<ext>` or `<name>.integration.test.<ext>` |
| Multi-unit flow (e.g. form → submit → feedback, or request → handler → response) | integration | `<name>.integration.test.<ext>` |
| CLI command / script entry point | integration | `<name>.integration.test.<ext>` |
| Background job, worker, or task | unit or integration | `<name>.unit.test.<ext>` or `<name>.integration.test.<ext>` |
| Full user journey, access control check, or cross-boundary navigation | e2e | `tests/e2e/<feature-group>/<name>.e2e.spec.<ext>` |
| API handler / endpoint | unit (handler called directly) or e2e (HTTP) | `<name>.unit.test.<ext>` or `<name>.e2e.spec.<ext>` |

Use `test_glob` from conventions to confirm the correct extension. When in doubt, prefer the lowest layer that can faithfully cover the invariant.

## Step 5 — Shared helper check

Search `shared_helper_dir` from conventions for any helper whose name matches the behavior you're about to write. If one exists, use it. If not, write the assertion inline — extraction happens later via `/test-atom-extract`, never speculatively.

## Step 6 — Write the test

### Scenarios to cover

For every behavior being tested, consider all four scenario classes before writing. Do not default to happy path only.

| Class | What it tests | When to include |
|-------|---------------|-----------------|
| **Happy path** | Normal input, expected outcome, system in valid state | Always — this is the baseline |
| **Sad path** | Invalid input, failed operations, system rejects gracefully (error message, fallback, correct status code) | Always — a feature is not tested until its failure mode is too |
| **Edge case** | Boundary values: empty input, zero, max length, first/last item, exactly-at-limit | When the SUT has a boundary condition (length check, numeric range, empty state) |
| **Corner case** | Unusual but valid combination of inputs that interact in non-obvious ways | When two or more parameters can combine to produce unexpected behavior |
| **Fuzz / random testing** | Randomly generated or malformed input to surface unexpected failures | **Only with explicit engineer approval.** Random inputs introduce non-determinism that makes CI fragile and failures hard to reproduce. Prefer deterministic edge/corner cases first. Only add fuzzing when the SUT processes untrusted external input and the invariant genuinely cannot be covered exhaustively. |

Each scenario class that applies gets its own `test(…)` call — do not merge happy + sad into one assertion block.

Hard requirements:

- **Filename** carries the layer suffix from Step 4.
- **Identifiers / selectors:**
  - For UI tests (Testing Library, Playwright): prefer accessible selectors in this priority order: `getByRole` → `getByLabel` → `getByPlaceholderText` → `getByText` → `getByTestId`. Never: CSS class selectors, tag selectors, `nth-child`, structural combinators. `getByTestId` only if no other option, with a `// testid: <reason>` comment.
  - For non-UI tests: use stable identifiers from the public API surface — IDs, keys, return values. Never couple tests to internal implementation names.
- **Strings** — if `i18n_helper` is set in conventions, resolve user-visible strings through it rather than hardcoding display strings in selectors or assertions. If `i18n_helper` is none, use named constants instead of raw strings.
- **Cite** mandatory:
  - Vitest: `test('should …', { meta: { spec: 'V.x.y' } }, async () => {…})`. Multiple cites = `spec: ['V.x.y', 'V.cross.1']`.
  - Playwright: tag in title: `test('@spec:V.x.y should …', async ({ page }) => {…})`. Multiple = include several `@spec:…` tags.
  - Other runners: embed the cite in a comment immediately above the test: `// @spec V.x.y`.
- **AAA blocks** for unit + integration tests:
  ```
  // arrange
  …
  // act
  …
  // assert
  …
  ```
  For E2E with multiple steps where AAA collapses unnaturally, use BDD blocks: `// scenario:`, `// given`, `// when`, `// then`.
- **Test name** = behavior as outcome, starts with `should …`. Never `tests handleSubmit` or `works`.
- One test asserts one outcome. If two assertions verify two different behaviors, write two tests.

## Step 7 — Verify

Run the new test in isolation using the runner command from conventions:

- Unit: `runner_unit -- <file>`
- Integration: `runner_integration -- <file>`
- E2E: `runner_e2e -- <file>`

After green, confirm the cite registers in the coverage report if the project has spec-coverage tooling.

## Vendor E2E skills

If the test is E2E and needs network mocking, multi-session, codegen, or trace inspection, check whether your project has dedicated E2E helper skills or plugins installed. Do not duplicate their guidance here.

## What this skill never does

- Invent a cite that doesn't reflect the test's actual semantics.
- Add a test hook attribute (e.g. `data-testid`) to UI source code to make a test easier. Use roles and labels. Only add a test hook when the SUT genuinely lacks an accessible name, and call that out separately for the engineer.
- Write a test that asserts implementation details (internal state, function call counts on third-party libs).
- Pre-build shared helpers before the second occurrence.
