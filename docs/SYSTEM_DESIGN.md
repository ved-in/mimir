# System Design

The detailed "how": data shapes, the parser pipeline, UI algorithms, messages, and tests. For the big picture and build order see `ARCHITECTURE.md`; for product behavior see `PRD.md`.

All code here is plain JavaScript with no runtime dependencies.

## 1. Coordinate Convention

**Page space:** origin at the top-left of the page as PDF.js displays it at scale 1 (page `/Rotate` applied), units = CSS px at scale 1. Display scale for a page is `pageElement.clientWidth / page.width`.

Only `pdfjs-bridge.js` knows about PDF user space (origin bottom-left). It converts everything to page space on the way in, and back to user space only inside `goToAnchor`. Core code never sees PDF coordinates.

## 2. Extraction Layer (`pdfjs-bridge.js`)

Gets the viewer via `window.wrappedJSObject.PDFViewerApplication`, waits for `pdfDocument` (poll every 100 ms with a timeout, then stop polling), and returns plain JSON so nothing page-world leaks into core.

```ts
interface ExtractedDoc {
  fingerprint: string;
  pages: ExtractedPage[];
}
interface ExtractedPage {
  index: number;                 // 0-based
  width: number; height: number;
  items: TextItem[];             // in content-stream order
}
interface TextItem {
  str: string;
  x: number; y: number;          // top-left, page space
  w: number; h: number;
  fontSize: number;
  fontName: string;
  bold: boolean; italic: boolean;   // inferred from font name/style
  rot: 0 | 90 | 180 | 270;       // non-zero items are ignored by the parser (side stamps, watermarks)
}
```

Helpers: `getPageElement(i)`, `getScale(i)`, `getViewerContainer()`, `onPageRendered(cb)`, `renderPageRegion(i, rect, scale)`, and `goToAnchor(i, x, y)` which converts to user space and calls `pdfLinkService.goToDestination([i, {name:'XYZ'}, xPdf, yPdf, null])`. If `PDFViewerApplication` is unavailable the extension disables itself quietly.

Extraction is chunked: one page per task, yielding with `setTimeout(0)` between pages. Pages are processed in this order: the last ~40 pages (references), then pages outward from the current page.

## 3. Parser Pipeline (`src/core/parse.js`)

Input `ExtractedDoc`, output `DocIndex`. Each stage is a pure function with its own unit tests.

1. **Lines.** Drop `rot ≠ 0` items. Group items into lines by baseline (tolerance ≈ 0.3 × fontSize), sort by `x`, insert a space when the gap exceeds ≈ 0.25 × fontSize. Keep a map from every character offset in the line text to its item and x-range (needed for mention rects).
2. **Columns and reading order.** Per page, histogram of line left edges and widths gives 1 or 2 columns; full-width lines (titles, wide captions, tables) are spanning blocks. Order: spanning blocks and columns top to bottom, left column before right within a band. Content-stream order is only a tie-breaker, never trusted.
3. **Clean.** Remove running headers/footers (lines in the top/bottom ≈ 8% of the page whose text with digits replaced by `#` repeats on ≥ 40% of pages), standalone page numbers, and line-break hyphenation (`exam-` + `ple` → `example`). Normalize NFKC (expands ligatures) and collapse whitespace.
4. **Body font.** Body font size = the size carrying the most characters; the reference for heading scoring.
5. **Sections** (`headings.js`). Score each short line (< ~100 chars):
   - larger than body by ≥ 5%, or bold at body size, or ALL CAPS: +
   - numbering `^\d+(\.\d+)*\.?\s+\S`, `^[A-Z](\.\d+)*\.?\s+\S` (appendices), roman numerals: +
   - known unnumbered names (Abstract, Introduction, Conclusion(s), References, Acknowledg(e)ments, Appendix): +
   - larger vertical gap above than the normal line gap: +
   - ends with a period, sentence-length, or inside a caption/table block: −

   Accept above a threshold. Level = numbering depth, else font-size rank. Label = numbering text if present. Documents with no headings produce no section entities.
