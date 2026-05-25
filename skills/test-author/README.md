# test-author

Authors a single test (or test file) for a given source file or feature. Always SPEC-bound — no test is written without a `SPEC.md` cite.

## Prerequisites

- `SPEC.md` must exist in the project root.

## Triggers

```
/test-author
"write a test for"
"test this file"
"add tests for"
```

## Inputs

Provide either a file path or a feature description:

```
/test-author src/lib/validation.py
/test-author "checkout must reject expired cards"
```

## What it does

1. **Bootstraps conventions** (first run only) — scans the project to derive shared helper directory, fixtures directory, test file pattern, runner commands per layer, i18n helper, and a §V group map from `SPEC.md` headings. Asks you to confirm findings, then writes `TEST_CONVENTIONS.md`. Subsequent runs load from that file instantly.
2. **Identifies the SUT** — reads the file, notes exports and imports to determine the right test layer.
3. **Finds a SPEC cite** — searches `SPEC.md` for a matching `§V.*` invariant using the `spec_map` from conventions. Halts if none found and asks whether to add an invariant, tie to an existing one, promote to `§V.cross`, or decline.
4. **Picks the test layer** — unit, integration, or e2e based on what the SUT does (pure function, service, UI component, CLI command, background job, full journey, API endpoint).
5. **Checks for existing shared helpers** in `shared_helper_dir` before writing anything inline.
6. **Writes the test** covering happy path, sad path, and any applicable edge/corner cases. Each scenario class gets its own `test()` call.
7. **Runs the test** in isolation using the runner command from conventions and confirms it is green.

## Test layer reference

| SUT shape | Layer |
|-----------|-------|
| Pure function, no UI or DB | unit |
| UI component or reactive primitive | component / integration |
| Service or repository function | unit / integration |
| Multi-unit flow | integration |
| CLI command / script entry point | integration |
| Background job, worker, task | unit / integration |
| Full user journey, access control, cross-boundary nav | e2e |
| API handler / endpoint | unit or e2e |

## Hard rules

- No cite, no test. A fabricated or stretched cite is not acceptable.
- One test = one outcome. Two behaviors = two tests.
- Test names start with `should …`.
- No CSS class selectors, tag selectors, or structural combinators in UI tests.
- No test hook attributes (e.g. `data-testid`) added to source unless the element genuinely lacks an accessible name — and that must be flagged separately.
- No assertions on implementation details (internal state, third-party call counts).
- Never pre-build a shared helper speculatively — write inline and extract later via `/test-atom-extract`.

## Shared conventions

Both `test-author` and `test-atom-extract` read from and write to `TEST_CONVENTIONS.md`. Whichever skill runs first on a project bootstraps the file. The other inherits it at no cost.
