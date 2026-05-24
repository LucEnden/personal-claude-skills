# Software Quality Review Skill

A specialized skill for assessing the quality of a single component in isolation using ISO-25010 criteria adapted for component-level evaluation.

## Purpose

Evaluate code quality across multiple dimensions:
- **Functional correctness**: Does it do what it's supposed to?
- **Performance**: Is it efficient?
- **Reliability**: Does it handle edge cases?
- **Security**: Are there vulnerabilities?
- **Maintainability**: Is it easy to understand and modify?
- **Compatibility**: Does it integrate well?
- **Usability** (for developers): Is it easy to use?
- **Portability**: Can it be reused elsewhere?

## Triggers

- `/quality [path]`
- `evaluate component [path]`
- `component quality review [path]`

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `component_path` | string | Yes | Path to the component file (TypeScript, React, utilities, etc.) |
| `standards_context` | string | No | Optional path to project standards file (e.g., `CLAUDE.md`, `coding-standards.md`) |
| `focus_areas` | array | No | ISO-25010 categories to emphasize (default: all) |

## ISO-25010 Assessment Criteria

### Functional Suitability
- **Completeness**: Does it cover all documented requirements?
- **Correctness**: Does it produce correct outputs for all documented inputs?
- **Appropriateness**: Does it solve the stated problem effectively?

### Performance Efficiency
- **Time behavior**: Is performance acceptable for typical use?
- **Resource utilization**: Does it avoid unnecessary allocations/renders?
- **Capacity**: Any hard limits properly handled?

### Compatibility
- **Co-existence**: Works with dependencies without conflicts?
- **Interoperability**: Clear interfaces, proper prop/argument contracts?

### Usability (for developers)
- **Recognizability**: Obvious what the component does from name/docs?
- **Learnability**: Easy to understand intent from code structure?
- **Operability**: Clear usage patterns, minimal cognitive load?
- **Error protection**: Validates inputs, handles edge cases?

### Reliability
- **Maturity**: Handles normal conditions reliably?
- **Availability**: No hidden race conditions or state issues?
- **Fault tolerance**: Gracefully handles unexpected inputs?
- **Recoverability**: Error states recoverable/testable?

### Security
- **Confidentiality**: No unvalidated user input usage?
- **Integrity**: No XSS/injection vulnerabilities?
- **Authenticity**: Proper access control if applicable?

### Maintainability
- **Modularity**: Cohesive, single responsibility?
- **Reusability**: Useful in other contexts?
- **Analysability**: Clear control flow, easy to debug?
- **Modifiability**: Easy to extend without breaking changes?
- **Testability**: Mockable, has clear dependencies?

### Portability
- **Adaptability**: No platform-specific hardcoding?
- **Replaceability**: Could be swapped for alternative implementation?

## Workflow

### 1. Analyze Purpose
Extract and infer the component's intended purpose:
- Extract docstrings, JSDoc comments, TypeScript types
- Infer from function/class name, parameters, return types
- Review test files or usage patterns if available
- Output: Clear statement of component's intended purpose

### 2. Gather Standards
Collect applicable coding standards and preferences:
- Check project `CLAUDE.md` for coding conventions
- Look for `.eslintrc`, `prettier` config, `tsconfig.json`
- If no standards found, ask engineer via Q&A:
  - Code style preferences (naming conventions, comment style)?
  - Framework patterns (React hooks vs classes)?
  - Performance constraints (bundle size, memory)?
  - Accessibility/localization requirements?

### 3. Assess ISO-25010 Criteria
Evaluate component against each criterion:
- Check for compliance with gathered standards
- Identify violations and gaps
- Note areas of strength
- Consider context (is this a utility, React component, validator, etc.?)

### 4. Generate Report
Create structured quality assessment with:
- Component name and inferred purpose
- ✓ **Strengths** (ISO-25010 criteria met well)
- ⚠ **Areas for Improvement** (current gap → recommended fix)
- ❌ **Issues** (violations requiring action)
- Standards alignment checklist
- Overall risk level (Low/Medium/High) with primary risk identified

## Usage Examples

```bash
# Basic component review
/quality src/components/TransactionForm.tsx

# With explicit standards context
/quality src/lib/validators.ts --standards CLAUDE.md

# Focus on specific ISO-25010 areas
/quality src/hooks/useTransactions.ts --focus maintainability,testability
```

## Output Format

```
## [Component Name]
**Path:** `src/components/Example.tsx`
**Purpose:** [Inferred intention from code analysis]

### Quality Assessment

✓ **Strengths** (ISO-25010 criteria met):
- **Modularity**: Clean separation of concerns, single responsibility principle followed
- **Testability**: Pure functions with clear inputs/outputs, easy to mock

⚠ **Areas for Improvement**:
- **Error protection**: No input validation on `data` prop → Add Zod schema validation
- **Learnability**: Complex logic without comments → Add JSDoc explaining algorithm

❌ **Issues** (violations of standards):
- **Security**: User input rendered without sanitization → Use DOMPurify or sanitize strings

### Standards Alignment
- Import ordering: ✓ Compliant
- Naming conventions: ✓ Compliant
- Comment style: ✓ Compliant
- TypeScript strictness: ⚠ Missing return type annotation

### Risk Assessment
- **Overall Risk Level:** Medium
- **Primary Risk:** Potential XSS vulnerability in user-controlled content rendering
- **Recommended Actions:** Implement input sanitization, add prop validation schema
```

## Notes

- Adapted from ISO/IEC 25010:2023 for component-level evaluation (not system-level)
- Focuses on "usability for developers" rather than end-user usability
- Infers purpose from code itself—no external documentation required
- Provides actionable recommendations, not just criticism
- Respects project standards from `CLAUDE.md` and config files
