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

Detect and fix hardcoded inline text in JSX/TSX that should use the project's
translation system.

## When Invoked

User calls `/lvde:i18n-fix [file1 file2 ...]` or asks to find/fix i18n
violations, hardcoded strings, or inline text issues.

---

## Step-by-Step Execution

### 1. Resolve target files

- If args given: use those file paths (relative from project root).
- If no args: glob `**/*.{jsx,tsx}` from project root, excluding
  `node_modules`, `dist`, `build`, `.next`, `.cache`.
- If no matching files found: report and stop.

### 2. Detect the project's i18n approach

Search the codebase (NOT just the target files) to determine how translations
are handled. Look for these signals in order:

| Signal (grep for) | Approach |
|---|---|
| `useTranslation` / `import.*react-i18next` | react-i18next — `const { t } = useTranslation()`, wrap text in `{t('key')}` |
| `useIntl` / `FormattedMessage` / `import.*react-intl` | react-intl — `const intl = useIntl()`, wrap text in `{intl.formatMessage({ id: 'key' })}` or `<FormattedMessage id="key" />` |
| `useTranslations` / `import.*next-intl` | next-intl — `const t = useTranslations()`, wrap text in `{t('key')}` |
| `useLingui` / `import.*@lingui` | Lingui — `const { _ } = useLingui()`, wrap text in `{_(msg\`key\`)}` |
| `import.*i18n` or `i18n.t(` | i18n-js / generic — wrap text in `{i18n.t('key')}` |
| Custom hook (e.g. `useLocale`, `useStrings`) | Document hook name and call signature from existing usage |

Steps:
1. Grep for each signal across the entire project.
2. Look at 2–3 real call-sites in existing files to understand the exact
   call signature (hook import path, function name, how keys are structured).
3. Check if there is a translation resource file (e.g. `en.json`,
   `messages/en.ts`, `locales/en.json`) to understand key naming conventions.
4. If multiple signals are found, prefer the one with the most existing usages.
5. If NO i18n signals found: stop and inform the user that no i18n system was
   detected. Do not invent one. Ask the user to describe their translation
   approach before proceeding.

Store:
- `I18N_APPROACH`: short label (e.g. "react-i18next")
- `HOOK_IMPORT`: import statement to add when a file lacks it
- `HOOK_SETUP`: the local variable setup line (e.g. `const { t } = useTranslation()`)
- `WRAP_TEXT(key)`: how to replace a bare string — e.g. `{t('key')}`
- `WRAP_ATTR(key)`: how to replace a JSX attribute value — e.g. `{t('key')}`
- `KEY_PREFIX_CONVENTION`: if translation keys follow a namespace or file-based
  prefix convention (infer from existing keys in the resource file)

### 3. Scan each target file for violations

For each target file, read the full contents and identify every hardcoded text
site that meets ALL of the following criteria:

**Must flag:**
- JSX text nodes (content between tags) that contain meaningful text
- JSX attribute *string literal* values that contain meaningful text (e.g.
  `placeholder="Enter name"`, `aria-label="Close dialog"`, `title="Submit"`)
- JSX attribute *template literal* values containing meaningful text segments

**Meaningful text** means the string:
- Is not empty or whitespace-only
- Is longer than 1 character
- Is not purely numeric (e.g. `42`, `3.14`)
- Is not purely symbols/punctuation (e.g. `...`, `->`, `|`)
- Does NOT match technical patterns:
  - kebab-case / snake_case / camelCase single tokens (no spaces): `my-class`, `btn_primary`, `handleClick`
  - hex colors: `#fff`, `#1a2b3c`
  - CSS values: `rgb(...)`, `rgba(...)`, `0px`, `100%`, `1.5rem`
  - URLs / paths: starting with `http`, `https`, `/`, `./`, `../`
  - data URIs: starting with `data:`
  - ARIA relation IDs (attribute names `aria-labelledby`, `aria-describedby`, `htmlFor`, `id`)
  - Test IDs (`data-testid`, `data-cy`)
  - Framework internals: `className`, `key`, `ref`, `name` (form field), `type`, `role`, `style`, `tabIndex`
- Does NOT already call a translation function (e.g. already wrapped in `t(...)`, `intl.formatMessage(...)`, etc.)

For each violation record:
- File path
- Line number(s)
- The raw string value
- Whether it's a text node or attribute (and which attribute name)
- A suggested translation key (derive from context: component name + a
  descriptive slug, e.g. `LoginForm.submitButton`, `NavBar.homeLink`)

### 4. Generate violation report

Output the report in this exact structure (raw Markdown, not code-fenced):

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

If there are zero violations: output "No i18n violations found." and stop.

### 5. Ask for confirmation

After the report, ask:

> Found N violations. Proceed with automatic fixes? (yes / no / select files)
>
> - **yes** — fix all violations in all files
> - **no** — stop here
> - **select** — you list which files to fix

Wait for user response before proceeding.

### 6. Fix violations (only if confirmed)

For each confirmed file:

1. Read the current file contents.
2. For each violation in that file (process bottom-up to preserve line numbers):
   a. Replace the raw hardcoded text with the translation call.
   b. For JSX text nodes:  
      `Hello World` → `{t('HomePage.heroTitle')}`  
      (wrap in expression container if not already inside one)
   c. For JSX attribute string literals:  
      `placeholder="Enter your email"` → `placeholder={t('LoginForm.emailPlaceholder')}`
   d. For template literal attribute values with hardcoded segments: replace
      only the hardcoded quasis, keeping dynamic expressions intact.
3. If the file does not already import the translation hook/function, add the
   import at the top (after existing imports, not mixed in the middle).
4. If the file is a React component and needs a hook call (e.g. `const { t } = useTranslation()`),
   add it as the first statement inside the component function body, but only
   if it's not already present.
5. Write the updated file.
6. Add the new translation keys to the resource file (e.g. `en.json`):
   - Use the suggested key from the violation report.
   - Set the value to the original hardcoded string (so the app still renders
     identically after the fix).
   - Preserve the existing JSON structure and formatting.
   - If a key already exists with that name, skip adding it.
   - If no resource file exists, create `locales/en.json` at the project root.

After all files are written, output a fix summary:

```
Fixed N violations across M files.
Updated translation keys: locales/en.json (+N keys)
```

---

## Rules

- Never guess or invent an i18n approach. Derive it entirely from the project.
- Never remove or alter existing translation calls.
- Never flag text that is already wrapped in a translation call.
- Process violations bottom-up within each file to keep line numbers stable.
- When adding a hook call to a component, match the style of existing hook
  calls in that file (spacing, destructuring style, etc.).
- Do not restructure or reformat code beyond the minimum needed to insert the
  translation call.
- If a component is a class component (not a function), use the appropriate
  non-hook API for the detected i18n library (e.g. `withTranslation` HOC for
  react-i18next) and note this in the fix summary.
- Keep suggested keys consistent: `ComponentName.descriptiveSlug` in
  camelCase, no spaces, no special characters except dots and dashes.

## Usage

```bash
# Scan and fix entire project
/lvde:i18n-fix

# Scan and fix specific files
/lvde:i18n-fix src/components/LoginForm.tsx src/pages/Home.tsx

# Scan a directory (pass all matching files)
/lvde:i18n-fix src/components/
```