6. **Captions** (`captions.js`). A line that starts a block with `^(Figure|Fig\.|Table|Tab\.)\s*(\d+|[A-Z]\.\d+)` **and** has a label delimiter (`:`, `.`, `|`, en dash) or a bold/different-style label. That separates real captions from sentences like "Figure 3 shows…". The caption extends until a vertical gap > 1.5× line height or a style change.
7. **References** (`references.js`). Find the last heading matching `References|Bibliography|Works Cited|Literature Cited`. Split entries by, in order of preference: leading `[n]`, leading `n.`, hanging indent (new entry = line at the left margin after indented lines), vertical gaps. Keys: numeric → `ref-<n>`; author-year → `ref-<surname><year>[a-z]`. Title/authors/year are best effort and omitted when uncertain; `raw` (≤ 400 chars) is always stored.
8. **Mentions** (`mentions.js`). Run patterns over cleaned line text, excluding caption labels, headings, and the reference list:
   - `(Figure|Fig\.|Figs\.)\s*(\d+)[a-z]?`, `(Table|Tab\.)\s*\d+`
   - `(Section|Sec\.|§)\s*\d+(\.\d+)*`, `Appendix\s*[A-Z]`
   - numeric citations `\[\d+([,–-]\s*\d+)*\]` (ranges expanded; each number gets its own sub-rect by character offset)
   - author-year `(Smith et al., 2020)`, `Smith and Jones (2020)`, `(Smith, 2020; Lee, 2019)`

   A mention is kept **only if its target entity exists**. Rect = union of the item x-ranges covered by the match, expanded 1–2 px. Ambiguous author-year keys resolve to the best match or are dropped (a missing card beats a wrong one).
9. **Previews.** Section body = following lines in reading order until the next section heading, cleaned as in step 3, heading excluded. `preview = takeCodePoints(body, 200)`: exactly 200 Unicode code points, or fewer if the section is shorter. The UI appends "…" when cut; it is not stored.
10. **Assemble** `docIndex` plus `quality` counts and warnings (`no text layer`, `no references section`, `page cap applied`).

### Figure/table crop rects (heuristics)
- **Figure:** region above its caption, from the bottom of the previous text block (or top margin) to the caption's bottom, within the caption's column (full width if the caption spans).
- **Table:** region below its caption, down to the next body-text block or the first vertical gap > ≈ 2× line height after table-like rows. If the caption sits *below* a dense block of short lines, use the region above.
- **Reject** (text-only card) if taller than 60% of the page, narrower than 20% of the column, or mostly body paragraphs.

## 4. Data Schema

```ts
interface DocIndex {
  fingerprint: string;
  pageCount: number;
  entities: Record<string, Entity>;
  mentions: Record<number, Mention[]>;      // key = pageIndex; each list sorted by rect.y
  status: { text: 'ready'; images: 'pending' | 'partial' | 'ready' };
  quality: { sections: number; figures: number; tables: number; references: number;
             mentions: number; warnings: string[] };
}

interface Rect { x: number; y: number; w: number; h: number }   // page space
interface Anchor { pageIndex: number; x: number; y: number }    // page space, top-left of the target

interface Mention {
  rect: Rect;
  text: string;           // "Figure 3", "[12]"
  entityId: string;       // one entity per mention ("[3, 7]" produces two mentions)
}

type Entity = SectionEntity | CitationEntity | FigureEntity | TableEntity;

interface EntityBase {
  id: string;             // "sec-3.1" | "ref-12" | "ref-vaswani2017" | "fig-2" | "tab-1"
  kind: 'section' | 'citation' | 'figure' | 'table';
  anchor: Anchor;         // navigation target; also the document-order key
}
interface SectionEntity extends EntityBase {
  kind: 'section'; label: string; title: string; level: number;
  preview: string;        // first 200 code points of the body
}
interface CitationEntity extends EntityBase {
  kind: 'citation'; raw: string;
  title?: string; authors?: string; year?: string;   // best effort only
}
interface FigureEntity extends EntityBase {
  kind: 'figure'; label: string; caption: string;
  cropRect?: Rect; cropDataUrl?: string;             // Phase B
}
interface TableEntity extends Omit<FigureEntity, 'kind'> { kind: 'table' }
```

## 5. Hover (`hover.js`)

