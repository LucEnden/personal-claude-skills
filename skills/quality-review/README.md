# quality-review

Assesses a single component's code quality using ISO-25010 criteria. Infers purpose from the code itself, writes inline `[REVIEW]` markers into the source file as findings are identified (halt-resilient), then produces a structured Markdown report saved to `.claude/reports/`.

## Trigger

```
/lvde:quality-review <path>
```

Also triggers on: "evaluate component", "component quality review".

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `component_path` | string | Yes | Path to the component file |
| `standards_context` | string | No | Path to standards doc (default: CLAUDE.md, .eslintrc, tsconfig) |
| `focus_areas` | array | No | ISO-25010 criteria to emphasize (default: all) |

## What It Does

1. Infers component purpose from docstrings, types, name, parameters, and test files
2. Gathers coding standards from `CLAUDE.md`, `.eslintrc`, `tsconfig.json` — asks if none found
3. Evaluates all 8 ISO-25010 criteria, classifying each finding as line-specific or component-level
4. Writes `[REVIEW]` markers into the source file
5. Writes a Markdown report to `.claude/reports/<component-name>-quality-<YYYY-MM-DD>.md`

## ISO-25010 Criteria

| Criterion | What Is Checked |
|-----------|----------------|
| Functional Suitability | Completeness, correctness, appropriateness |
| Performance Efficiency | Time behavior, resource utilization, capacity |
| Compatibility | Co-existence with dependencies, clear interfaces |
| Usability (dev-facing) | Recognizability, learnability, operability, error protection |
| Reliability | Maturity, availability, fault tolerance, recoverability |
| Security | Input validation, no XSS/injection, access control |
| Maintainability | Modularity, reusability, analysability, modifiability, testability |
| Portability | Adaptability, replaceability |

## Inline Markers

Two types of findings, two marker styles:

**Line-specific** — inserted on the line immediately above the violating line, bottom-to-top order.
Format: `[REVIEW-<SEVERITY>]: <finding>. Fix: <suggestion>`

**Component-level** — inserted as a block at the top of the file (after any file-level comments, before imports). Used for holistic concerns with no single line to point at.
Format: block comment with header `[REVIEW-COMPONENT]`, one line per severity entry.

Comment syntax (both types) is derived from the file extension and surrounding code context. For mixed-syntax files (e.g. `.tsx`), whichever syntax is valid at that specific location is used.

Markers are written before the report — if halted mid-run, markers already written persist.

> Remove `[REVIEW]` markers from the source file after addressing each item. Do not commit markers.

## Report Structure

Saved to `.claude/reports/`, filename: `<component-name>-quality-<YYYY-MM-DD>.md`

| Section | Content |
|---------|---------|
| Header | Path, inferred purpose, date, analyst/model |
| Findings | Table: Line(s) / Severity / Criterion / Finding / Suggestion |
| Risk Assessment | Overall risk level, primary risk, recommended actions |

Findings table order: component-level rows first, then ascending line number, then severity descending within same line.

## Key Principles

- **Write markers before report** — if halted, per-file markers persist
- **Inference over documentation** — purpose inferred from code itself
- **Standards-aware** — respects project conventions from CLAUDE.md and config files
- **Actionable** — recommendations include why and how to fix, not just criticism
- **Component-scoped** — ISO-25010 adapted for single modules, not system-level
- **Developer-centric** — evaluates usability for developers, not end users
