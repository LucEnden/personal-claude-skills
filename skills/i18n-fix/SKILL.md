---
name: i18n-fix
description: >
  Detects hardcoded inline text in JSX/TSX that bypasses the project's i18n
  system, reports all violations, then (with confirmation) rewrites each site
  to use the project's actual translation mechanism. Framework-agnostic: works
  with react-i18next, react-intl, next-intl, lingui, custom hooks, or any
  other i18n approach. Invoke as /lvde:i18n-fix [file ...].
---

# i18n Fix

Detect + fix hardcoded inline text in JSX/TSX that should use project translation system.

## When Invoked

User calls `/lvde:i18n-fix [file1 file2 ...]` or asks to find/fix i18n violations, hardcoded strings, inline text issues.

---

## Step-by-Step Execution

### 1. Resolve target files

- Args given: use those file paths (relative from project root).
- No args: glob `**/*.{jsx,tsx}` from project root, excluding `node_modules`, `dist`, `build`, `.next`, `.cache`.
- No matching files: report and stop.

### 2. Detect the project's i18n approach

Search codebase (NOT just target files) to determine how translations are handled. Check signals in order:

| Signal (grep for) | Approach |
|---|---|
| `useTranslation` / `import.*react-i18next` | react-i18next — `const { t } = useTranslation()`, wrap text in `{t('key')}` |
| `useIntl` / `FormattedMessage` / `import.*react-intl` | react-intl — `const intl = useIntl()`, wrap text in `{intl.formatMessage({ id: 'key' })}` or `<FormattedMessage id="key" />` |
| `useTranslations` / `import.*next-intl` | next-intl — `const t = useTranslations()`, wrap text in `{t('key')}` |
| `useLingui` / `import.*@lingui` | Lingui — `const { _ } = useLingui()`, wrap text in `{_(msg\`key\`)}` |
| `import.*i18n` or `i18n.t(` | i18n-js / generic — wrap text in `{i18n.t('key')}` |
| Custom hook (e.g. `useLocale`, `useStrings`) | Document hook name + call signature from existing usage |

Steps:
1. Grep each signal across entire project.
2. Check 2–3 real call-sites in existing files — exact call signature (hook import path, function name, key structure).
3. Check translation resource file (e.g. `en.json`, `messages/en.ts`, `locales/en.json`) for key naming conventions.
4. Multiple signals: prefer one with most existing usages.
5. NO i18n signals: stop, inform user no i18n system detected. Don't invent one. Ask user to describe translation approach before proceeding.

Store:
- `I18N_APPROACH`: short label (e.g. "react-i18next")
- `HOOK_IMPORT`: import statement to add when file lacks it
- `HOOK_SETUP`: local variable setup line (e.g. `const { t } = useTranslation()`)
- `WRAP_TEXT(key)`: how to replace bare string — e.g. `{t('key')}`
- `WRAP_ATTR(key)`: how to replace JSX attribute value — e.g. `{t('key')}`
- `KEY_PREFIX_CONVENTION`: if translation keys follow namespace or file-based prefix (infer from existing keys in resource file)

### 3. Scan each target file for violations

Per target file, read full contents and identify every hardcoded text site meeting ALL criteria:

**Must flag:**
- JSX text nodes (content between tags) with meaningful text
- JSX attribute *string literal* values with meaningful text (e.g. `placeholder="Enter name"`, `aria-label="Close dialog"`, `title="Submit"`)
- JSX attribute *template literal* values with meaningful text segments

**Meaningful text** — string must:
- Not be empty or whitespace-only
- Be longer than 1 character
- Not be purely numeric (e.g. `42`, `3.14`)
- Not be purely symbols/punctuation (e.g. `...`, `->`, `|`)
- NOT match technical patterns:
  - kebab-case / snake_case / camelCase single tokens (no spaces): `my-class`, `btn_primary`, `handleClick`
  - hex colors: `#fff`, `#1a2b3c`
  - CSS values: `rgb(...)`, `rgba(...)`, `0px`, `100%`, `1.5rem`
  - URLs / paths: starting with `http`, `https`, `/`, `./`, `../`
  - data URIs: starting with `data:`
  - ARIA relation IDs (attribute names `aria-labelledby`, `aria-describedby`, `htmlFor`, `id`)
  - Test IDs (`data-testid`, `data-cy`)
  - Framework internals: `className`, `key`, `ref`, `name` (form field), `type`, `role`, `style`, `tabIndex`
