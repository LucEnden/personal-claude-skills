---
name: quality-review
description: >
  Assess a single component's code quality using ISO-25010 criteria adapted for
  component-level evaluation. Infers purpose from code/docs, checks against
  coding standards, and provides structured quality assessment.
  Invoke as /lvde:quality-review <path>.
---

## Purpose

Evaluate code quality across 8 ISO-25010 dimensions: functional suitability, performance efficiency, compatibility, usability, reliability, security, maintainability, and portability. Scope limited to single components (functions, React components, utilities, modules).

Write inline `[REVIEW]` markers directly into the source file as findings are identified (halt-resilient), then produce a compact summary report.

## How It Works

### Step 1: Analyze Purpose

Extract and infer the component's intended purpose from:
- Docstrings, JSDoc comments, TypeScript types
- Function/class name, parameters, return types
- Test files, usage patterns if available

Output: Clear statement of component's intended purpose.

### Step 2: Gather Standards

Collect applicable coding standards and preferences:
- Check project `CLAUDE.md` for conventions
- Look for `.eslintrc`, `prettier`, `tsconfig.json`
- If no standards found, ask via Q&A:
  - Code style preferences (naming, comments)?
  - Framework patterns (React hooks, imports)?
  - Performance constraints (bundle, memory)?
  - Accessibility/localization requirements?

### Step 3: Assess ISO-25010 Criteria

Evaluate each criterion. As you identify findings, classify them:

- **Line-specific finding** — maps to a concrete location in the file (e.g., missing null check on line 42, magic number on line 89)
- **Component-level finding** — holistic concern with no single line to point at (e.g., no error boundary, low testability, missing input validation throughout)

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

**Write markers immediately after assessing all criteria — do not wait until the report is written.**

#### Line-specific findings

Insert a comment on the line immediately above the violating line. Write markers **from the highest line number to the lowest** (bottom-to-top) so earlier insertions do not shift subsequent line numbers.

**Comment syntax:** Derive the correct comment syntax from the file extension and surrounding code context. For mixed-syntax files (e.g. `.tsx` where `//` and `<!-- -->` are both valid depending on location), use whichever syntax is valid at the specific line being annotated. The marker text is always: `[REVIEW-<SEVERITY>]: <finding>. Fix: <suggestion>`

Severity in the marker is `HIGH`, `MEDIUM`, or `LOW` (uppercase).

Example (TypeScript):
```typescript
// [REVIEW-MEDIUM]: Hardcoded timeout value. Fix: extract as named constant REQUEST_TIMEOUT_MS.
const timeout = 5000;
```

#### Component-level findings

Write a comment block at the top of the file, after any existing file-level comments (copyright headers, module docs) but before imports. If no existing file-level comments, place at the very top.

**Block syntax:** Derive the correct block-comment syntax from the file extension and surrounding code context. The block must open and close using whichever multi-line comment delimiters are valid for that file type. Each severity line inside the block follows the format: `<SEVERITY>: <finding>. Fix: <suggestion>`. The block header is always `[REVIEW-COMPONENT]`.

If there are no component-level findings, omit the block entirely.

If there are no findings at all (line-specific or component-level), skip Step 4 entirely.

### Step 5: Write summary report

Save the report to `.claude/reports/` (create directory if absent).

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

Sort Findings table: component-level rows first, then by line number ascending, then by Severity descending within the same line.

Tell the user the saved path after writing.

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

- **Write markers before report** — if halted, per-file markers persist
- **Inference over documentation** — purpose inferred from code itself
- **Standards-aware** — respects project conventions from CLAUDE.md and config files
- **Actionable** — recommendations include "why" and "how to fix", not just criticism
- **Component-scoped** — ISO-25010 adapted for single modules, not system-level
- **Developer-centric** — evaluates usability for developers, not end users
