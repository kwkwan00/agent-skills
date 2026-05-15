---
name: html-to-pdf-export
description: Build a client-side HTML→PDF exporter for React/Vite apps using `html-to-image` (rasterization) + `@react-pdf/renderer` (page assembly). Use when implementing "Export PDF" buttons that need to capture rendered DOM (whitepapers, dashboards, reports) and produce a paginated A4/Letter PDF the operator downloads. Triggers include requests like "add a PDF export to this page", "the PDF is cutting off paragraphs", "PDF pages have huge whitespace", "the PDF is missing headings/tables", or any debugging of html-to-image + @react-pdf/renderer pagination output. Skip for server-side PDF generation, vector-PDF needs, or print-CSS approaches (`window.print`).
---

# Client-side HTML→PDF Export (html-to-image + @react-pdf/renderer)

This skill packages the working recipe for in-browser HTML→PDF export under the constraint that the input is a **rendered React component tree** (typically a long-form whitepaper or dashboard page) and the output is a **paginated A4 PDF** the operator downloads. The two libraries — `html-to-image` for rasterization and `@react-pdf/renderer` for page assembly — interact in subtle ways that produced multiple regressions before converging on the approach below.

## Tech-stack assumption

```json
"dependencies": {
  "html-to-image": "^1.11.x",
  "@react-pdf/renderer": "^4.x",
  "react": "^18.x"
}
```

Vite + TypeScript. No server-side step.

## When to use this skill

Trigger this skill when:
- Implementing an "Export PDF" affordance on a React-rendered page.
- Debugging a paginated PDF whose **content is being cut mid-paragraph** or **missing headings**.
- Pages in the generated PDF have **large trailing whitespace** because a section couldn't fit on the remaining page.
- A `<h3>` / heading element renders correctly in-browser but appears blank or invisible in the exported PDF.

Skip this skill (use a different approach) when:
- Server-side PDF generation is acceptable (use Puppeteer + headless Chrome).
- Vector PDFs are required (use `@react-pdf/renderer` directly with primitives — no rasterization).
- The page already prints correctly via `window.print()` and a "Save as PDF" dialog is acceptable UX.

## The architecture (short version)

1. **Capture each top-level section as ONE canvas** via `html-to-image.toCanvas`. Section here means each direct child of the export-container element (typically each `.card`).
2. **Slice each section's canvas at internal element boundaries** (paragraphs, headings, callouts, tables, figures) into mini-blocks. Slices are sub-rectangles of the section's canvas — internal text rendering is preserved exactly.
3. **Bin-pack mini-blocks across section boundaries** into A4-sized pages. A page closes only when the next mini-block would overflow.
4. **Render each page as one rasterized image** inside `@react-pdf/renderer`'s `<Page>` with width-or-height fill depending on aspect ratio.

Why this combination of choices is load-bearing:

| Decision | Why |
|---|---|
| Capture each *section* whole, not each `<h3>` individually | `html-to-image` clones into a `<foreignObject>` SVG **out of the original DOM context**. Isolated `<h3>` captures lose their text rendering (blank or invisible). Whole-section capture keeps the heading nested in its parent during clone, which is what the cascade and font resolution need. |
| Slice the section *canvas* (not the DOM) at element bottoms | Slicing the rasterized canvas vertically just extracts a horizontal strip; whatever rendered correctly inside the section's canvas stays correct. This avoids the per-element capture trap above. |
| Mini-block bin-packing across sections | Section-level page boundaries leave ~50% trailing whitespace whenever a section is too tall for the remaining page. Mini-block packing densely fills pages by mixing the tail of one section with the head of the next. |
| `pageWidth = max(blocks.map(b => b.width))` not `blocks[0].width` | Different elements have different widths (the trailing footer is wider than card-padded children). Using the first block's width and `drawImage` silently clips the right edge of any wider block. |
| `INTER_BLOCK_GAP_PX = 0` between mini-blocks | Mini-blocks already include their original surrounding margins (the section was captured intact, slices are sub-rectangles). Adding a synthetic gap double-spaces. |
| Aspect-aware Page rendering | A slice taller than A4 (rare — paragraphs > 50 lines) needs `height: 100%` not `width: 100%` so it scales to fit instead of overflowing and clipping. |
| **Hide** the export button via `style.display = "none"`, **don't** `filter` it | `html-to-image`'s `filter: el => el.id !== EXPORT_BTN_ID` removes the element from the clone, but **its layout slot collapses**, shifting subsequent content up by ~28px. Breakpoints collected from the live DOM no longer match the rasterized canvas. Using `display: none` (and restoring in `finally`) makes live DOM and captured canvas match exactly. |

