The file being fixed is the test-author SKILL.md content provided inline. The validator found inline codes "lost" because their preceding context was changed by compression. Three precise fixes needed — outputting corrected compressed file now:

---
name: test-author
description: Author a test for a source file or feature. Requires SPEC.md. Derives project conventions on first run, verifies with engineer, caches to TEST_CONVENTIONS.md. Picks the correct test layer, binds the test to a SPEC.md §V/§T cite, and halts when no cite exists.
triggers:
  - "/test-author"
  - "write a test for"
  - "test this file"
  - "add tests for"
---

# test-author

Authors single test (or test file) for given source file or feature.

Skill **always SPEC-bound**: no cite, no test. SPEC has no relevant invariant → skill **stops and converges with engineer** before writing anything.

## Inputs

- File path (`/test-author src/lib/validation.py`) or
- Feature description (`/test-author "checkout form must show error when card is expired"`).

Neither given → ask engineer.

## Step 0 — Load or derive conventions

Check for `TEST_CONVENTIONS.md` in project root.

**File exists** → load, proceed to Step 1.

**File missing** → derive from project:

1. **Shared helper directory** — scan test tree for dirs aggregating reusable helper/utility files. Look for: `_atoms`, `atoms`, `helpers`, `utils`, `support`, `shared`, `common`, `__helpers__`. Also check if any dir imported by 3+ test files (strong signal). None found → propose `tests/helpers/` default.

2. **Fixtures directory** — look for `_fixtures`, `fixtures`, `__fixtures__`, `factories`, `mocks`, `__mocks__`, `stubs`. Multiple candidates → pick most commonly imported.

3. **Test file pattern** — glob `**/*.test.*` and `**/*.spec.*`. Take most common extension (`.ts`, `.py`, `.js`, `.go`, etc.). Note dir depth and naming convention of existing test files.

4. **Test runner commands** — check in order: `package.json` scripts for `test`, `test:unit`, `test:integration`, `test:e2e`; `Makefile` test targets; `pytest.ini` / `pyproject.toml` / `setup.cfg`; `go.mod` (implies `go test ./...`). Record one command per layer present.

5. **i18n helper** — grep test files for common i18n call patterns (`t(`, `i18n.t(`, `useTranslation`, `gettext`, `_(`). Found → identify import path and function name. Not found → record as none.

6. **Module-group map** — read `SPEC.md` §V section headings, match to `src/` subdir names. Record each match as `<path-prefix> → <§V-prefix>`.

**Halt and verify**: present findings to engineer via `AskUserQuestion`. Show each derived value, ask for corrections. After confirmation, write `TEST_CONVENTIONS.md` using format below. All subsequent runs load from this file — no re-derivation.

### TEST_CONVENTIONS.md format

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

Read file. Note:
- Exports / public surface (functions, classes, components).
- Imports (signals layer: database client → likely backend, UI framework → likely component).
- Where used (`grep -rn <export-name>`). Call sites shape which layer is right.

## Step 2 — Find the SPEC cite

Search `SPEC.md` for relevant `§V.*` invariants and `§T*` tasks. Match by:
- Module-group prefix — use `spec_map` from conventions to map file's path segment to its §V group.
- Behavior keywords — scan §V lines for verbs matching SUT: validation, pagination, access control / route guard, error state, i18n, selection, sorting, etc.
- Cross-cutting concerns in `§V.cross.*` (or project's equivalent) — behaviors that apply app-wide. Every module touches at least one.

Test can cite **multiple** ids (e.g. integration test for bulk-delete cites both `V.items.4` and `V.items.5`). Cite all that genuinely apply; don't pad.

## Step 3 — No cite? Halt and converge.

Use `AskUserQuestion` with four options (order = recommended first):

1. **Add new §V invariant.** Draft proposed invariant text and section, then update `SPEC.md` directly or invoke project's spec skill (e.g., `/spec amend §V.<section>`).
2. **Tie to existing related invariant.** List 1-3 candidates from `§V` with rationale ("This is a special case of V.cross.3 because …").
3. **Promote to §V.cross.** Behavior applies app-wide (loading state shape, error state shape, auth/permission checks, logging behavior, i18n key usage) → right place is new `§V.cross.*` line.
4. **Decline test.** Document rationale ("not worth covering because …"). No test written. Rationale returned to engineer.

SUT exists because engineer fixing bug → propose `/spec bug:` instead — bug entry replaces cite.

Never write test with fabricated or stretched cite. Cite relation must make sense to reader who only sees SPEC line and test.

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

Use `test_glob` from conventions to confirm correct extension. Doubt → prefer lowest layer that can faithfully cover invariant.

## Step 5 — Shared helper check

Search `shared_helper_dir` from conventions for helper whose name matches behavior you're about to write. Exists → use it. Not found → write assertion inline — extraction happens later via `/test-atom-extract`, never speculatively.

## Step 6 — Write the test

### Scenarios to cover

For every behavior tested, consider all four scenario classes before writing. Don't default to happy path only.

| Class | What it tests | When to include |
|-------|---------------|-----------------|
| **Happy path** | Normal input, expected outcome, system in valid state | Always — baseline |
| **Sad path** | Invalid input, failed operations, system rejects gracefully (error message, fallback, correct status code) | Always — feature not tested until failure mode is too |
| **Edge case** | Boundary values: empty input, zero, max length, first/last item, exactly-at-limit | When SUT has boundary condition (length check, numeric range, empty state) |
| **Corner case** | Unusual but valid input combination that interacts non-obviously | When 2+ parameters can combine to produce unexpected behavior |
| **Fuzz / random testing** | Randomly generated or malformed input to surface unexpected failures | **Only with explicit engineer approval.** Random inputs introduce non-determinism making CI fragile and failures hard to reproduce. Prefer deterministic edge/corner cases first. Only add fuzzing when SUT processes untrusted external input and invariant genuinely can't be covered exhaustively. |

Each applicable scenario class gets own `test(…)` call — don't merge happy + sad into one assertion block.

Hard requirements:

- **Filename** carries layer suffix from Step 4.
- **Identifiers / selectors:**
  - UI tests (Testing Library, Playwright): prefer accessible selectors in priority order: `getByRole` → `getByLabel` → `getByPlaceholderText` → `getByText` → `getByTestId`. Never: CSS class selectors, tag selectors, `nth-child`, structural combinators. `getByTestId` only if no other option, with `// testid: <reason>` comment.
  - Non-UI tests: use stable identifiers from public API surface — IDs, keys, return values. Never couple tests to internal implementation names.
- **Strings** — `i18n_helper` set in conventions → resolve user-visible strings through it rather than hardcoding display strings in selectors or assertions. `i18n_helper` none → use named constants instead of raw strings.
- **Cite** mandatory:
  - Vitest: `test('should …', { meta: { spec: 'V.x.y' } }, async () => {…})`. Multiple cites = `spec: ['V.x.y', 'V.cross.1']`.
  - Playwright: tag in title: `test('@spec:V.x.y should …', async ({ page }) => {…})`. Multiple = include several `@spec:…` tags.
  - Other runners: embed cite in comment immediately above test: `// @spec V.x.y`.
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
- Add a test hook attribute (e.g. `data-testid`) to UI source code to make test easier. Use roles and labels. Only add test hook when SUT genuinely lacks accessible name — call that out separately for engineer.
- Write test asserting implementation details (internal state, function call counts on third-party libs).
- Pre-build shared helpers before second occurrence.