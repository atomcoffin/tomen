# Tomen — Handoff for Fresh Conversation

This document captures the current state of Tomen so a new Claude conversation can pick up without losing context. The source of truth is the code in this repo; this document is orientation, not a substitute for reading the actual files.

## What Tomen is

An ebook reader PWA. Premium experience for sideloaders — the Calibre/r-ebooks crowd who already manage their own library and want a beautiful reader for it. Target: $12.99 one-time Android IAP through the Play Store. Free version available separately for desktop/web.

Positioning: **EPUB-only, beautifully rendered.** Not trying to compete with Kindle/Apple Books; trying to be the reader serious readers actually pick. Consistent Tomen typography across all books (Philosophy B rendering — we don't honor publisher CSS, we apply our own).

## Current committed state (April 2026)

### Shipped and working
- **Name + brand**: Tomen (capitalized, like Kindle). Repo at `atomcoffin/tomen` on GitHub Pages.
- **Chartreuse brand color**: `#C8FF00` light theme, `#B8E635` dark, `#B8D800` sepia. Token architecture: `--brand` (backgrounds/fills), `--brand-ink` (text on paper), `--brand-deep` (hover/active), `--brand-text` (text on brand backgrounds).
- **Typography**: Three reading faces — Siche Text/Display (serif), Uncut Sans Variable (sans), Dolph YY (mono). Licensing requires "reasonable precautions" per EULAs.
- **Nav architecture**: Dual toolbar (top + bottom), 720px max-width inner, subtle chartreuse-tinted paper background.
  - Top bar: Home icon (left), centered book title + time remaining, Settings/profile gear (right)
  - Bottom bar: Contents + Marks [+ Markup?] (left), Appearance (center), prev/next (right)
- **EPUB parser normalization**: Converts div-based paragraphs and chapter titles from publishers like Kobo/Calibre into semantic HTML. Handles `div.tx`/`div.tx1` → `<p>`, `div.ct` → `<h2>`, strips `span.koboSpan`, detects decorative section breaks, detects part-title pages (chapters with only a heading + <200 chars other content).
- **TOC with hierarchy**: Both EPUB 2 NCX and EPUB 3 nav parsers recursively walk nested structures. Picks whichever has more entries. TOC panel indents by depth.
- **Part-title pages**: Chapters detected as part dividers get large centered typography with a chartreuse underline flourish.
- **Heading sizes**: h1 2.4em, h2 2.0em (chapter titles), h3 1.4em. Generous margin below for breathing room.
- **Delete book UX**: Per-book `⋯` menu on library cards, confirmation dialog with preview, cleans up IDB + localStorage.
- **Touchscreen bottom bar fix**: Safe-area inset properly preserved on mobile so buttons aren't in iOS home-indicator gesture zone.
- **Swipe stripe**: Chartreuse vertical bar at edge, width grows with swipe progress, pops wider at threshold.

### Probably shipped but VERIFY before assuming
There's uncertainty about whether the following made it into the live build. Reason: a previous session discovered the code present locally but it's unclear if it was ever deployed, or if the local copy got polluted with speculative code. First action in the new chat is to verify.

- **Marks v2** — edit/delete marks, add notes, toggle bookmark. Search `index.html` for: `openMarkMenu`, `openNoteEditor`, `confirmDeleteMark`, `removeMarkAndUpdateUI`, `renderMarksList` (should render per-mark `⋯` menu buttons and note previews).
- **In-reader mark tap** — tapping a `.mark-highlight` in the rendered book should open the mark menu. Check `bookAreaTapped` for logic that calls `openMarkMenu` when `ev.target.closest('.mark-highlight')` matches.
- **Markup/highlighter button** in bottom bar — direct entry into marking mode, separate from the bookmark-icon Marks panel. Search for `ph-highlighter` and `enterMarkingModeFromToolbar`.

If everything above is PRESENT: test each interaction live and debug specific issues (mobile tap targets, menu clipping near screen edges, virtual keyboard pushing layout, first-time discoverability).

If anything is ABSENT: build from scratch following the patterns already established (book-menu popover style, confirmDialog reuse).

### Service worker cache
Last confirmed version: `tomen-v1.8.3`. Bump on every meaningful release.

## Known design decisions (don't re-litigate these)

- **Chartreuse is the brand color.** Don't second-guess. Chosen deliberately over ember, highlighter yellow, neon green, coral.
- **EPUB only.** No PDF support in v1. Positioning is "use Calibre to convert PDFs first." Don't bolt on PDF.
- **Philosophy B rendering.** Consistent Tomen typography across all books. We strip publisher CSS, apply our own.
- **Typography is a differentiator.** Ship curated faces, don't let users bring their own fonts.
- **Non-judgmental labels.** "Immersed/Steady/Brisk" for reading pace, not "Slow/Fast."
- **Hybrid time format.** Precise under 1h ("12 min"), fuzzy over ("about 2h 30min").
- **Chrome scope separation.** Top bar = app-level (leave book, global settings). Bottom bar = book-level (navigate this book, adjust reading, turn pages).
- **Brand color shows up in chrome and marks, not reading content.** Links in content use `--brand-ink` (dark olive), not `--brand` (chartreuse).
- **Spine colors per-book are auto-hashed.** Not brand chartreuse — each book gets its own spine color.

## Code structure

Single file, around 5500+ lines, at `index.html`. Key function clusters by approximate location:
- Top: HTML structure for screens + overlays + scripts
- Middle: CSS for library, reader, overlays, themes
- Bottom: JS — state, IDB wrappers, EPUB parser + normalizer, renderer, pagination, marks, touch handlers, helpers
- `sw.js` handles PWA caching; `manifest.json` for install metadata

## Todo list, in priority order

### 1. Verify Marks v2 state (see above)

### 2. More EPUB testing
Load 5-10 diverse books (Project Gutenberg, StandardEbooks, Calibre-converted MOBI, indie Vellum exports). Screenshot what breaks. Iterate on the normalizer's class hints (`_PARA_CLASS_HINTS`, `_HEADING_CLASS_HINTS`, `_BREAK_CLASS_HINTS`) to handle more publisher patterns.

### 3. Pro feature definition (non-code decision)
$12.99 IAP. What's free vs paid? Possibilities: limited library size, locked themes/fonts, no marks, free trial. Not a code task — a product decision everything else depends on.

### 4. Library search
Filter books by title/author substring. ~30 min of work.

### 5. Grid cover progress indicators
Small % overlay on cover thumbnails in grid view.

### 6. Landing page / marketing site
Single-page site. What is Tomen, screenshots, install instructions.

### 7. Play Store listing
Title, descriptions, feature graphic (1024×500), screenshots per device size, app icon, privacy policy, content rating.

### 8. Capacitor wrapping for Android
Convert PWA to installable APK/AAB for Play Store.

### 9. File association for .epub
Requires Capacitor intent filters.

### 10. Email Caterina Santullo
Typeface licensing for PWA + Play Store release. Still outstanding.

### 11. Repo privacy decision
GitHub Pro $4/mo for private Pages. Siche + Dolph EULAs have "reasonable precautions" clauses.

## Post-launch backlog

- Volume button page turning (Capacitor)
- System brightness integration (Capacitor)
- Per-book spine face (typography per-book)
- Collections / tags for library
- Reading stats (decide: does Tomen do gamification?)

## Explicitly deferred

- TTS
- Sync across devices
- Cloud backup
- PDF support
- Per-book fonts (user-selectable)
- Subscription model (rejected — audience is Calibre users, anti-subscription)

## Working conventions

- **Think out loud and push back when warranted.** Flag uncertainties. Don't be a yes-machine.
- **Honesty over enthusiasm**: "This might not work because X" > "Great idea, let's build it."
- **Verify before code**: grep the file, check what's already there, before writing new code.
- **One parse-check before shipping**: `node -e` snippet to validate `<script>` blocks.
- **Bump cache version on every shipped change.**
- **Single-file architecture** stays — don't suggest splitting into modules.

## How to start the new conversation

1. Attach these files: `HANDOFF.md` (this file), `index.html`, `sw.js`, `manifest.json`, `DESIGN.md`
2. Opening message to new Claude: 
   > "Picking up Tomen from a previous session. HANDOFF.md has the state. First task: verify whether Marks v2 is present in index.html by searching for the functions listed in that doc. Then we'll decide next steps based on what's actually there."
3. Let Claude verify state before doing any new work. Don't rush into features.
