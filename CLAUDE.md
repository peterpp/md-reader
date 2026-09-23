# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Markdown Reader 2.x — a Manifest V3 browser extension (Chrome/Edge/Firefox) that renders `.md`/`.mkd`/`.mdx`/`.markdown` URLs (http, https, file) as styled documents. Upstream considers this 2.x codebase unmaintained; this is a fork being patched locally.

## Commands

pnpm is required — `preinstall` aborts under npm/yarn.

```bash
pnpm install
pnpm dev          # webpack watch build into extension/, auto-reloads the extension (webpack-ext-reloader)
pnpm build        # sync manifest version from package.json → prod webpack build → dist/md-reader-<version>.zip
node --test tests/                     # run all tests (no npm script exists)
node --test tests/graphviz.test.mjs    # single test file
pnpm lint <files> # prettier --write; also runs on staged files via husky/lint-staged
```

- Load `extension/` as an unpacked extension to try changes in the browser.
- Tests are plain `node:test` `.mjs` files that `import()` `.ts` sources directly, relying on Node's built-in TypeScript type stripping. Only modules free of `@/` alias imports and DOM/`chrome` globals are testable this way (e.g. `src/core/graphviz.ts`).
- There is no type-check or ESLint step; esbuild-loader strips types without checking. Run `npx tsc --noEmit` manually if needed.
- Version lives in `package.json`; `build:manifest` copies it into `src/manifest.json`.

## Architecture

Four webpack entries (`build/webpack.common.js`), output to `extension/`:

- **`src/main.ts` → `content.js`** — content script injected on markdown URLs (manifest match patterns by file extension), or injected on demand by the background (see `detect.js`). Bails unless enabled and `document.contentType` is plain text/markdown. Takes the browser's raw `<pre>` text, renders it into `<article>`, builds the heading sidebar, floating buttons, and scroll-spy. Raw `<pre>` is kept hidden for the "toggle raw" view (`core/lifecycle.ts`).
- **`src/detect.ts` → `detect.js`** — tiny content script on every other URL; if the server sent `Content-Type: text/markdown`/`text/x-markdown`, it asks the background (`inject` action) to inject `content.css` + `content.js` via `chrome.scripting`. Its `exclude_matches` must mirror the markdown match patterns so the reader is never injected twice.
- **`src/background.ts` → `background.js`** — service worker. Persists settings (`chrome.storage.local`) and pushes changes to the active tab via `actionMap` (setting key → content-script action name in `main.ts`'s `actions`). Also serves `fetch` requests used by the content script's 500 ms auto-refresh polling, and keyboard commands (`core/commands.ts`).
- **`src/popup/` → `popup.html`** — Svelte 3 + SMUI settings UI; writes settings by messaging the background with `{action: 'storage', data: {key, value}}`.

Adding a user setting means touching: `core/data.ts` (type + default), the popup UI, `background.ts` `actionMap`, and a handler in `main.ts` `actions`.

### Two kinds of "plugins"

1. **markdown-it plugins** (`src/core/markdown.ts`): user-toggleable syntax extensions keyed by name in `PLUGINS`; the list of names/defaults is `src/config/md-plugins.ts`. `mdRender` re-creates the MarkdownIt instance whenever options are passed (theme/plugin changes). Graphviz (`plugins/graphviz-block.ts`) is installed _after_ Mermaid and only when Mermaid is enabled, so it can override ` ```mermaid ` fences whose body is DOT syntax; it emits a placeholder `<pre>` with base64 source (`core/graphviz.ts`).
2. **DOM post-processing plugins** (`src/plugins/index.ts`, registered via `usePlugin`): functions receiving `{ event }`; they subscribe to `contentRendered` (emitted after each render with the article element) or `click`. Examples: `graphviz-renderer.ts` (lazily renders placeholders with `@viz-js/viz`), `table-columns.ts` (shrinks overflowing tables after layout), `img-viewer.ts`, `block-copy.ts`.

### Other notes

- Path alias `@/` → `src/` (webpack + tsconfig).
- Theme CSS comes from the external `@md-reader/theme` package; local overrides are in `src/style/*.less`. Page theme (`light`/`dark`/`auto`) is applied via attributes on `<html>` (`shared/setTheme`); Mermaid diagrams must re-render on theme change.
- `src/core/ele.ts` is a small DOM wrapper (`Ele`) used instead of a framework in the content script.
- CSS class names are centralized in `src/config/class-name.ts`; UI strings are in `src/_locales/*/messages.json` and `src/config/i18n/locale.json`.
- Prettier style: no semicolons, single quotes, trailing commas, `arrowParens: avoid`.

## Commits

- When a commit fixes a GitHub issue, append `(fix #<issue number>)` to the commit headline, e.g. `Scale inline code with its surrounding text (fix #168)`. Don't put the issue reference in the commit message body.
