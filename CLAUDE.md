# Tinybook Formatter — Project Context

A standalone HTML tool for bookfunnels.io that turns a manuscript plus book metadata into a print-ready 5" × 8" interior PDF, and optionally an EPUB 3.0 for Apple Books, Kobo, Google Play, and KDP.

**Live URL:** https://garyw23gcwb.github.io/bookfunnels-formatter/
**Owner:** Gary White, Get Clients With Books (bookfunnels.io)
**Current version:** see `APP_VERSION` constant in `index.html` (v0.7.1 at time of writing)

---

## What it is

A single-file HTML app (`index.html`). Everything is client-side:

- React 18 via CDN (no build step)
- Babel standalone for JSX transform
- jsPDF 2.5.1 for PDF generation
- JSZip 3.10.1 for EPUB packaging
- mammoth.js 1.6.0 for .docx parsing
- Fontsource via jsDelivr for TTF font embedding

Users paste a manuscript or upload a `.txt`, `.md`, or `.docx`, fill in metadata, and click Generate. Output is a print-ready PDF with embedded fonts, drop caps with per-letter tuck-in, justified body text, running headers, TOC with dot leaders, and mirrored verso/recto margins. A second button generates an EPUB.

Hosted on GitHub Pages. Integrated as a custom menu link tool inside Gary's white-label GoHighLevel setup (bookfunnels.io), same pattern as the 3D Book Mockup Generator at `https://garyw23gcwb.github.io/book-mockup/`.

---

## Formatting spec

All measurements in PostScript points (72pt = 1 inch).

**Text size presets** (`TEXT_PRESETS` at the top of the script):

- `default`: 10pt body, 15pt leading
- `lisa` (labelled "Spacious"): 11pt body, 16.5pt leading

`createSpec(preset)` builds the active `SPEC` from the selected preset. `getDropCapSize()` derives the drop cap font size from current line height — no separate setting.

**Page (both presets):**

- Page: 360 × 576 pts (5" × 8")
- Mirrored margins:
  - Verso (left page): inside 42pt / outside 66pt / top 24pt / bottom 30pt
  - Recto (right page): inside 66pt / outside 42pt / top 24pt / bottom 30pt
- Chapter number label: Gelasio Regular 16pt
- Chapter title: Gelasio Bold 20pt
- Epigraph: EB Garamond Italic 18pt
- Drop cap: Gelasio Regular, auto-computed to span 3 body lines
  - Font size = `(dropCapLines × lineHeight) / capHeightRatio`
  - Default preset ≈ 62.5pt. Spacious preset ≈ 68.75pt.
  - Cap-height ratio for Gelasio/most serifs: 0.72
- Drop cap per-letter tuck-in: `DROP_CAP_TUCK` table tightens the right edge for L, T, I, F, J, P, Y so following text sits visually flush
- Body text is justified with a ragged last line (industry standard)
- Running headers: Inter Regular 8pt
  - Verso: book title, left-aligned
  - Recto: chapter title, right-aligned
- Page numbers: Inter Regular 8pt
  - Verso: bottom-left
  - Recto: bottom-right
  - Y position: 546pt from top

**Centering rule.** All centered text (CTA, title page, copyright, dedication, TOC, chapter titles, epigraphs) centers on the **text block midline** via `getCenterX()`, not the page midline. Mirrored margins make these two different by ~12pt — using page midline causes a visible leftward offset of text against full-width banners. Don't reintroduce `SPEC.pageW / 2` for centering.

---

## How the PDF rendering works

Two-pass rendering for accurate TOC page numbers:

1. **Pass 1 (dry run):** Calculate how many pages each chapter will consume without actually drawing anything. Store the page number each chapter will start on.
2. **Pass 2 (render):** Walk through front matter → TOC with real page numbers and dot leaders → chapters → back-of-book reader bonus CTA. Running headers and page numbers applied per page.

Chapters always start on recto (odd) pages. If a chapter would start on verso, insert a blank page first. Same rule applies to the back-of-book CTA — it lands on a fresh recto after the final chapter.

**Widow/orphan control** (`WIDOW_ORPHAN` constant): prevents 1–2 line paragraph tails alone at page top or bottom. Toggleable for debugging.

**Smart quotes:** all user text (titles, body, epigraphs, dedication, copyright) runs through `smartQuotes()` / `smartQuotesDeep()` before rendering.

**Drop cap positioning:** the top of the drop cap's cap-height aligns with the top of the first body line's cap-height. jsPDF positions text by baseline, so we work backwards:

```
firstLineCapTop = firstLineBaseline - (bodySize × capHeightRatio)
dropCapBaseline = firstLineCapTop + (dropCapSize × capHeightRatio)
```

