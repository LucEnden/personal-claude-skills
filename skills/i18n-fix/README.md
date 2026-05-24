# i18n-fix

Detect and fix hardcoded inline text in JSX/TSX that bypasses the project's
translation system.

## What it does

1. **Detects your i18n approach** — inspects the codebase for react-i18next,
   react-intl, next-intl, lingui, i18n-js, or any custom translation hook.
   Never assumes; always derives from actual usage.
2. **Scans for violations** — finds JSX text nodes and attribute string
   literals that contain meaningful human-readable text not yet wrapped in a
   translation call. Skips CSS classes, IDs, URLs, hex colors, and other
   non-translatable technical strings.
3. **Reports violations** — structured Markdown table per file with line
   numbers, raw text, and suggested translation keys.
4. **Fixes (with confirmation)** — rewrites each violation using the project's
   actual translation call pattern, adds missing imports/hook calls, and
   appends the new keys (with original text as default value) to the
   translation resource file.

## Usage

```bash
# Scan and fix entire project
/lvde:i18n-fix

# Specific files
/lvde:i18n-fix src/components/LoginForm.tsx src/pages/Home.tsx
```

## Notes

- Works for any i18n library — does not hardcode `t()` or any other API.
- If no i18n system is detected, stops and asks before doing anything.
- Fixes are bottom-up within each file to keep line numbers stable.
- Class components get the appropriate non-hook API (e.g. `withTranslation`).
