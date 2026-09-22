# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

See @AGENTS.md for project-specific rules (what to avoid, what's outdated, key facts). Read it first — this file adds commands and architecture context.

## Project

Adobe AEM Edge Delivery Services (Franklin) site, "eliza", based on `@adobe/aem-boilerplate`. Plain JS/CSS, no framework, no bundler. Content is authored in a document (Google Docs/Word) and served by the AEM backend as HTML; this repo only supplies decoration scripts and styles.

- Preview: https://main--eliza--fastcmsdomain.aem.page/
- Live: https://main--eliza--fastcmsdomain.aem.live/

## Commands

```sh
npm i                  # install devDependencies (no runtime deps)
npx -y @adobe/aem-cli up   # local dev server at localhost:3000, proxies real content
npm run lint            # eslint (blocks/, scripts/) + stylelint (blocks/**/*.css, styles/*.css)
npm run lint:js         # eslint only
npm run lint:css        # stylelint only
npm run lint:fix        # autofix both
```

There is no test suite configured in this repo (no `test` script, no test framework installed). CI (`.github/workflows/main.yaml`) runs `npm ci && npm run lint` on every push — that's the full gate.

## Architecture

- **`scripts/scripts.js`** is the entry point, run at the bottom of the file (`loadPage()`). It defines the three-phase load lifecycle used by `aem.js`:
  - `loadEager` — sets `<html lang>`, decorates the main element, renders the first section (blocking on the first image for LCP), then conditionally kicks off font loading.
  - `loadLazy` — loads header/footer fragments, renders remaining sections, handles URL-hash scroll, then loads `styles/lazy-styles.css` and fonts.
  - `loadDelayed` — imports `consent-check.js`; put anything non-critical here.
  - `decorateMain` runs `buildAutoBlocks` before section/block decoration — it turns bare links into synthetic blocks *before* any block's own `decorate()` sees the DOM. Currently handles `*/fragments/*` links (via `blocks/fragment/fragment.js`) and `/widgets/*` links (synthesized into a `widget` block, see below).
  - A Trusted Types policy is installed at the top of the file to sanitize `srcdoc` and `document.write`/`createContextualFragment` sinks — relevant if a block ever injects HTML.
- **`scripts/aem.js`** is the vendored Helix framework (block/section decoration, lazy image loading, RUM, etc.) — see AGENTS.md, never edit it.
- **`blocks/<name>/`** — one folder per block, each with `<name>.js` (default export `decorate(block)`, mutates the block element in place) and `<name>.css` (selectors scoped to `.<name>`, per AGENTS.md). Blocks receive markup already provided by the backend; author-omitted cells are common, so check `.children`/`.length` before indexing.
- **`blocks/fragment/fragment.js`** exports `loadFragment(path)`, which fetches `{path}.plain.html`, rewrites relative media URLs to the fragment's own base, and recursively runs `decorateMain`/`loadSections` on it. This is the one block allowed to be imported by other code (see AGENTS.md) — `scripts.js` and `blocks/header/header.js` both import it directly.
- **`blocks/widget/widget.js`** is a dynamic-loading pattern: an authored `/widgets/<path>/<name>` link becomes a `widget` block, which fetches `<name>.html/.css/.js` from `/widgets/.../` at runtime, injects the HTML/CSS, and calls the widget module's default export. Query params on the widget link become `data-*` attributes on the block. Use this as the template if adding new dynamically-loaded, self-contained UI rather than a standard block.
- **`styles/styles.css`** loads eagerly (via `head.html`); **`styles/lazy-styles.css`** loads after first section render; **`styles/fonts.css`** loads conditionally (desktop width or already cached in `sessionStorage`) to avoid FOUC/CLS trade-offs — keep that split when adding global styles.
- Consent handling (`scripts/consent-check.js`, `scripts/consented.js`) is deferred to `loadDelayed` and gates any consent-requiring script.