- One `pointermove` listener on the viewer container, passive, rAF-throttled. Find the page with `event.target.closest('.page')`, convert the pointer to page space `((clientX - pageRect.left) / scale, (clientY - pageRect.top) / scale)`, and hit-test that page's mention list (binary search on `y`, then a short linear scan).
- On hit: start a 150 ms hover-intent timer, then show the popover. Cancel on leave, scroll, or `Esc`. Optional light highlight: one reusable element positioned over the hovered mention.
- Never intercepts clicks, text selection, or native links (no overlay elements with pointer events).
- Popover: a single host element on `document.body` with a **shadow root** (style isolation). Content is set with `textContent` / `img.src`, never `innerHTML` (PDF text is untrusted). Placement below the cursor, flipped above if it would overflow, clamped horizontally, max width ≈ 360 px.
- Content by kind: citation → title/authors/year, else `raw`; section → title + 200-char preview; figure/table → thumbnail + caption snippet (text-only until the crop exists).

## 6. Viewport Tracking (`viewport.js`)

- On scroll/resize (rAF, then ~100 ms trailing debounce): for each page element intersecting the viewer container, compute the visible vertical span in page space, then collect mentions whose rect intersects it.
- Entities from those mentions are deduplicated and ordered by `(anchor.pageIndex, anchor.y)`.
- Emit `VIEWPORT_CHANGED { activeEntityIds }` only when the ordered list changed.
- Needs only page element size and position, so it works for pages PDF.js hasn't rendered.

## 7. Crops (`crop.js`, Phase B)

- Queue ordered by distance from the current page; run with `requestIdleCallback` (fallback `setTimeout`), one page at a time.
- Render the region at scale ≈ 1.5 to an offscreen canvas, downscale to ≤ 600 px wide, `toDataURL('image/jpeg', 0.8)`. One page render serves every crop on that page.
- Keep at most 40 crops (LRU); on completion set `cropDataUrl` and emit `INDEX_STATUS`.

## 8. Sidebar (`sidebar.js`)

- Listens for `VIEWPORT_CHANGED` / `INDEX_STATUS` where `sender.tab.id` is the active tab; on tab switch calls `GET_VIEWPORT`.
- Pulls payloads with `GET_ENTITIES { ids }` for IDs not yet cached; renders with keyed diffing so unchanged cards are never re-created.
- Click → `NAVIGATE_TO_ENTITY { entityId }` → `bridge.goToAnchor(...)`.
- Empty states: not a PDF, index not ready, no text layer, nothing mentioned in view.
- Citations are hover-only in v1 (not listed in the sidebar).

## 9. Messaging Protocol

| From → To | Transport | Type | Payload |
| :--- | :--- | :--- | :--- |
| content → sidebar | `runtime.sendMessage` | `INDEX_STATUS` | `{ text, images, quality }` |
| content → sidebar | `runtime.sendMessage` | `VIEWPORT_CHANGED` | `{ activeEntityIds: string[] }` |
| sidebar → content | `tabs.sendMessage` | `GET_VIEWPORT` | `{}` → `{ activeEntityIds }` |
| sidebar → content | `tabs.sendMessage` | `GET_ENTITIES` | `{ ids }` → `Entity[]` |
| sidebar → content | `tabs.sendMessage` | `NAVIGATE_TO_ENTITY` | `{ entityId }` |
| action button → background | event | `action.onClicked` | toggles the sidebar |

## 10. Testing and Evaluation

- **Corpus:** `test/corpus/` with ≥ 8 PDFs the user actually reads, from different sources: LaTeX 2-column, LaTeX 1-column, Word/Google Docs export, textbook chapter, technical report, author-year citations, many tables, appendices. Each has `<name>.pages.json` (made by `tools/extract-fixture.mjs`, a dev-only Node script) and a hand-written `<name>.expected.json` (section titles, figure/table/reference counts, ~10 spot-checked mentions).
- **Unit tests** (`npm test`, Vitest, dev-only): one per stage, plus `takeCodePoints`.
- **Property checks** on every corpus document: every `preview` has ≤ 200 code points, and exactly 200 when the section is long enough; no preview contains a running header or page number; every mention resolves to an existing entity; mention rects lie inside the page.
- **Eval** (`npm run eval`) prints per-document and overall scores. Targets: heading recall and precision ≥ 90% on numbered-heading documents; caption recall ≥ 90%; reference count exact on ≥ 80% of documents; mention precision ≥ 95%; Phase A within budget.
- **Manual acceptance:** the criteria in `PRD.md` §7 via `web-ext run` with the Network panel open.
