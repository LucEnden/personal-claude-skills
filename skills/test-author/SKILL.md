---
name: test-author
description: Author a test for a source file or feature. Picks the correct test layer, binds the test to a SPEC.md §V/§T cite, halts to confront the engineer when no cite exists, and enforces selector and naming rules.
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

## Step 1 — Identify the SUT

Read the file. Note:
- Exports / public surface (functions, classes, components).
- Imports (signals the layer: database client → likely backend, UI framework → likely component).
- Where it's used (`grep -rn <export-name>`). Knowing the call sites shapes which layer is right.

## Step 2 — Find the SPEC cite

Search `SPEC.md` for relevant `§V.*` invariants and `§T*` tasks. Match by:
- Page-group prefix — map the file's path segment to the corresponding §V group in your project (e.g., a file under `src/checkout/` maps to `V.checkout.*`; a file under `src/settings/` maps to `V.settings.*`).
- Behavior keywords — scan §V lines for verbs that match what the SUT does: validation, pagination, route guard, error state, i18n, selection, sorting, etc.
- Cross-cutting concerns in `§V.cross.*` (or your project's equivalent section) — behaviors that apply app-wide. Every component touches at least one.

> **Adapt:** Before running this skill on a new project, add a §V group map to this section (path prefix → §V prefix). Example: `src/orders/ → V.orders`, `src/auth/ → V.auth`. Without this map, cite resolution requires reading §V in full each time.

A test can cite **multiple** ids (e.g. an integration test for a bulk-delete button cites both `V.items.4` and `V.items.5`). Cite all that genuinely apply; do not pad.

## Step 3 — No cite? Halt and converge.

Use `AskUserQuestion` with these four options (order = recommended first):

1. **Add a new §V invariant.** Draft the proposed invariant text and section, then update `SPEC.md` directly or invoke your project's spec skill (e.g., `/spec amend §V.<section>`).
2. **Tie to an existing related invariant.** List 1-3 candidates from `§V` with rationale ("This is a special case of V.cross.3 because …").
3. **Promote to §V.cross.** If the behavior applies app-wide (loading state shape, error state shape, i18n key usage), the right place is a new `§V.cross.*` line.
4. **Decline the test.** Document the rationale ("not worth covering because …"). No test is written. The rationale gets returned to the engineer.

If the SUT exists because the engineer is fixing a bug, propose `/spec bug:` instead — the bug entry replaces the cite.

Never write a test with a fabricated or stretched cite. The cite relation must actually make sense to a reader who only sees the SPEC line and the test.

## Step 4 — Pick the layer

| SUT shape | Layer | File convention |
|-----------|-------|-----------------|
| Pure function with no UI or DB dependencies | unit | `<name>.unit.test.<ext>` |
| UI component or hook (renders, no navigation) | component or integration | `<name>.component.test.<ext>` or `<name>.integration.test.<ext>` |
| Multi-component flow (form → submit → feedback) | integration | `<name>.integration.test.<ext>` |
| Page-level user journey, route guard, or cross-page nav | e2e | `tests/e2e/<page-group>/<name>.e2e.spec.<ext>` |
| API handler / endpoint | unit (handler called directly) or e2e (HTTP) | `<name>.unit.test.<ext>` or `<name>.e2e.spec.<ext>` |

When in doubt, prefer the lowest layer that can faithfully cover the invariant.

> **Adapt:** Replace `<ext>` with your project's test file extension (`.ts`, `.py`, `.js`, etc.). Adjust directory conventions to match your project layout. If your project uses Storybook, component tests live in `*.stories.<ext>` with a `play()` function.

## Step 5 — Atom check

Search `tests/_atoms/` for any helper whose name matches the behavior you're about to write. If one exists, use it. If not, write the assertion inline — extraction happens later via `/test-atom-extract`, never speculatively.

> **Adapt:** Replace `tests/_atoms/` with your project's shared test helper directory.

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
- **Selectors** (for UI tests using Testing Library or Playwright) in this priority order: `getByRole` → `getByLabel` → `getByPlaceholderText` → `getByText` → `getByTestId`. Never: CSS class selectors, tag selectors, `nth-child`, structural combinators. `getByTestId` only if no other option, with a `// testid: <reason>` comment.
- **Strings** — resolve user-visible text via your project's i18n test helper rather than hardcoding display strings in selectors or assertions. If the project has no i18n, use named constants instead of raw strings.

  > **Adapt:** Replace references to the i18n helper (e.g., `t('key')`) with your project's actual helper path and function.

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

Run the new test in isolation using your project's test commands, for example:

- Unit / integration: `npm run test:unit -- <file>` or equivalent
- Component: `npm run test:component -- <file>` or equivalent
- E2E: `npm run test:e2e -- <file>` or equivalent

After green, run `npm run test:spec` (or equivalent) to confirm the cite registers in the coverage report.

> **Adapt:** Replace `npm run test:*` with your project's actual test runner commands.

## Vendor Playwright skills

If the test is E2E and needs network mocking, multi-session, codegen, or trace inspection, defer to the vendor Playwright agent skills installed via `npx playwright-cli install --skills`. Do not duplicate their guidance here.

## What this skill never does

- Invent a cite that doesn't reflect the test's actual semantics.
- Add a `data-testid` (or equivalent test hook) to source code to make a test easier. Use roles + labels. Only add a testid when the SUT genuinely lacks an accessible name, and call that out separately for the engineer.
- Write a test that asserts implementation details (internal state, function call counts on third-party libs).
- Pre-build atoms before the second occurrence.
