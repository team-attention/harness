# Browser Mode (agent-browser)

Use this mode for web applications. agent-browser gives DOM-level access via Playwright — faster and more precise than pixel-based interaction.

## Setup

Verify agent-browser is available:

```bash
command -v agent-browser >/dev/null 2>&1 && echo "OK" || echo "MISSING"
```

If `MISSING`, install: `npm install -g agent-browser && agent-browser install`

If install fails, fall back to computer mode or report error.

Generate session ID:

```bash
openssl rand -hex 2
```

Session ID format: `qa-XXXX`. **Inline session name literally in every command** — shell variables do NOT persist across Bash calls.

## Interaction Patterns

### Navigate
```bash
agent-browser --session qa-XXXX open <url>
```

### Get Element References (for clicking)
```bash
agent-browser --session qa-XXXX snapshot -i
```
Returns accessibility tree with `@ref` numbers. Always snapshot before acting.

### Click, Fill, Type
```bash
agent-browser --session qa-XXXX click @<N>
agent-browser --session qa-XXXX fill @<N> "text"
agent-browser --session qa-XXXX press Enter
```

### Screenshot (evidence)
```bash
agent-browser --session qa-XXXX screenshot .qa-reports/screenshots/name.png
```
After every screenshot, use Read on the file so the user can see it inline.

### JavaScript Evaluation
```bash
agent-browser --session qa-XXXX eval "document.title"
agent-browser --session qa-XXXX eval "JSON.stringify(performance.getEntriesByType('navigation')[0])"
```

### Console Errors
```bash
agent-browser --session qa-XXXX errors
```

### Scroll
```bash
agent-browser --session qa-XXXX scroll down
agent-browser --session qa-XXXX scroll up
```

### Close
```bash
agent-browser --session qa-XXXX close
```

## Core Rules

1. **Snapshot for action, screenshot for evidence** — `snapshot` gives @ref numbers, `screenshot` saves visual proof
2. **Always snapshot before acting** — get @ref numbers first
3. **Re-snapshot after every action** — @ref numbers go stale after page changes
4. **Click by @ref only** — never use CSS selectors or eval DOM queries
5. **Inline everything** — shell vars don't persist across Bash calls

## Diff-Aware Mode (feature branch, no URL)

1. Analyze branch diff: `git diff main...HEAD --name-only`
2. Identify affected pages/routes from changed files
3. Detect running app on common ports (3000, 4000, 8080)
4. Test each affected page with screenshot evidence
5. Report findings scoped to branch changes

## Framework Detection

- `__next` in HTML or `_next/data` -> Next.js
- `csrf-token` meta tag -> Rails
- `wp-content` in URLs -> WordPress
- Client-side routing -> SPA