- NOT already call a translation function (e.g. already wrapped in `t(...)`, `intl.formatMessage(...)`, etc.)

Per violation record:
- File path
- Line number(s)
- Raw string value
- Text node or attribute (and which attribute name)
- Suggested translation key (derive from context: component name + descriptive slug, e.g. `LoginForm.submitButton`, `NavBar.homeLink`)

### 4. Generate violation report

Output in this exact structure (raw Markdown, not code-fenced):

---

# i18n Violation Report

- **i18n approach detected:** `<I18N_APPROACH>`
- **Translation call pattern:** `<example call>`
- **Files scanned:** N
- **Violations found:** N

## Violations

### `<file path>`

| Line | Type | Raw Text | Suggested Key |
|------|------|----------|---------------|
| 42 | JSX text | `Hello World` | `HomePage.heroTitle` |
| 67 | attr: placeholder | `Enter your email` | `LoginForm.emailPlaceholder` |

*(repeat per file; omit file section if no violations)*

## Summary

| File | Count |
|------|-------|
| `src/components/Foo.tsx` | 3 |

**Total:** N violations across M files.

---

Zero violations: output "No i18n violations found." and stop.

### 5. Ask for confirmation

After report, ask:

> Found N violations. Proceed with automatic fixes? (yes / no / select files)
>
> - **yes** — fix all violations in all files
> - **no** — stop here
> - **select** — you list which files to fix

Wait for user response before proceeding.

### 6. Fix violations (only if confirmed)

Per confirmed file:

1. Read current file contents.
2. Per violation in that file (process bottom-up to preserve line numbers):
   a. Replace raw hardcoded text with translation call.
   b. JSX text nodes:  
      `Hello World` → `{t('HomePage.heroTitle')}`  
      (wrap in expression container if not already inside one)
   c. JSX attribute string literals:  
      `placeholder="Enter your email"` → `placeholder={t('LoginForm.emailPlaceholder')}`
   d. Template literal attribute values with hardcoded segments: replace only hardcoded quasis, keep dynamic expressions intact.
3. File missing translation hook/function import: add at top (after existing imports, not mixed in middle).
4. React component needing hook call (e.g. `const { t } = useTranslation()`): add as first statement inside component function body, only if not already present.
5. Write updated file.
6. Add new translation keys to resource file (e.g. `en.json`):
   - Use suggested key from violation report.
   - Set value to original hardcoded string (app renders identically after fix).
   - Preserve existing JSON structure and formatting.
   - Key already exists: skip.
   - No resource file: create `locales/en.json` at project root.

After all files written, output fix summary:

```
Fixed N violations across M files.
Updated translation keys: locales/en.json (+N keys)
```

---

## Rules

- Never guess/invent i18n approach. Derive entirely from project.
- Never remove or alter existing translation calls.
- Never flag text already wrapped in translation call.
- Process violations bottom-up per file to keep line numbers stable.
- Adding hook call to component: match style of existing hook calls in that file (spacing, destructuring style, etc.).
- Don't restructure or reformat code beyond minimum needed to insert translation call.
- Class component (not function): use appropriate non-hook API for detected i18n library (e.g. `withTranslation` HOC for react-i18next) and note in fix summary.
- Keep suggested keys consistent: `ComponentName.descriptiveSlug` in camelCase, no spaces, no special characters except dots and dashes.

## Usage

```bash
# Scan and fix entire project
/lvde:i18n-fix

# Scan and fix specific files
/lvde:i18n-fix src/components/LoginForm.tsx src/pages/Home.tsx

# Scan a directory (pass all matching files)
/lvde:i18n-fix src/components/
```