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

## Data & storage (current state — needs work)

- All data lives in browser `localStorage`, under a single key
  (`miasCrochetApp.v1`), as one big JSON blob (`{ projects: [...] }`).
- Every mutation calls `saveDB()`, which re-serializes and writes the entire
  blob back to `localStorage`.
- Manual backup/restore exists: "Backup" downloads the whole DB as a `.json`
  file; "Restore" reads a `.json` file and **overwrites all data**.
- Known limitations (flagged by the user as an area to fix properly):
  - Single browser/device only — nothing syncs.
  - No schema versioning/migration path if the data shape changes later.
  - No autosave-conflict handling, no undo history, no partial restore.
  - A full-blob rewrite on every paint stroke could get slow on large grids.
- **This is on the near-term To Do list** — see below. No decisions made yet
  on the replacement approach (e.g. IndexedDB, structured localStorage with
  versioning, or an actual backend) — to be discussed.

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
- Tools: Paint, Bucket (flood fill), Erase.
- Active color swatch + custom color picker + click-to-select from the
  project palette.
- Pointer-based painting supporting mouse and touch, including drag-to-paint
  across multiple cells.
- "Clear Whole Grid" action (with confirmation).

### Stitch guide ("View" mode)
- Toggleable guide bar that steps through rows one at a time (Prev/Next
  buttons, arrow-key navigation).
- Highlights the current row and dims the rest of the grid.
- Shows a per-color stitch-count breakdown for the current row.
- Respects the pattern's configured row-numbering direction.

### Backup / restore
- Backup: downloads the entire `localStorage` DB as a timestamped `.json`
  file.
- Restore: uploads a `.json` file and replaces all current data after
  confirmation.

## To Do / backlog

Populated collaboratively as we go — check items off as they ship, add new
ones as they come up. Nothing here is committed to until we discuss it.

- [ ] Decide on and implement a proper storage strategy (replace/augment the
      single-blob `localStorage` approach).
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

### 2026-09-12
- Added this `CONTEXT.md` living document: functionality snapshot, storage
  notes, To Do/backlog framework, per-update checklist, and this changelog.
  No app functionality changes.
