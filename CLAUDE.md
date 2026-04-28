# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

BentoPDF is a fully client-side PDF toolkit (~80+ tools) shipped as a static site. There is no backend — all PDF processing happens in the browser via WASM modules and JS libraries. Privacy is the headline feature, so any work that would require uploading user files to a server is off-spec.

The build is dual-licensed (AGPL-3.0 / commercial); contributions require signing the CLA (see `CONTRIBUTING.md`).

## Common commands

| Task | Command |
|---|---|
| Dev server (port 5173) | `npm run dev` |
| Type-check + full build (i18n pages, sitemap, security headers) | `npm run build` |
| Build + bundled VitePress docs | `npm run build:with-docs` |
| Build for production w/ CDN WASM | `npm run build:production` |
| Preview the built `dist/` on port 3000 | `npm run serve` |
| Build + serve in Simple Mode (no marketing UI) | `npm run serve:simple` |
| Lint / autofix | `npm run lint` / `npm run lint:fix` |
| Security-only lint (no-unsanitized, eval) | `npm run lint:security` |
| Format with Prettier | `npm run format` |
| Tests (watch UI) | `npm run test` |
| Tests (single run, CI) | `npm run test:run` |
| Run a single test file | `npx vitest run src/tests/<name>.test.ts` |
| Coverage (thresholds: 80% lines/funcs/branches/statements) | `npm run test:coverage` |
| Docs dev / build / preview | `npm run docs:dev` / `docs:build` / `docs:preview` |
| Release (bumps version + tags) | `npm run release` (patch) / `release:minor` / `release:major` |

`npm run build` actually runs four steps in sequence: `tsc` → `vite build` → `scripts/generate-i18n-pages.mjs` → `scripts/generate-sitemap.mjs` → `scripts/generate-security-headers.mjs`. Treat `dist/` as authoritative output only after the full chain runs.

Husky + lint-staged run `eslint --fix` and `prettier --write` on commit. Don't bypass hooks.

## Architecture

### Multi-page static site, one bundle per tool

Each PDF tool is its own HTML entry point. The user navigates between pages — there is no SPA router. Entry points are explicitly enumerated in `vite.config.ts` under `build.rollupOptions.input`. Tool HTML lives in `src/pages/<tool>.html` (~117 tools), with category hub pages and marketing pages (`index.html`, `about.html`, `pdf-converter.html`, etc.) at the repo root. Adding a new tool means: HTML in `src/pages/`, a logic module in `src/js/logic/`, an entry in the rollup `input` map, an entry in `src/js/config/tools.ts`, and i18n keys.

`flattenPagesPlugin` in `vite.config.ts` rewrites `dist/src/pages/foo.html` → `dist/foo.html` after build. `rewriteHtmlPathsPlugin` rewrites absolute paths when `BASE_URL` is set (subdirectory hosting).

### Folder layout under `src/`

- `src/js/main.ts` — homepage bootstrap (i18n, lucide icons, tool grid, runtime config).
- `src/js/logic/<tool>-page.ts` — per-tool page controllers. One file per tool; each is loaded by the matching HTML page via `<script type="module">`. This is where most feature work happens.
- `src/js/utils/` — shared helpers: `helpers.ts`, `pdf-operations.ts`, `load-pdf-document.ts`, password prompts, image effects, plus WASM loaders (`pymupdf-loader.ts`, `ghostscript-loader.ts`, `cpdf-helper.ts`, `libreoffice-loader.ts`, `tesseract` runtime). WASM loaders read URLs from `VITE_WASM_*` envs at build time.
- `src/js/config/` — static tool catalog (`tools.ts`), font/language/TSA mappings, WASM CDN config.
- `src/js/handlers/fileHandler.ts` — shared file-input/drag-drop logic.
- `src/js/i18n/` — i18next setup; translations live in `public/locales/<lang>/`. Build-time generation (`scripts/generate-i18n-pages.mjs`) emits per-language copies of every page into `dist/<lang>/`. The dev server (`languageRouterPlugin` in `vite.config.ts`) routes `/de/foo` → `src/pages/foo.html`.
- `src/js/workflow/` — visual node-based workflow builder (Rete.js) that chains tools into pipelines.
- `src/js/canvasEditor.ts` + `src/js/logic/edit-pdf-page.ts` — the in-browser PDF annotation/redaction editor.
- `src/partials/` — Handlebars partials (navbar, footer, simple-mode variants) injected at build time via `vite-plugin-handlebars`. Branding (`brandName`, `brandLogo`, `simpleMode`, `appVersion`) is injected as Handlebars context.
- `src/types/` and `src/js/types/` — TypeScript types. Path alias `@/types` → `src/js/types/index.ts`.
- `public/workers/` — pre-built Web Workers (heavy ops like merge, attachments, PDF↔JSON, table-of-contents). Workers in this dir are excluded from ESLint and shipped as-is. New workers should usually live in TS under `src/` and be bundled.

