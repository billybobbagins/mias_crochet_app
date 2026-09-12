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
- [ ] Paste `firestore.rules` into the Firebase console (Firestore Database →
      Rules tab) — not auto-deployed, must be done manually once.
- [ ] Enable GitHub Pages for this repo so the app is reachable at a URL
      (Settings → Pages → deploy from `main` / root).
- [ ] Smoke-test the full login → create → paint → reload-on-another-device
      flow end to end once Pages is live.
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
- Replaced `localStorage`-only storage with Firebase (Firestore + Auth):
  added a shared-passcode login gate, moved projects/patterns to Firestore
  documents (grid data stored as a serialized JSON string per pattern),
  enabled offline persistence for resilience + automatic multi-device sync,
  added a one-time importer for pre-existing local browser data, and updated
  Backup/Restore to read/write Firestore instead of `localStorage`. Added
  `firestore.rules` (must be pasted into the Firebase console manually).
