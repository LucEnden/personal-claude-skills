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

## How It Works

### Step 1: Analyze Purpose
Extract and infer the component's intended purpose from:
- Docstrings, JSDoc comments, TypeScript types
- Function/class name, parameters, return types
- Test files, usage patterns if available
- Output: Clear statement of component's intended purpose

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
Evaluate each criterion:

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

### Step 4: Generate Report

```
## [Component Name]
**Path:** `src/path/to/component.tsx`
**Purpose:** [Inferred intention]

### Quality Assessment

✓ **Strengths** (ISO-25010 criteria met):
- **[Criterion]**: Why it excels

⚠ **Areas for Improvement**:
- **[Criterion]**: Current gap → Recommended fix

❌ **Issues** (standard violations):
- **[Issue]**: Impact → Required change

### Standards Alignment
- [Standard]: ✓ Compliant / ⚠ Partial / ❌ Non-compliant

### Risk Assessment
- **Overall Risk Level:** Low / Medium / High
- **Primary Risk:** [Main concern]
- **Recommended Actions:** [What to do next]
```

### Step 5: Save Report

Save the report as a Markdown file in `.claude/reports/` (create directory if absent).

Filename: `<component-name>-quality-<YYYY-MM-DD>.md`

Example: `.claude/reports/TransactionForm-quality-2026-05-25.md`

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

- **Inference over documentation**: Purpose is inferred from code itself—no external docs required
- **Standards-aware**: Respects project conventions from CLAUDE.md and config files
- **Actionable**: Recommendations include "why" and "how to fix", not just criticism
- **Component-scoped**: ISO-25010 adapted for single modules, not system-level
- **Developer-centric**: Evaluates usability for developers, not end users
