# Architecture

The high-level picture: what the pieces are, why they are shaped this way, and in what order to build them. Product behavior lives in `PRD.md`. Algorithms, data schema, and message formats live in `SYSTEM_DESIGN.md`. Rules for coding agents live in `/AGENTS.md`.

**Core idea:** one standard method for every PDF. Read text with positions and fonts from PDF.js, run a small **layout-based parser** that finds sections, captions, references, and in-text mentions, and drive hover cards and the sidebar from its output. Nothing depends on how the PDF was produced (no LaTeX/hyperref names, no embedded link annotations).

**Design constraint: lightweight, for daily use.** The extension ships with **zero runtime dependencies**: no bundled PDF engine, no WASM, no OCR, no ML models, no local server. It reuses the PDF.js that Firefox already has.

## 1. Key Decisions

| # | Decision | Why |
| :- | :--- | :--- |
| D1 | Read the PDF through the **PDF.js document API** (`getPage().getTextContent()`), not by scraping `.textLayer` spans | PDF.js renders pages lazily; the DOM only has pages near the viewport |
| D2 | **One parser for all PDFs**: geometry + font + text patterns, no per-producer code paths | Works for LaTeX, Word exports, reports, textbooks. Weak documents yield fewer entities; they don't break |
| D3 | The parser finds **mentions** (`Figure 3`, `[12]`, `Section 4`, `Smith et al., 2020`) with their rectangles and resolves each to an entity. Native PDF links are ignored | Same behavior with or without embedded links; one code path to test |
| D4 | **Own hit-testing, no per-mention DOM**: one delegated `pointermove` listener checks the pointer against the parser's mention rects | No overlay elements to sync with lazy rendering; never blocks clicks or text selection |
| D5 | **Viewport tracking is geometric**: visible page region ∩ mention rects | No observers on thousands of nodes; works on pages PDF.js hasn't rendered |
| D6 | The parser is **pure functions** in `src/core/` (no DOM, no `browser.*`) taking plain JSON | Testable in Node on real PDFs without Firefox |
| D7 | **Two-phase indexing**: text structure first (< 1.5 s), image crops later in idle time | Meets the time budget; cards work text-only until crops arrive |
| D8 | Crops are **JPEG data URLs**, not Blob URLs | A Blob URL made in the page-origin content script is not reachable from the sidebar's `moz-extension://` origin |
| D9 | Sidebar receives **entity IDs only** and pulls payloads on demand; no background-script state | Small messages; no state lost to event-page unloads |
| D10 | Navigate with PDF.js's own `pdfLinkService.goToDestination` and an explicit destination array | Reuses native scrolling and the viewer's back-history |
| D11 | **In-memory only**; `storage.session` optional, never `storage.local` | Privacy; evicts on tab close |
| D12 | **No heavy libraries, ever, at runtime.** Dev-only tools are allowed if they never ship | Keeps the extension small, fast to load, and easy to audit |

## 2. System Overview

```
┌──────────────────────── Firefox tab: PDF.js viewer ────────────────────────┐
│ page world: window.PDFViewerApplication (pdfDocument, pdfLinkService, ...) │
│                       ▲ via window.wrappedJSObject                         │
│ ┌─────────────────────┴──────────────── content script ──────────────────┐ │
│ │ pdfjs-bridge ─► ExtractedDoc ─► core/parse ─► docIndex                 │ │
│ │ (only file          (page-space     (pure JS layout parser)            │ │
│ │  touching PDF.js)    JSON)                                             │ │
│ │ hover.js    : pointermove → hit-test mentions → popover                │ │
│ │ viewport.js : scroll → visible region ∩ mentions → active entity ids   │ │
│ │ crop.js     : idle queue → page-region canvas → JPEG data URL          │ │
│ └──────────────┬─────────────────────────────────────────▲───────────────┘ │
└────────────────┼─────────────────────────────────────────┼─────────────────┘
      runtime.sendMessage                          tabs.sendMessage
                 ▼                                         │
           Sidebar (sidebar.html/js): cards for the active tab; click → navigate
           background.js: toolbar button → sidebarAction.toggle()
```

## 3. Repository Layout

```
AGENTS.md  opencode.json
docs/PRD.md  docs/ARCHITECTURE.md  docs/SYSTEM_DESIGN.md
manifest.json  package.json
src/
  core/                    PURE. No DOM, no browser.*. Node-testable.
    types.js  text.js  layout.js  clean.js  headings.js
    captions.js  references.js  mentions.js  parse.js
  content/
    main.js  pdfjs-bridge.js  hover.js  viewport.js  crop.js
  sidebar/  sidebar.html  sidebar.js  sidebar.css
  background/  background.js
tools/                     DEV ONLY, never shipped
  extract-fixture.mjs      PDF → ExtractedDoc JSON (Node)
  eval.mjs                 run parser on corpus, print scores
test/
  corpus/                  PDFs + <name>.pages.json + <name>.expected.json
  core/*.test.js
```

