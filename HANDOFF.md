# Tomen — Handoff for Fresh Conversation

This document captures the current state of Tomen after the April 23 2026 session. The source of truth is the code in the repo; this is orientation, not a substitute for reading the files.

## What Tomen is

An ebook reader PWA. Premium experience for sideloaders — the Calibre/r-ebooks crowd who already manage their own library and want a beautiful reader for it. Target: $12.99 one-time Android IAP through the Play Store. Free version available separately for desktop/web.

Positioning: **EPUB-only, beautifully rendered.** Not trying to compete with Kindle/Apple Books; trying to be the reader serious readers actually pick.

**Stated ambition (per user, this session):** Tomen is meant to be a competitive, polished product — "the version of those closed-community readers that's available for everyone to use." Designed with intention, not hacked together. This framing is why the reader was refactored to a dual-slot architecture this session; scale works against you on refactors later.

## Current committed state (end of April 23 2026 session)

### Service worker cache
Last shipped version: **`tomen-v1.16.0`**. Bump on every meaningful release.

### Shipped this session

**Marks v2** — fully implemented and working.
- Per-mark `⋯` menu with Edit note / Toggle bookmark / Delete
- Note editor overlay (bottom-sheet), saves to mark.note field, 5000 char cap
- Tap-highlight-in-reader opens mark menu (via `bookAreaTapped` → `openMarkMenu`)
- Dual-tree aware: marks can be created on either the left or right slot
- Key functions: `openMarkMenu`, `closeMarkMenu`, `toggleMarkBookmark`, `confirmDeleteMark`, `removeMarkAndUpdateUI`, `openNoteEditor`, `saveNoteFromEditor`

**Import fix** — folder imports skip hidden dotfolders now (Calibre's `.caltrash`, `.DS_Store`, etc). Both `collectFilesFromDir` (File System Access API path) and the `webkitdirectory` fallback. Fixed after discovering duplicates were coming from hidden Calibre trash.

**Library search** — text input in the library header, filters by title+author, substring, diacritic-insensitive. On ≤480px phones, search drops to its own row below the header. Escape clears. Last-read pin suppressed during search. Not persisted across sessions.

**Per-book typography override infrastructure** (Chunk 1 of spine work).
- New `fontFace` field on book records (null = use global default; values: `serif`/`sans`/`mono`)
- `applyReadingFace(face)` — pure DOM swap
- `setReadingFace(face)` — dispatches: if book open, writes to `currentBook.fontFace` and saves; else treats as global change
- `setGlobalReadingFace(face)` — Profile overlay's picker
- New "Default typography" section in Profile overlay
- Book-open applies override or global fallback; book-close reverts to global

**Spine view redesign** (Chunk 2) — ripped out skeuomorphism, now flat.
- No texture, no bands, no bookmark triangle, no page sliver, no inset shadows, no gradient backgrounds
- Flat solid colored rectangles with subtle inset border for seam between same-hue spines
- **Progress fill mechanic**: bottom portion of each spine fades toward paper as progress accumulates. Filling up, not draining. 0% = fully saturated; 100% = fully faded. Fill line crisp at the top of the read portion.
- **Per-book typography on spines**: whatever the book's reading face is set to, the spine title renders in that face. Color × typeface = two axes of identity.
- **Luminance-aware title color** — `titleColorForSpine(hex)` picks dark or light text based on spine color brightness. Threshold 0.55.
- Last-read indicator: single chartreuse dot at base of the spine (replaces old pinned-margin + spacer).
- Subtitles: split on `: `, rendered inline in italic at 62% of title size (so "Recursion: A Novel" reads as title + appendix, not run-on).
- Top-anchored titles by default; `.center` modifier for short titles via `titleCenter` in `spineLayout`.
- `measureVertical` / `fitVertical` / `spineLayout` thread font-family through for accurate per-book fit.

**Dual-slot reader architecture** (Phase A of reader refactor) — the big one.
- Replaced single `.r-pages` multi-column container with `.r-spread` flex row containing `.r-slot-left` + `.r-slot-right`, each with a `.r-slot-inner` that renders the full chapter.
- Left slot is authoritative for reads (selection ranges, anchor offsets, sentence-select). Right slot is a display twin. Both are interactive — users can select/highlight/tap on either side, everything resolves to shared chapter offsets.
- Left slot keeps `id="r-inner"` so existing code referencing that ID continues to work unchanged. Both slot-inners also have class `r-inner` so existing CSS rules apply to both.
- `renderChapter` writes HTML to both, applies marks to both, resets both.
- `measureSpreads` reads off the left (both identical); caches `_slotColumnWidth`, `_slotColumnGap`, `_pagesPerSpread`.
- `applySpread` translates each slot independently — left shows page `2S`, right shows page `2S+1`.
- `findSpreadForChapterOffset`, `fadeSwap`, `awaitChapterImagesThenMeasure` all updated for dual-slot.
- Sentence-select / editing-mode pointer handlers wired to both inners; `isInsideInner` + `findBlockAncestor` + `findContainingInner` generalized.
- Mobile: `.r-slot-right { display: none }` for single-page mode; `_pagesPerSpread` detects and adjusts.

**Page-turn animation** — slide-per-slot, 300ms, no fade.
- Pure transform translation of each slot-inner from current X to target X. No opacity changes, no two-phase animation, no drift.
- Both slots animate at the same instant (in sync), but each is its own independent animation — architectural foundation for future per-slot effects.
- Curve: `cubic-bezier(0.4, 0, 0.2, 1)`.
- `_animatingTurn` flag blocks re-entry; defensive timeout unlocks it after max duration as a safety net.

**Click zone / hold-ring fix** — fixed a long-standing issue where Opera's (and some other desktops') `matchMedia('(hover: hover) and (pointer: fine)')` returns false even on real mouse setups, which silently disabled the hold-to-turn IIFE. New gate: matchMedia as a hint, but falls back to observing any real non-zero `movementX/Y` mousemove and activating handlers then. Means a ~sub-second delay after page load before hold works, but then it works.