The cap's measured width is reduced by `dropCapTuckedWidth(rawWidth, letter)` so narrow-shouldered caps (L, T, etc.) sit closer to the body text on their right.

---

## How the EPUB rendering works

`generateEPUB()` builds a valid EPUB 3.0 zip:

- One XHTML file per chapter, plus front matter and a back-of-book CTA
- CSS styles drop caps, justified text, italics
- `page-break-before` on whichever element opens the chapter (`p.chapter-num` when numbered, `h1.chapter-opener` when unnumbered) plus `page-break-after: avoid` so the title stays glued to the number — fixes the old bug where "Chapter 1" landed alone on one page with its title on the next
- mimetype, container.xml, content.opf, nav.xhtml all generated and zipped with JSZip
- File downloads with `.epub` extension

Amazon converts EPUB to Kindle format server-side at upload time, so this single file covers Apple Books, Kobo, Google Play, and KDP.

---

## .docx support

`mammoth.browser.min.js` runs `convertToHtml` with a style map so H1/H2 structure survives the conversion. `flattenDocxHtml()` walks the resulting DOM and emits:

- `h1` → `TITLE\n\n` (detected as chapter heading)
- `h2` → `## TITLE\n\n` (marked so render can style it as a bold oversized subheading)
- `p`  → `TEXT\n\n`
- `br` → `\n`
- everything else → inner text only

Result is fed to the same `detectChapters()` pipeline as pasted text.

Upload must use `arrayBuffer` format (not the raw File object) or mammoth crashes silently.

---

## Font embedding

Fonts are fetched from the Fontsource CDN on jsDelivr, converted to binary strings, and registered with jsPDF's VFS before any text is rendered.

**URL pattern:**
```
https://cdn.jsdelivr.net/fontsource/fonts/{slug}@5/latin-{weight}-{style}.ttf
```

Pinned to v5 for stable long-lived caching.

**Variants loaded:**

| Family | Weight | Style | jsPDF style |
|---|---|---|---|
| Gelasio | 400 | normal | normal |
| Gelasio | 700 | normal | bold |
| Gelasio | 400 | italic | italic |
| EB Garamond | 400 | italic | italic |
| Inter | 400 | normal | normal |

**Validation:** Each fetched file's first 4 bytes are checked — must be `00 01 00 00` (TrueType) or `OTTO` (OpenType). Failed fetches (wrong URL, 404 serving HTML) are skipped with a console warning.

**Historical failure:** v0.4.0 used the NPM path `https://cdn.jsdelivr.net/npm/@fontsource/gelasio@latest/files/gelasio-latin-400-normal.ttf` which returned 404. v0.5.0 switched to the correct Fontsource CDN pattern. Don't change this unless Fontsource's CDN pattern changes again.

**Verification:** After generating a PDF, check console for `Font embedding: 5/5 variants loaded`. If opening the PDF in Preview, check the Fonts panel — should see Gelasio, EBGaramond, and Inter as embedded subsets. File size should be ~59KB+ (vs ~13KB if fonts fall back to Helvetica).

---

## UI design

UI matches the Bookfunnels 3D Book Mockup Generator for visual consistency. Users see the same design language when moving between bookfunnels.io tools.

**Colour palette (see `BRAND` constant):**

- Page background: `#F5F7FA`
- Card background: `#FFFFFF`
- Primary text: `#1A212C` (navy)
- Secondary text: `#5A6478`
- Uppercase labels: `#6B7484`
- Borders: `#E8EBF0`
- Primary button: `#8B9BFF` base, `#4F5FD9` hover (strong brand blue, reads as clearly active vs disabled lavender at 0.55 opacity)
- Step badge background: `#EEF1FF` with `#3D87CC` number

**Layout:**

- Hero header with centered title and descriptive subtitle
- Main content in a single white rounded card (16px radius, subtle shadow)
- 5-step flow: Book Details → Manuscript → Chapters → Preview → Download
- Numbered step badges on each step heading
- Top-right corner: Load Sample Data and version badge
- Footer: small grey divider bar + "Powered by Bookfunnels.io"

**Typography:**

- UI font: Inter (via Google Fonts)
- Manuscript textarea: Gelasio (shows users what their text will look like)

---

## Development workflow

**Local testing:** Open `index.html` directly in a browser. No dev server needed — it's all client-side. Font fetching, PDF/EPUB generation, everything works from `file://`.

```
open index.html
```

**Deployment:** Commit to `main`, push to GitHub, GitHub Pages rebuilds in ~30 seconds. No build step.

**Before committing:**

