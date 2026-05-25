---
name: test-atom-extract
description: Scan the test tree for duplicated setup-then-assert patterns and propose an atom extraction into a shared test helper directory. Read-only proposal first; engineer approves before any file is written.
triggers:
  - "/test-atom-extract"
  - "extract test atom"
  - "find duplicate tests"
---

# test-atom-extract

Finds test code duplication and proposes extracting it into a shared atom helper directory.

> **Adapt:** This skill uses `tests/_atoms/` as the default atom directory, `tests/_fixtures/` for shared fixtures, and `tests/_lib/flows/` for navigation helpers. Replace these paths with your project's equivalent directories before use.

## When to invoke

- Periodically as the test tree grows.
- After authoring a test whose body felt familiar — there's a good chance the same shape exists elsewhere.
- Never preemptively (no atom is written before a second real occurrence).

## Process

1. **Scan** all test files for repeated patterns. Look for:
   - Identical sequences of 3+ statements that include at least one assertion.
   - Identical fixture setup followed by identical assertions on the same role, label, or selector.
2. **Cluster** matches by behavior, not by SUT. Group "required-field validation" hits even when they're on different forms.
3. **Filter**: keep clusters with **≥ 2 occurrences across different test files**. Discard same-file duplication (that's a parameterized test candidate, not an atom).
4. **Propose** for each surviving cluster:
   - Atom name (behavior-as-verb, e.g. `assertRequiredField`).
   - Target file within the atom directory (`forms`, `tables`, `dialogs`, or a new file if no existing one fits).
   - Signature (inputs are selectors and i18n keys or string constants; never hardcoded display strings).
   - Diff for each test site showing the rewrite.
5. **Halt and ask** via `AskUserQuestion`: accept / reject / rename / merge with existing atom. Only after explicit approval does the skill write.

## Hard rules

- One atom = one assertion concept. Split if a single helper needs "and" in its description.
- Atom signature accepts selectors and text keys or constants as parameters. It does not hardcode either.
- Atom does not introduce new layers of indirection (no class hierarchies, no `BaseAtom` superclasses).
- If a candidate cluster's body diverges by more than 1-2 parameters, do not extract. The "near-identical" cases stay inline until they actually converge.

## Out of scope

- Page Object Models. If multiple e2e tests share a navigation sequence, that's a test fixture (in `tests/_fixtures/` or equivalent) or a named flow helper (in `tests/_lib/flows/` or equivalent), not an atom.
- Authoring new tests. This skill only refactors existing ones. New tests are handled by `/test-author`.
