# Product Requirements Document (PRD)

> Working title: **PDF Context Reader** (Firefox extension). Replace with the real name when decided.

## 1. Problem & Summary

Reading an academic paper as a PDF means constant **jump navigation**: you click `[12]`, `Figure 3`, or `Section 4`, get thrown to another page, lose your place, and scroll back.

This extension removes that. When a PDF opens in Firefox's built-in viewer (PDF.js), it indexes the paper once, then:

1. shows a **Wikipedia-style hover card** when you hover any internal link, and
2. keeps a **sidebar** that shows the figures, tables, and section previews related to what is currently on screen.

You see the target without leaving your place.

## 2. Goals / Non-goals

**Goals**
- Never require a manual scroll just to peek at a figure, table, section, or reference.
- Sidebar reflects only the linked entities in the current viewport.
- Fast: index text in < 1.5 s for a 10-page paper; hover lookup < 5 ms (in-memory map lookup).
- Fully local: zero network requests, zero telemetry, no data written to disk.
- **Lightweight for daily use:** zero runtime dependencies; no bundled PDF engine, WASM, OCR, ML models, or local server. It must feel invisible on pages that are not PDFs and cheap on PDFs that are.

**Non-goals (v1)**
- No summarization / LLM features. No annotation or highlighting. No editing PDFs.
- No Chrome/Safari support. No OCR for scanned PDFs.
- No guarantees on slide decks or heavily graphical layouts; they degrade to fewer detected items.

## 3. Target Users & Scope

- **Users:** students and researchers reading arXiv-style papers on desktop Firefox.
- **Document types:** any PDF with a text layer: LaTeX papers, Word/Google Docs exports, reports, textbooks, preprints. One layout-based parser handles all of them; it does not depend on embedded PDF links or on how the PDF was produced.
- **Out of scope for v1:** scanned PDFs (no text layer; detected and skipped with a message) and slide decks.

## 4. Functional Requirements

### F1. One-time pre-processor
- Runs once per document after PDF.js reports the document is loaded; must not block scrolling.
- Builds `docIndex` in two phases:
  - **Phase A (text, blocking, < 1.5 s):** sections, citations, figure/table captions, and the map of in-text mentions (e.g. "Figure 3", "[12]") to entities.
  - **Phase B (images, background):** figure/table crops, prioritized by pages nearest the viewport.
- Entities:
  - **Section:** label, title, and a **200-character preview** (see F1.1).
  - **Citation:** marker, raw reference text, and best-effort title / authors / year.
  - **Figure / Table:** label, caption, cropped image.
- **F1.1 Preview rule (exact):** take the section body text that follows the heading, normalized (whitespace collapsed, line-break hyphenation removed, ligatures expanded, running headers/footers and page numbers removed, heading itself excluded). Store the first **200 Unicode code points**, no more, no less. The UI adds "…" if text was cut; the ellipsis is not part of the stored value or the 200.

### F2. Hover cards
- Trigger: pointer rests ~150 ms on a **mention** ("Figure 3", "[12]", "Section 4", "Smith et al., 2020") that the parser resolved to a known entity. Works whether or not the PDF has native links.
- Content by target type:
  - Citation → title, authors, year (falls back to raw reference text if parsing is uncertain).
  - Section → title + 200-char preview.
  - Figure / Table → thumbnail + caption snippet (text-only card until the crop is ready).
- Positioned near the cursor, flips/clamps to stay inside the viewport, dismissed on pointer leave or scroll.
- Text that does not resolve to a known entity shows no card. Clicks on native links are never intercepted.

### F3. Sidebar (`sidebar_action`)
- Lists entities mentioned in the text currently visible in the viewport: figure/table cards (image + caption) and section cards (title + 200-char preview). Citations are hover-only in v1.
- Updates on scroll with debouncing; stable order (by position in document); no flicker.
- Clicking a card scrolls the PDF to the target using PDF.js's native navigation, so the viewer's own back-history still works.
- Empty state: a short hint when nothing linked is visible.
- The sidebar cannot be opened automatically (Firefox requires a user action); the toolbar button toggles it.

## 5. Non-Functional Requirements

| Area | Requirement |
| :--- | :--- |
| Performance | Phase A < 1.5 s (10-page paper); scroll stays at 60 fps; hover lookup < 5 ms |
| Footprint | Zero runtime dependencies; shipped code target < 60 KB minified; no listeners or timers on non-PDF pages; no polling once the document is ready; at most 400 pages indexed |
| Privacy | No network calls. No host permissions beyond reading the PDF viewer page. No `storage.local` persistence of document content |
| Memory | Index and images live in the content script's memory (optionally `storage.session`), freed on tab close; crops are JPEG data URLs, max 600 px wide |
| Safety | All PDF-derived text is untrusted: rendered with `textContent`, never `innerHTML` |
| Compatibility | Firefox ≥ 115 (desktop), built-in PDF viewer; local `file://` PDFs require the user to allow file access |
| Resilience | If anything fails to parse, degrade silently to the normal viewer; never break native PDF.js behavior |

## 6. Edge Cases to Handle

- One- and two-column layouts, and full-width figures/tables spanning both columns.
- Numeric citations (`[12]`, `[3, 7]`, `[3–5]`) and author-year citations (`Smith et al., 2020`).
- Documents with no numbered headings, no references section, or no text layer (skip gracefully, never error).
- Sentences that start with "Figure 3 shows…" must not be mistaken for the caption itself.
- Sections that start at the bottom of a page and continue on the next.
- Subsections nested several levels (`3.2.1`) and appendices (`A`, `B.1`).
- Citations with multiple links in one marker (`[3, 7]`).
- Pages not yet rendered by PDF.js (lazy rendering): index from the document API, not the DOM.
- PDF re-opened or zoomed: index is keyed by document fingerprint and is zoom-independent.

## 7. Acceptance Criteria (MVP)

1. On *Attention Is All You Need* (arXiv 1706.03762): hovering `[17]` shows the right reference; hovering `Figure 1` shows the figure crop and caption; hovering `Section 3.2` shows its title and exactly 200 characters.
2. Scrolling to a paragraph that mentions Figure 2 shows Figure 2 in the sidebar within ~150 ms; scrolling away removes it.
3. Clicking a sidebar card scrolls to the target; Alt+← returns to the previous position.
4. DevTools Network panel shows no requests originating from the extension.
5. Meets the footprint targets in §5, and on a corpus of at least 8 PDFs from different sources runs without console errors and meets the eval thresholds in `SYSTEM_DESIGN.md` §10 (including mention precision ≥ 95%).

## 8. Phases

| Phase | Scope |
| :--- | :--- |
| 0 | Feasibility spike: confirm the content script can read PDF.js internals (see `ARCHITECTURE.md` §7) |
| 1 | Index (Phase A), hover cards for citations and sections |
| 2 | Sidebar + viewport tracking + reverse navigation |
| 3 | Figure/table crops (Phase B), crops in hover cards and sidebar |
| 4 | Corpus-wide parser tuning; footprint check; polish; packaging |
