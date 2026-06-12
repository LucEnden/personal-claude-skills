---
name: quality-review
description: >
  Assess a single component's code quality using ISO-25010 criteria adapted for
  component-level evaluation. Infers purpose from code/docs, checks against
  coding standards, and provides structured quality assessment.
  Invoke as /lvde:quality-review <path>.
---

## Purpose

Evaluate code quality across 8 ISO-25010 dimensions: functional suitability, performance efficiency, compatibility, usability, reliability, security, maintainability, portability. Scope: single components (functions, React components, utilities, modules).

Write inline `[REVIEW]` markers directly into source file as findings found (halt-resilient), then produce compact summary report.

## How It Works

### Step 1: Analyze Purpose

Extract and infer component's intended purpose from:
- Docstrings, JSDoc comments, TypeScript types
- Function/class name, parameters, return types
- Test files, usage patterns if available

Output: clear statement of component's intended purpose.

### Step 2: Gather Standards

Collect applicable coding standards:
- Check project `CLAUDE.md` for conventions
- Look for `.eslintrc`, `prettier`, `tsconfig.json`
- If no standards found, ask via Q&A:
  - Code style preferences (naming, comments)?
  - Framework patterns (React hooks, imports)?
  - Performance constraints (bundle, memory)?
  - Accessibility/localization requirements?

### Step 3: Assess ISO-25010 Criteria

Evaluate each criterion. Classify findings:

- **Line-specific finding** — maps to concrete location in file (e.g., missing null check on line 42, magic number on line 89)
- **Component-level finding** — holistic concern, no single line (e.g., no error boundary, low testability, missing input validation throughout)

| Category | What to Check |
|----------|---------------|
| **Functional Suitability** | Completeness, correctness, appropriateness to stated problem |
| **Performance Efficiency** | Time behavior, resource utilization, capacity limits |
| **Compatibility** | Co-existence with dependencies, clear interfaces |
| **Usability** (dev-facing) | Recognizability, learnability, operability, error protection |
| **Reliability** | Maturity, availability, fault tolerance, recoverability |
| **Security** | Confidentiality (input validation), integrity (no XSS/injection), authenticity |
| **Maintainability** | Modularity, reusability, analysability, modifiability, testability |
| **Portability** | Adaptability, replaceability |

### Step 4: Write inline markers into the source file

**Write markers immediately after assessing all criteria — do not wait until report is written.**

#### Line-specific findings

Insert comment on line immediately above the violating line. Write markers **from highest line number to lowest** (bottom-to-top) so earlier insertions don't shift subsequent line numbers.

**Comment syntax:** Derive from file extension and surrounding code context. For mixed-syntax files (e.g. `.tsx` where `//` and `<!-- -->` both valid depending on location), use whichever syntax valid at the specific line. Marker text always: `[REVIEW-<SEVERITY>]: <finding>. Fix: <suggestion>`

Severity: `HIGH`, `MEDIUM`, or `LOW` (uppercase).

Example (TypeScript):
```typescript
// [REVIEW-MEDIUM]: Hardcoded timeout value. Fix: extract as named constant REQUEST_TIMEOUT_MS.
const timeout = 5000;
```

#### Component-level findings

Write comment block at top of file, after any existing file-level comments (copyright headers, module docs) but before imports. If no existing file-level comments, place at very top.

**Block syntax:** Derive from file extension and surrounding code context. Block must open/close using valid multi-line comment delimiters for that file type. Each severity line: `<SEVERITY>: <finding>. Fix: <suggestion>`. Block header always `[REVIEW-COMPONENT]`.

If no component-level findings, omit block entirely.

If no findings at all (line-specific or component-level), skip Step 4.

### Step 5: Output findings

Check for `SPEC.md` at project root.

#### If SPEC.md exists — append §T task rows

For each finding (component-level first, then by line ascending, High before Low within same line), append task row to §T section of `SPEC.md`.

- Read existing §T rows to find highest task number `N`; increment for each new row.
- Row format: `T<N>|.|fix <criterion>: <finding summary> (<file>:<line>)|`
  - Component-level findings use `(<file>:component)` as location
  - Status `.` = not started
  - Description ≤ 60 chars; truncate if needed
  - Deps: cite relevant §V invariants if any apply, otherwise omit
- If §T section absent, add after last existing section.

Example rows appended to §T:
```
T8|.|fix Reliability: no error boundary crashes parent tree (Form.tsx:component)|
T9|.|fix Maintainability: hardcoded timeout value (Form.tsx:42)|
T10|.|fix Usability: unclear param name `d` (Form.tsx:89)|
```

Tell user: "Added N task(s) to §T in SPEC.md."

#### If SPEC.md absent — write report to `.claude/reports/`

Save report to `.claude/reports/` (create directory if absent).

Filename: `<component-name>-quality-<YYYY-MM-DD>.md`

Example: `.claude/reports/TransactionForm-quality-2026-05-25.md`

Report structure:

```markdown
# Quality Review: ComponentName

- **Path:** `src/path/to/component.tsx`
- **Purpose:** <inferred purpose>
- **Date:** <ISO 8601 date and time>
- **Analyst:** <model name>

## Findings

| Line(s) | Severity | Criterion | Finding | Suggestion |
|---------|----------|-----------|---------|------------|
| component | High | Reliability | No error boundary — uncaught throws crash parent tree | Wrap in ErrorBoundary |
| 42 | Medium | Maintainability | Hardcoded timeout value | Extract as REQUEST_TIMEOUT_MS |
| 89 | Low | Usability | Unclear parameter name `d` | Rename to `durationMs` |

## Risk Assessment

- **Overall Risk:** Low / Medium / High
- **Primary Risk:** <main concern>
- **Recommended Actions:** <what to do next>

> **Remove `[REVIEW]` markers** from the source file after addressing each item. Do not commit markers.
```

Sort Findings table: component-level rows first, then by line number ascending, then Severity descending within same line.

Tell user saved path after writing.

## Usage

```bash
# Basic review
/lvde:quality-review src/components/TransactionForm.tsx

# With standards context
/lvde:quality-review src/lib/validators.ts --standards CLAUDE.md

# Focus on specific areas
/lvde:quality-review src/hooks/useData.ts --focus maintainability,testability
```

## Parameters

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `component_path` | string | Yes | Path to component file |
| `standards_context` | string | No | Path to standards doc (default: CLAUDE.md, .eslintrc, tsconfig) |
| `focus_areas` | array | No | ISO criteria to emphasize (default: all) |

## Key Principles

- **Write markers before output** — if halted, per-file markers persist
- **SPEC.md first** — append findings as §T task rows when SPEC.md exists at project root; fall back to `.claude/reports/` only when absent
- **Inference over documentation** — purpose inferred from code itself
- **Standards-aware** — respects project conventions from CLAUDE.md and config files
- **Actionable** — recommendations include "why" and "how to fix", not just criticism
- **Component-scoped** — ISO-25010 adapted for single modules, not system-level
- **Developer-centric** — evaluates usability for developers, not end users