### PDF processing libraries — pick the right one

The project layers multiple PDF engines and you must pick the right one:

- `pdf-lib` — pure-JS, bundled. Default for structural ops (merge, split, rotate, watermark, metadata, forms, encryption). Cannot render or rasterize.
- `pdfjs-dist` — Mozilla's renderer. For displaying PDFs and rasterizing pages to canvas.
- `pdfkit` + `blob-stream` — generation from scratch (CSV→PDF, JSON→PDF, etc.).
- `qpdf-wasm` (`@neslinesli93/qpdf-wasm`) — bundled. For repair, linearize, decrypt with stronger password handling than pdf-lib.
- `pymupdf` (CDN, AGPL) — text/markdown/SVG/DOCX extraction, image/table extract, EPUB/MOBI/XPS conversion, compression, deskew. Loaded via `pymupdf-loader.ts`.
- `coherentpdf`/cpdf (CDN, AGPL) — bookmark-aware merge, split-by-bookmarks, TOC, JSON, attachments. Loaded via `cpdf-helper.ts`.
- `ghostscript` (CDN, AGPL) — PDF/A and font-to-outline. Loaded via `ghostscript-loader.ts`.
- `libreoffice-wasm` (`@matbee/libreoffice-converter`) — Office formats → PDF. Loaded via `libreoffice-loader.ts`.
- `tesseract.js` — OCR. Worker URLs configurable via `VITE_TESSERACT_*`.
- `wasm-vips` — TIFF encoding. Excluded from `optimizeDeps`.
- `embedpdf-snippet` (vendored at `vendor/embedpdf/`) — the editing/annotation viewer. Its `pdfium.wasm` is copied into `dist/embedpdf/` by `viteStaticCopy`.

The three AGPL engines (PyMuPDF, Ghostscript, CPDF) are **not bundled** — they load from jsDelivr at runtime by default. Air-gapped deployments override `VITE_WASM_*` URLs to point at internal hosts.

### Build modes

- **Default** — full BentoPDF marketing site.
- **Simple Mode** (`SIMPLE_MODE=true`) — uses `simple-index.html` as the main entry, hides marketing/branding sections via the `__SIMPLE_MODE__` define and per-page DOM hiding in `main.ts`. Built for internal-org deployments.
- **Custom branding** — `VITE_BRAND_NAME`, `VITE_BRAND_LOGO`, `VITE_FOOTER_TEXT` injected through Handlebars partials.
- **Disabled tools** — `DISABLE_TOOLS=foo,bar` (build-time) or `disabledTools` in `dist/config.json` (runtime). `src/js/utils/disabled-tools.ts` enforces both.
- **Compression** — `COMPRESSION_MODE=g|b|o|all` controls whether `.gz`/`.br` siblings are emitted (vite-plugin-compression). `all` ships originals plus both compressed forms.
- **Subdirectory hosting** — `BASE_URL=/pdf/` rewrites all asset paths and is consumed by both Vite and `rewriteHtmlPathsPlugin`.

### Dev server quirks

- The dev server adds COOP/COEP headers (`same-origin` / `require-corp`) so SharedArrayBuffer-dependent WASM works.
- `/cors-proxy?url=...` middleware proxies requests to a fixed host allowlist (jsDelivr, Google Fonts, the workers.dev proxy, plus any host derived from `VITE_*` env URLs). Add hosts via `VITE_DEV_CORS_PROXY_EXTRA_HOSTS=host1,host2`. The production CORS proxy lives in `cloudflare/` (Workers).
- `VITE_DEV_HOST=0.0.0.0` exposes the dev server on the LAN (e.g., for mobile testing).

### Tests

Vitest in `jsdom`. Setup at `src/tests/setup.ts` polyfills `DOMMatrix` and clears the DOM between tests. PDF fixtures live in `src/tests/fixtures/`. Integration-style tests exist for the actual PDF operations (`pdf-operations.test.ts`, `digital-sign-pdf.test.ts`, etc.) — when adding a new tool, add at least a smoke test under `src/tests/` matching the existing naming pattern.

### Security posture

This is a security-conscious project (CodeQL, Trivy, no-unsanitized lint, CSP headers generated by `scripts/generate-security-headers.mjs`). When touching DOM or string-interpolation code, prefer `dompurify` and `escapeHtml` from `src/js/utils/helpers.ts` rather than direct `innerHTML` writes. The `lint:security` rule set is treated as errors in CI.