### Still waiting on user testing from this session
- Real-world stability of the dual-slot refactor across the full library
- Edge cases: very long chapters, chapter boundaries, resize during animation, rapid clicking, cross-chapter marks

## Pending phases (explicitly deferred, not urgent)

**Phase C: Lazy measurement.** Currently both slots measure the full chapter column-count up front. For 100+ page chapters this is 50-150ms. Defer measurement until visible pages need it; expand as user turns pages. Worth doing only when a specific book feels slow to open.

**Phase D: Right-slot lazy content.** Right slot currently holds full chapter HTML in memory as a display twin. Phase D: defer right-slot rendering until it's actually needed (i.e., not on mobile, not hidden). Pure memory optimization. Never user-visible unless you run out of memory.

## Known design decisions (don't re-litigate)

- **Chartreuse is the brand color** (`#C8FF00` / `#B8E635` / `#B8D800` per theme)
- **EPUB only** (no PDF in v1)
- **Philosophy B rendering** — consistent Tomen typography, we strip publisher CSS
- **Curated fonts only** — no user-supplied fonts, but per-book override lets users pick among shipped faces
- **Non-judgmental labels** — "Immersed/Steady/Brisk", not "Slow/Fast"
- **Chrome scope separation** — Top bar app-level, bottom bar book-level
- **Brand color shows up in chrome and marks, not reading content**
- **Spine colors per-book are auto-hashed, then overridden from cover extraction when available**

## Code structure

Single file, around 6800+ lines, at `index.html`. Key clusters by approximate location:
- Top: HTML structure for screens + overlays
- Middle: CSS for library, reader, overlays, themes
- Bottom: JS — state, IDB wrappers, EPUB parser + normalizer, renderer, pagination, marks, reading faces, touch handlers, helpers

Supporting files: `sw.js` (PWA caching, bump version on deploy), `manifest.json` (install metadata), `DESIGN.md` (design language), fonts folder.

## Todo list, current priority

### 1. EPUB testing
Load 5-10 diverse books from the library. Screenshot anything weird. Iterate on the normalizer (`_PARA_CLASS_HINTS`, `_HEADING_CLASS_HINTS`, `_BREAK_CLASS_HINTS`) for publisher patterns that don't render cleanly. Not a coding task — needs the user loading files and reporting.

### 2. Quick-reads sort
User wants a sort mode that surfaces books you could finish in one sitting. Open design questions: what counts as "quick" (remaining time under N min? under N pages? under N%)? Discuss before implementing.

### 3. Grouping
User mentioned wanting this but hasn't specced it. What does grouping mean — by author? by progress? user-defined collections/tags? Discuss before implementing.

### 4. Pro feature definition (non-code decision)
$12.99 IAP. What's free vs paid? Product decision everything else depends on.

### 5. Grid cover progress indicators
Small % overlay on cover thumbnails in grid view.

### 6. Landing page / marketing site
### 7. Play Store listing
### 8. Capacitor wrapping for Android
### 9. File association for .epub
### 10. Email Caterina Santullo — typeface licensing
### 11. Repo privacy decision (GitHub Pro vs public)

## Post-launch backlog
- Volume button page turning (Capacitor)
- System brightness integration (Capacitor)
- Collections / tags
- Reading stats (decide: gamification?)
- Phase C/D reader perf passes

## Explicitly deferred
- TTS
- Sync across devices
- Cloud backup
- PDF support
- Subscription model

## Working conventions

- **Push back when warranted.** Don't be a yes-machine. "This might not work because X" > "Great idea."
- **Verify before code**: grep the file before writing new functions.
- **Parse-check before shipping**: `node -e` snippet to validate `<script>` blocks.
- **Bump cache version** on every shipped change (`sw.js` CACHE_VERSION).
- **Single-file architecture stays.**
- **Scope discipline**: don't fix tangential things "while in the area" during a refactor.

## How to start the new conversation

1. Attach: `HANDOFF.md`, `index.html`, `sw.js`, `manifest.json`, `DESIGN.md`.
2. Opening message:
   > "Picking up Tomen from the April 23 session. HANDOFF.md has the state. I've been living with the dual-slot reader for a bit — let's [your goal here]."
3. Consider whether a real "do X" or a "test Y in the library and report bugs" is the right framing before starting.
