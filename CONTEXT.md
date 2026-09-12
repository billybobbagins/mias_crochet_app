# Mia's Crochet Patterns — Project Context

This is a **living document**. Update it every time we make a change to the app:
refresh "Current Functionality" if behavior changed, add a dated entry to the
Changelog, and keep the To Do list current (check things off, add things we
discover). Treat this file as the source of truth for "what does the app do
right now" and "what's next" — more reliable than trying to remember across
sessions.

## Project summary

A single-user web app for designing graphgan-style (pixel/corner-to-corner)
crochet patterns: charting a grid of colored squares that represent stitches,
then following the chart row-by-row while crocheting. Built for personal use
("Mia" is the only intended user), currently treated as a dev project — no
production/hosting hardening needed yet.

## Tech stack

- Single static file: `index.html` (HTML + CSS + vanilla JS, no build step,
  no npm dependencies). Google Fonts (Nunito) loaded from CDN.
- Local dev server config at `.claude/launch.json` (runs `http-server` on
  port 4173 via `npx`).
- No backend, no bundler, no framework. Everything runs client-side in the
  browser.

## Data & storage (current state)

Storage is now **Firebase** (Firestore + Authentication), replacing the old
single-`localStorage`-blob approach:

- **Auth**: one fixed, shared login account (email/password under the hood).
  Mia sees a simple "enter passcode" screen; the app signs in with a fixed
  email constant (`LOGIN_EMAIL` in `index.html`) and whatever passcode she
  types. Firebase persists the signed-in session across visits, so she only
  types it again if she signs out or clears site data.