## Gotchas worth flagging early

1. **Per-element capture breaks heading rendering.** `html-to-image.toCanvas(h3Element)` produces a canvas where the `<h3>`'s text often doesn't render. Symptoms: thin invisible lines on the page where headings should be. Fix: capture the section, slice the canvas afterward.

2. **Forced pixel cuts produce mid-paragraph cuts.** Earlier iterations used `cursor + pageHeightPx` arithmetic to plan boundaries. When a single paragraph exceeds a page (long-form text + narrow viewport), no breakpoint fits the budget and the planner cuts mid-sentence. Fix: when no breakpoint fits inside `(cursor, target]`, **overshoot** to the smallest breakpoint past `target` and scale-to-fit on render.

3. **`filter` shifts layout.** See above. Use `style.display = "none"` instead.

4. **Width mismatches cause silent right-edge clipping.** `ctx.drawImage(c, x, y)` clips silently when `x + c.width > canvasWidth`. Always use max-width for the page canvas and clamp `x ≥ 0`.

5. **`box-sizing: border-box` resets and `style.height` overrides.** Setting `options.height` or `options.style.height` on the cloned element (to add padding for descender bleed) interacts badly with global border-box resets. Don't try to fix glyph clipping by inflating individual element captures — capture sections whole instead.

6. **Whole-section captures leave trailing whitespace** when section heights are uneven. Only a fine-grained mini-block bin-packer fills pages densely. Don't stop at "section capture works"; iterate to mini-block packing.

## Reference implementation

A complete, drop-in-able TypeScript implementation lives at `references/exportPdf.tsx` in this skill directory. It exports a single function:

```ts
exportWhitepaperPdf({
  container: HTMLElement,    // the element whose children get captured
  filename: string,          // output PDF filename
  title: string,             // PDF metadata title
  hideElementId?: string,    // id of the export button to hide during capture
}): Promise<void>
```

To use:

```ts
import { exportWhitepaperPdf } from "./whitepaper/exportPdf";

const containerRef = useRef<HTMLDivElement>(null);
const [exporting, setExporting] = useState(false);

const onExport = async () => {
  if (!containerRef.current) return;
  setExporting(true);
  try {
    await exportWhitepaperPdf({
      container: containerRef.current,
      filename: "my-whitepaper.pdf",
      title: "My Whitepaper",
      hideElementId: "pdf-export-btn",
    });
  } finally {
    setExporting(false);
  }
};

return (
  <div ref={containerRef}>
    <header>
      <h1>My Whitepaper</h1>
      <button id="pdf-export-btn" onClick={onExport} disabled={exporting}>
        {exporting ? "Exporting…" : "Export PDF"}
      </button>
    </header>
    <section className="card">…</section>
    <section className="card">…</section>
    {/* etc */}
  </div>
);
```

## Required CSS conventions

The reference implementation's slicer uses these selectors as breakpoints:

```
.card-title, figure, hr, .callout, table, tr, li, p, h3
```

Make sure the page renders these as standard block-level elements. If a custom heading component renders as a `<div>`, give it the `.card-title` class (or one of the other selectors) so the slicer can find its bottom edge. The `Callout` component in particular often renders as a styled `<div>` with no class — add `className="callout"` so the slicer treats it as an atomic block instead of cutting through its content.

## When to revisit

The mini-block bin-packing approach has known trade-offs:

- **Card frames are lost between section's mini-blocks** that share a page. If preserving the visual `.card` border around content groups matters more than dense packing, fall back to whole-section capture (one section = one page minimum) and accept the trailing whitespace.
- **Page text is rasterized**, not selectable / searchable. If selectable text is required, switch to native `@react-pdf/renderer` primitives (`<Text>`, `<View>`) — but that's a different skill (full content rewrite onto PDF primitives instead of rasterizing the rendered DOM).
- **Slow on very long documents** (~5–8 sections × 200ms toCanvas each = ~1–2 seconds). Parallelize via `Promise.all` (already done in the reference).

If a future export gains a new element type (definition lists, blockquotes, code blocks taller than a page), add the relevant tag to `INTRA_SECTION_BREAK_SELECTORS` and verify the bin-packer still produces dense pages.