1. Bump `APP_VERSION.version` (semver: patch for bug fixes, minor for features)
2. Add a one-line entry to `APP_VERSION.changes` at the top of the array
3. Verify brace/bracket/paren balance in the JSX:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const start = html.indexOf('<script type=\"text/babel\">') + 26;
const end = html.indexOf('</script>', start);
const jsx = html.substring(start, end);
const b = (jsx.match(/\{/g) || []).length - (jsx.match(/\}/g) || []).length;
const sq = (jsx.match(/\[/g) || []).length - (jsx.match(/\]/g) || []).length;
const p = (jsx.match(/\(/g) || []).length - (jsx.match(/\)/g) || []).length;
console.log('Deltas (should all be 0):', b, sq, p);
"
```

**Git conventions:**

- Branch: `main` (no feature branches for this project — single-file, single-developer)
- Commit messages: imperative mood, include version bump
  - Good: `v0.7.1: CTA page text now centers on text block, not page`
  - Bad: `fixed stuff`

---

## Current state

See `APP_VERSION.changes` in `index.html` for the full changelog.

**Known working at v0.7.1:**

- Font embedding (5 variants)
- Chapter auto-detection (patterns: "Chapter N", "Chapter N: Title", "N. Title", named sections like "Making It Happen")
- Epigraph parsing (both inline `"quote" — author` and multi-line)
- Drop cap sizing, alignment, and per-letter tuck-in
- Justified body text with ragged last line
- Two-pass TOC with accurate page numbers and dot leaders
- Mirrored margins with recto/verso logic
- Centering on text block (not page) — applied universally
- Front Reader Bonus page (optional CTA with URL) plus back-of-book CTA on recto after final chapter
- Title page, copyright page, dedication page, TOC
- Running headers alternating verso/recto
- Body size toggle (Default 10/15 vs Spacious 11/16.5)
- Smart chapter title wrapping (avoids orphaning "The" / "Of" / etc.)
- Widow/orphan control (no 1–2 line paragraph tails alone)
- Smart quotes conversion across all user text
- `.txt`, `.md`, and `.docx` upload
- EPUB 3.0 export (JSZip)

**Outstanding:**

- Full manuscript stress test on a real book to surface anything missed
- No known structural defects at v0.7.1

---

## Code structure

Single file, approximate layout for v0.7.1 (~1926 lines):

- Lines 1-25: `<head>`, external scripts (jsPDF, JSZip, mammoth, React, Babel), Google Fonts link for UI
- Lines 26-50: `BRAND` palette and `FONT` constant
- Lines 53-91: `TEXT_PRESETS`, `createSpec()`, `getDropCapSize()`
- Lines 94-98: `WIDOW_ORPHAN` settings
- Lines 100-134: `APP_VERSION` (version string + changelog array)
- Lines 143-215: `flattenDocxHtml()`, `smartQuotes()`, `smartQuotesDeep()`
- Lines 217-228: `DROP_CAP_TUCK` table and `dropCapTuckedWidth()`
- Lines 231-330: `detectChapters()` — regex-based chapter detection
- Lines ~330-390: `loadFontsForPDF()` — Fontsource CDN fetch + jsPDF VFS registration
- Lines 391-1119: `generatePDF()` — the main PDF rendering function
  - Helpers: `newPage`, `isRecto`, `getMargins`, `getTextWidth`, `getTextX`, `getCenterX`, `resetBodyFont`, `addRunningHeader`, `addPageNumber`, `wrapText`
  - Front matter rendering (CTA, title, copyright, dedication, TOC)
  - Two-pass logic
  - Chapter rendering with drop cap + tuck-in + justified body
  - Back-of-book CTA
- Lines 1120-1352: `generateEPUB()` — EPUB 3.0 zip builder
- Lines 1353+: React components (`StepBadge`, `ChapterCard`, `PagePreview`, `TinybookFormatter` main component with 5 steps)

---

## Gary's communication style

Direct, warm, brief, human. No em dashes anywhere in code comments, commit messages, or UI copy. No AI contrasting negation constructions ("not just X but Y"). Short sentences. Action-oriented.

When I finish a piece of work, tell Gary concisely what changed and what to test. Don't over-explain. If something requires his visual judgement (font looks right, drop cap position looks right), ask him to check specifically.

---

## Related projects

- **3D Book Mockup Generator:** `https://github.com/garyw23gcwb/book-mockup` — sibling tool, same UI design language
- **SIP skills pipeline:** 13-skill Claude pipeline for generating Strategic Implementation Plans. Separate project; lives in Gary's Claude skill system.
- **bookfunnels.io:** Gary's white-label GoHighLevel platform. These tools plug in as custom menu links.