- **Data model** (Firestore): `users/{uid}/projects/{projectId}` holds
  project metadata (name, palette, timestamps); each project has a
  `patterns/{patternId}` subcollection holding one document per pattern
  (rows/cols/cellSize/aspect/numbering + `cellsJson`, the grid serialized as
  a JSON string, to sidestep Firestore's no-nested-arrays restriction).
- **Offline persistence is enabled** (`firestoreDB.enablePersistence()`),
  so the app keeps working without a connection and syncs automatically once
  back online. This is what gives both resilience *and* multi-device sync in
  one mechanism — no separate offline-storage layer needed.
- **Security**: Firestore rules (see `firestore.rules` in repo root — must be
  pasted into the Firebase console manually, it isn't auto-deployed) lock
  all reads/writes to the one signed-in account's own `users/{uid}` subtree.
- **Legacy import**: on first successful login, if this browser still has
  old `localStorage` data (key `miasCrochetApp.v1`) and the Firestore account
  is empty, the app offers to import it rather than losing it.
- **Backup/restore** still exists, now reading/writing Firestore instead of
  `localStorage`: Backup downloads the in-memory DB as `.json`; Restore wipes
  all Firestore data for the account and re-imports from the uploaded file.
- Known trade-offs to keep in mind:
  - No realtime cross-tab/cross-device push updates yet — data is fetched
    once at login and written through on change, not live-synced while two
    sessions are open simultaneously. Fine for a single person using one
    device at a time; would need `onSnapshot` listeners if that changes.
  - The shared-passcode model is "good enough for a hobby app for one
    person," not a real multi-user auth system — don't scale this pattern up
    without redesigning auth if more users are ever added.

## Current functionality

### Home screen
- Grid of project cards (name, pattern count, last-updated date, palette
  swatch preview).
- Create / delete projects. Delete requires confirmation.
- Backup (download JSON) and Restore (upload JSON, full overwrite) buttons.

### Projects
- Each project has its own color palette: add a color (hex + name picker),
  remove a color from the palette (doesn't affect cells already painted with
  it, since cells store raw hex).
- Rename / delete project.
- A project holds one or more patterns, shown as tabs.

### Patterns
- Create / rename / duplicate / delete patterns within a project.
- Configurable grid size (1–100 rows × 1–100 cols), resizable after creation
  (shrinking prompts a confirmation since it discards out-of-bounds cells).
- Adjustable cell size (px) and cell aspect ratio (to approximate real
  stitch proportions, since crochet stitches aren't square).
- Row/column numbering: direction can be flipped (top→bottom vs
  bottom→top, left→right vs right→left), and which side highlights
  odd-numbered rows/columns is configurable (matches how graphgan patterns
  are conventionally read).

### Pattern editor ("Edit" mode)
- Horizontal toolbar above the grid (replaces the old side-by-side sidebar,
  which used to get pushed below the grid on anything but very wide screens):
  Undo/Redo, Tools, active color, Clear Grid, and a Grid Settings dropdown,
  left to right.
- Tools: Paint, Bucket (flood fill), Erase, and **Pan** — icon-only buttons
  (simple monochrome inline-SVG icons, no emoji). Pan lets you drag to scroll
  a grid that's wider/taller than the viewport (mainly for touch — dragging
  with Paint/Fill/Erase active always paints instead of scrolling, since
  touch has no separate "scroll" gesture available while a paint tool is
  selected).
- Active color is a single button showing the current color. Clicking it
  opens a dropdown with the project's palette ("yarn I actually have for this
  project") to pick from, plus an "add a new color" mini-form (color wheel +
  name) for adding to the palette mid-project — still available, just moved
  off the main toolbar into this dropdown.
- Grid Settings (rows/cols/resize, row/column numbering, odd-highlight side,
  cell size, aspect ratio) lives behind a dropdown toggle instead of always
  being visible.
- Undo/Redo: reverts/replays paint strokes, bucket fills, and Clear Grid (not
  grid resize — that already has its own confirmation dialog). History is
  in-memory only, capped at 50 steps, and resets when you switch patterns.
- Pointer-based painting supporting mouse and touch, including drag-to-paint
  across multiple cells (except when Pan is the active tool).
- "Clear Grid" action (with confirmation, now undoable afterward).

### Stitch guide ("View" mode)
- Toggleable guide bar that steps through rows one at a time (Prev/Next
  buttons, arrow-key navigation).
- Highlights the current row and dims the rest of the grid.
- Shows a per-color stitch-count breakdown for the current row.
- Respects the pattern's configured row-numbering direction.

### Login
- Simple full-screen passcode gate before the app loads (backed by Firebase
  Auth, one shared account — see Data & Storage above).
- "Log out" button on the home screen header.

### Backup / restore
- Backup: downloads the whole Firestore-backed DB as a timestamped `.json`
  file.
- Restore: uploads a `.json` file, wipes existing Firestore data for the
  account, and re-imports from the file (after confirmation).

## To Do / backlog

Populated collaboratively as we go — check items off as they ship, add new
ones as they come up. Nothing here is committed to until we discuss it.

- [x] Decide on and implement a proper storage strategy — done via Firebase
      (Firestore + Auth), see Data & Storage above.
- [x] Paste `firestore.rules` into the Firebase console — done.
- [x] Enable GitHub Pages for this repo — done, app is live there.
- [x] Widen desktop layout, replace sidebar with a top toolbar, simplify tool
      icons, move palette into an active-color dropdown, add undo/redo, add
      a Pan tool for mobile scrolling, fix intermittently-vanishing grid
      lines, neutral-tone the grid area, slim the header.
- [ ] Real end-to-end test of this UX pass on desktop and an actual phone
      (Claude couldn't browser-test it live — no Chrome tool available this
      session, and no access to the real passcode). Please verify: toolbar
      dropdowns (color + grid settings), undo/redo across paint/fill/clear,
      Pan-tool scrolling on a phone with a wide pattern, and that grid lines
      no longer vanish anywhere on a large grid.
- [ ] (add more here as we plan upcoming work)

## Update checklist (run through this on every change we ship)

- [ ] Update **Current Functionality** above if behavior changed.
- [ ] Add a dated entry to the **Changelog** below.
- [ ] Update **To Do / backlog** (check off completed items, add newly
      discovered ones).
- [ ] Manually smoke-test in a browser: create a project, create a pattern,
      paint/fill/erase, resize the grid, run the stitch guide, backup and
      restore.
- [ ] Commit with a clear message.

## Changelog

### 2026-09-12 (pattern editor UX pass)
- Widened the desktop layout (`main` max-width 1200px→1600px) and slimmed the
  header (smaller logo/title, dropped the subtitle).
- Replaced the side-by-side Tools/Palette/Grid-Settings sidebar with a
  horizontal toolbar above the grid, so the grid always gets full width
  instead of being pushed below the sidebar on narrower screens.
- Replaced emoji tool buttons with simple monochrome inline-SVG icons; added
  a 4th tool, **Pan**, so touch users can scroll a pattern wider than their
  screen (dragging with a paint tool always paints, so a separate scroll
  mode was needed).
- Active color is now a single button; clicking it opens a dropdown holding
  the project's yarn palette plus the "add a new color" wheel+name form
  (previously always-visible panels).
- Grid Settings (resize, numbering, cell size, aspect) moved behind a
  dropdown toggle instead of always being visible.
- Added Undo/Redo for paint strokes, bucket fills, and Clear Grid (not grid
  resize), 50-step in-memory history, reset on pattern switch.
- Fixed grid lines intermittently vanishing in a periodic pattern: the grid
  used to draw its lines via a CSS Grid `gap` + background-color trick, which
  is subject to sub-pixel rounding errors under fractional display scaling;
  replaced with real borders on every cell so each line renders independent
  of its neighbors.
- Re-themed the grid area (blank-cell checker, grid lines, row/column label
  chips) to neutral white/gray instead of the app's warm cream tones, so it
  no longer tints how painted yarn colors read. Rest of the app keeps its
  orange palette.
- Not verified live in a browser this session (no Chrome tool available, and
  Claude doesn't have the login passcode) — see To Do above for what to check.

### 2026-09-12
- Added this `CONTEXT.md` living document: functionality snapshot, storage
  notes, To Do/backlog framework, per-update checklist, and this changelog.
  No app functionality changes.
- Replaced `localStorage`-only storage with Firebase (Firestore + Auth):
  added a shared-passcode login gate, moved projects/patterns to Firestore
  documents (grid data stored as a serialized JSON string per pattern),
  enabled offline persistence for resilience + automatic multi-device sync,
  added a one-time importer for pre-existing local browser data, and updated
  Backup/Restore to read/write Firestore instead of `localStorage`. Added
  `firestore.rules` (must be pasted into the Firebase console manually).