## 4. Manifest (shape)

- Manifest V3, `browser_specific_settings.gecko.strict_min_version: "115.0"`.
- `sidebar_action` with `default_panel: sidebar/sidebar.html`; `action` button toggles it (inside a user-action handler, as Firefox requires).
- Content script `matches: ["<all_urls>"]`, `run_at: document_idle`. Many PDFs have no `.pdf` in the URL, so URL patterns can't detect them. `main.js` exits immediately, with no listeners or timers, unless `document.contentType === 'application/pdf'` and `PDFViewerApplication` exists.
- No permissions beyond the content-script match; no remote hosts.

## 5. Lifecycle

```
Tab loads PDF in PDF.js → content script: not a PDF viewer? exit (zero cost)
  ▼
bridge waits for pdfDocument → extracts page text items (chunked, yielding between pages)
  ▼
core/parse → docIndex                              (Phase A, target < 1.5 s / 10 pages)
  ▼
attach hover (pointermove) + viewport tracker; start crop queue (Phase B, idle time)
  ▼
scroll → visible region ∩ mentions → debounce → VIEWPORT_CHANGED → sidebar GET_ENTITIES
hover  → hit-test mentions → entity → popover
click card → NAVIGATE_TO_ENTITY → pdfLinkService.goToDestination
```
No text layer (scanned PDF) → record a warning, attach nothing, tell the user in the sidebar.

## 6. Footprint Budget (daily-use constraint)

| Area | Budget |
| :--- | :--- |
| Shipped code | Zero runtime dependencies; target < 60 KB minified (≈ 150 KB source) |
| Non-PDF pages | Content script returns immediately: no listeners, timers, or observers |
| Idle CPU on a PDF | No polling after the document is ready; scroll and pointer handlers are rAF-throttled and passive |
| Parse time | < 1.5 s for 10 pages; chunked so no single task exceeds ~50 ms; large documents index progressively |
| Page cap | Index at most 400 pages; beyond that, warn and index the first 400 plus the last 40 (for references) |
| Memory | Crops are JPEG ≤ 600 px wide, ≤ 60 KB each, LRU cache of at most 40; all freed on tab close |
| Hover latency | < 5 ms hit-test; popover appears 150 ms after the pointer rests |

## 7. Risks and Spikes (resolve in M0)

| ID | Risk | Spike / mitigation |
| :- | :--- | :--- |
| R1 | Content script may not be able to read `PDFViewerApplication` in Firefox's viewer | Minimal extension that logs `wrappedJSObject.PDFViewerApplication.pdfDocument.fingerprints`. If it fails, re-plan before building more (see below) |
| R2 | Page-space conversion errors (rotation, zoom) make hit-test rects drift | Draw debug rectangles for all mentions on a few PDFs; keep all conversion in the bridge |
| R3 | Layout heuristics fail on some document types | Corpus eval drives tuning; failures degrade to fewer entities, never to wrong cards |
| R4 | Figure/table crop heuristics fail on unusual layouts | Text-only card fallback; tune on corpus |
| R5 | Two-column or mixed layouts misorder text, corrupting previews | Column detection tests on corpus; previews must pass the property checks |
| R6 | Callbacks into page-world objects need `exportFunction` | Prefer polling until ready, then stop |
| R7 | `file://` PDFs need explicit user permission | Document in README |
| R8 | Slide decks, scans, heavy graphics | Out of scope for v1; detect and degrade gracefully |

If R1 fails, the fallback is to read the PDF bytes ourselves. That would mean bundling a PDF text extractor, which conflicts with D12, so it is a decision to make together at that point rather than something to add by default.

## 8. Build Order (one milestone per agent session)

| M | Deliverable | Done when |
| :- | :--- | :--- |
| M0 | Scaffold, manifest, scripts, `tools/extract-fixture.mjs`, spikes R1/R2 | Logging extension prints fingerprint and page-space text items on an arXiv PDF; extract script produces `pages.json` for the corpus |
| M1 | `core`: `text`, `layout`, `clean` + tests | Lines/columns/cleaning tests pass on the corpus |
| M2 | `headings` + section previews | Heading eval ≥ target; preview property checks pass |
| M3 | `references` + `captions` | Reference and caption eval ≥ target |
| M4 | `mentions` + hover popover (text-only cards) | Mention precision ≥ 95%; hover works on 3 corpus PDFs; debug-rect view matches text |
| M5 | `viewport.js` + sidebar + navigation | PRD acceptance criteria 2 and 3 |
| M6 | `crop.js` and image cards in hover and sidebar | PRD acceptance criterion 1 (figure part) |
| M7 | Corpus-wide tuning, footprint check, packaging (`web-ext build`) | All acceptance criteria; zero console errors on corpus; size and CPU budgets met |
