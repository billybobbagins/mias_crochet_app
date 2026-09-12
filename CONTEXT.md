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

- **Auth**: two fixed login accounts (email/password under the hood),
  isolated from each other by Firestore uid — "Mia" and "Developer". The
  login screen shows a small account picker before the passcode field
  (`LOGIN_ACCOUNTS` in `index.html`); each account's data lives under its
  own `users/{uid}` subtree, enforced by `firestore.rules`. The "Developer"
  account is for testing without touching Mia's real data. Firebase persists
  the signed-in session across visits, so re-entering the passcode is only
  needed after signing out or clearing site data. Both accounts now exist:
  "mia" uses Mia's real email (`miajade.kha@gmail.com`, chosen so real
  usernames/emails stay an option later) and "dev" is the existing
  developer account — see `LOGIN_ACCOUNTS` in `index.html` for the mapping.
- **Data model** (Firestore): `users/{uid}/projects/{projectId}` holds
  project metadata (name, `yarnIds`, timestamps); each project has a
  `patterns/{patternId}` subcollection holding one document per pattern
  (rows/cols/cellSize/aspect/numbering/stitchDirection/guidePos/`yarnIds` +
  `cellsJson`, the grid serialized as a JSON string, to sidestep Firestore's
  no-nested-arrays restriction). Two more per-account collections:
  `users/{uid}/yarnStash/{yarnId}` (the global yarn catalog — see Yarn Stash
  below) and `users/{uid}/yarnPresets/lists` (a single doc holding the
  editable brand/material/size/hook-size dropdown option lists). Cells
  always store raw hex directly — the yarn system is a curation/picker
  layer on top of that, not a new storage format for the grid.
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
- **Backup/Restore removed** (2026-09-12): now that data syncs live to
  Firestore, a manual JSON backup/restore was redundant and risked
  confusion (e.g. restoring an old file over synced data). Replaced with a
  **Refresh** button on the home screen that re-pulls from Firestore, useful
  when testing sync across two devices/tabs.
- Known trade-offs to keep in mind:
  - No realtime cross-tab/cross-device push updates yet — data is fetched
    once at login and written through on change, not live-synced while two
    sessions are open simultaneously. Fine for a single person using one
    device at a time; would need `onSnapshot` listeners if that changes.
  - The fixed-passcode-per-account model is "good enough for a hobby app for
    two people," not a real multi-user auth system — don't scale this
    pattern up (e.g. self-serve signup) without redesigning auth if more
    users are ever added.

## Current functionality

### Home screen
- Grid of project cards (name, pattern count, last-updated date — no color
  preview, since projects are moving toward a yarn-based model rather than
  flat color swatches).
- Create / delete projects. Delete requires confirmation.
- Refresh button (re-pulls the latest data from Firestore) and Log out.
- In-app header just says "Crochet Projects" (no logo icon) so it and the
  "← All Projects" back button fit on one line on a phone. No footer
  disclaimer text anymore (removed — was leftover copy from the
  `localStorage`-only era and no longer accurate).

### Yarn Stash / Project Yarn / Pattern Yarn
Three-tier system for curating which colors are offered when picking a
color to paint with. Cells still just store raw hex — this is entirely a
picker/organization layer, not a grid-data change.
- **Yarn Stash**: the global, per-account catalog of every yarn the user
  owns — name + color are required, brand/material/size/recommended hook
  size are optional. Managed from its own screen (🧶 **Yarn Stash** button
  on the home header): a card grid of yarns (add/edit/delete) plus a
  **Preset Options** section below it for editing the four dropdown option
  lists those optional fields draw from (each also has an inline "add a
  custom option" while adding/editing a yarn).
- **Project Yarn**: a per-project *selection* from the Stash — picked when
  the project is created (alongside its name) and manageable afterward via
  a "🧶 Project Yarn" button in the project header. Unchecking a yarn there
  also removes it from any of that project's patterns that had it selected.
- **Pattern Yarn**: a per-pattern *selection* from the Stash — picked when
  the pattern is created (alongside rows/cols, defaulting to the project's
  full yarn list) and manageable afterward via "Manage Pattern Yarn" in the
  toolbar's active-yarn dropdown (which now only lists the pattern's
  current yarn, not the whole Stash). Checking a Stash yarn that isn't yet
  part of the project auto-adds it to Project Yarn too — a deliberate
  simplification of "pick from Project Yarn / pull from Stash / add
  brand-new" into one list with an auto-expand side effect.
- All three "add a new yarn" entry points (Stash screen, Project Yarn
  modal, Pattern Yarn modal, and both creation modals) let you type a name
  + pick a color inline; the Stash screen's Add/Edit form additionally
  offers the four optional preset dropdowns.
- **Migration**: projects/patterns created before this system (which had a
  flat `project.palette` instead) are upgraded automatically the first time
  they're loaded — each old palette entry becomes its own new Stash entry,
  and the project/its patterns get `yarnIds` pointing at them. No prompt,
  nothing lost. (Known edge case, not engineered around: if a client goes
  offline mid-migration and reloads before the write syncs, it could create
  duplicate Stash entries — low-probability for a single-user app.)

### Projects
- Rename / delete project.
- A project holds one or more patterns, shown as tabs.

### Patterns
- Create / rename / duplicate / delete patterns within a project. Rename is
  a small pencil icon right next to the pattern name heading (same for
  project names, next to the project heading); Duplicate/Delete are icon
  buttons in the header actions row; Edit/Done Editing are the two text
  buttons there.
- Configurable grid size (1–100 rows × 1–100 cols), resizable via Pattern
  Settings (shrinking prompts a confirmation since it discards out-of-bounds
  cells).
- Cell zoom (toolbar +/− buttons, 12–44px) and a Cell Height:Width Ratio
  preset dropdown (Taller/Tall/Square/Wide/Wider) to approximate real stitch
  proportions, since crochet stitches aren't square.
- Row/column numbering: direction can be flipped (top→bottom vs
  bottom→top, left→right vs right→left), and which side highlights
  odd-numbered rows/columns is configurable (matches how graphgan patterns
  are conventionally read). Highlighted label chips are neutral grey, not
  the app's orange accent, so they don't compete visually with painted yarn
  colors.
- Stitch direction (horizontal = step row-by-row, vertical = step
  column-by-column) is a Pattern Settings field that controls how the
  stitch guide steps through the pattern — see Stitch guide below.

### Pattern editor ("Edit" mode)
- Two-row toolbar above the grid: Row 1 = Undo/Redo · Paint/Fill/Erase ·
  Zoom−/Zoom+ · Pan · active-yarn button. Row 2 = Clear Grid (left) and
  Pattern Settings (right). Two explicit rows (not a single wrapping row) so
  it lays out predictably on a phone in landscape.
- Tools: Paint, Bucket (flood fill), Erase — icon-only buttons. **Pan** is a
  separate hand-icon toggle next to the zoom buttons rather than grouped
  with the paint tools, since it plays a different role (viewport, not
  drawing) — click it to scroll a grid wider/taller than the viewport,
  mainly for touch.
- Scroll position (pan/scroll offset within the grid) survives every
  re-render, including switching tools — previously any state change reset
  the grid's scroll to the top-left because the grid DOM was fully replaced
  each time.
- Active yarn is a single button showing the current color. Clicking it
  opens a dropdown listing the pattern's yarn ("Pattern Yarn") to pick from,
  plus a "Manage Pattern Yarn" action — see Yarn Stash / Project Yarn /
  Pattern Yarn above. The pan icon is a simple 4-way move/cross-arrow, not
  a hand — the original hand icon read oddly at this size.
- **Pattern Settings** (renamed from "Grid Settings") is Save/Cancel-gated:
  opening it drafts the current rows/cols/numbering/ratio; edits only apply
  to the draft. Save commits (running the resize-crop logic if rows/cols
  changed) and persists; Cancel discards. Closing it any other way (outside
  click, the toggle button, or switching pattern/mode) with unsaved changes
  prompts to discard or keep editing.
- Undo/Redo: reverts/replays paint strokes, bucket fills, Clear Grid, pattern
  renames, and Pattern Settings saves that changed rows/cols (numbering and
  aspect ratio are cosmetic display prefs, so they're intentionally excluded
  from undo). History is in-memory only, capped at 50 steps, and resets when
  you switch patterns. Bucket fill's undo/redo used to be unreliable because
  it re-rendered the DOM mid-gesture (before `pointerup`), unlike paint/erase
  — fixed by deferring bucket's render to `pointerup` like every other tool.
- Pointer-based painting supporting mouse and touch, including drag-to-paint
  across multiple cells (except when Pan is the active tool).
- "Clear Grid" action (with confirmation, undoable afterward).
- **Zoom gestures**: two-finger pinch on the grid (touch) and plain
  mouse-wheel/trackpad scroll while hovering the grid (desktop) both zoom
  cell size in/out, on top of the toolbar Zoom −/+ buttons. Wheel-over-grid
  intentionally replaces page-scroll-by-wheel there — use the scrollbars,
  trackpad drag, or the Pan tool to move around once zoomed in on desktop.
  Untested on a real device as of this writing — flag anything that feels
  off (accidental painting from a 2-finger touch, jumpy zoom, etc.).

### Stitch guide ("View" mode)
- Entry point moved: a "Start Stitch" button (arrow-right icon) now lives in
  the pattern header's action row (next to Duplicate/Delete), replacing the
  old bordered "Start Stitch Guide" box with subtext below the grid. Once
  the guide is active, the Prev/Next/Stop bar still appears below the grid
  as before.
- Steps through the pattern one row or column at a time (Prev/Next buttons,
  arrow-key navigation), depending on the pattern's Stitch direction setting
  (horizontal = rows, vertical = columns).
- Highlights the current row/column and dims the rest of the grid.
- Shows a per-yarn stitch-count breakdown for the current row/column.
- Respects the pattern's configured numbering direction.
- Auto-centers the highlighted row/column in the viewport on every Prev/Next
  step or arrow-key press.
- Resumes where you left off: your position is saved per pattern
  (`pattern.guidePos` in Firestore) every time you step, so closing the app
  mid-pattern and reopening the guide later picks back up at the same spot.
  Position resets to the start if you change the Stitch direction setting.
  When resuming from a non-zero position, a "Resuming where you left off"
  note and a "Revert to beginning" button appear in the guide bar.
- The grid can be panned/scrolled and zoomed in view mode the same as in
  edit mode (previously view mode blocked all touch panning — a real bug,
  not by design — since it always had `touch-action:none` set even outside
  the Pan tool; fixed by only disabling native touch handling in edit mode
  when a paint tool, not Pan, is selected).

### Login
- Full-screen passcode gate before the app loads, with a small account
  picker ("Mia" / "Developer") above the passcode field — each is a
  separate Firebase Auth account with its own isolated Firestore data (see
  Data & Storage above).
- "Log out" button on the home screen header.

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
- [x] Second UX pass: neutral grid-label color, scroll-position-preserving
      re-renders (fixes the Pan→Paint jump-to-top-left bug), bucket-fill
      undo/redo fix, zoom +/− replacing manual cell-size slider, Cell
      Height:Width Ratio presets, two-row toolbar, icon-only pattern header
      actions, Pattern Settings Save/Cancel with undo/redo coverage for
      resize/rename, "color"→"yarn" terminology, Backup/Restore removed in
      favor of a Refresh button, project cards drop the color-swatch row,
      two isolated login accounts (Mia / Developer).
- [x] Create the "mia" Firebase Auth account — done, using Mia's real email
      (`miajade.kha@gmail.com`).
- [ ] Real end-to-end test of this second UX pass on desktop and an actual
      phone (Claude couldn't browser-test the live login flow — no access to
      either account's real passcode). Please verify: zoom +/− and Pan on a
      phone with a wide pattern (no more jump-to-top-left switching tools),
      bucket-fill undo/redo, toolbar as two clean rows on a phone in
      landscape, Pattern Settings Save/Cancel/discard-prompt and its
      undo/redo, pattern rename undo, Refresh pulling fresh data, and each
      login account seeing separate projects.
- [x] Third UX pass: stitch direction setting (horizontal/vertical) driving
      the stitch guide's step axis, auto-centering the highlighted
      row/column in the viewport while guiding, per-pattern saved guide
      position (resume where you left off), pinch-to-zoom (touch) and
      wheel-to-zoom (desktop) on the grid, view-mode pan/scroll fix
      (previously blocked by a stray `touch-action:none`), "Start Stitch"
      moved into the pattern header actions and re-iconed, header simplified
      to "Crochet Projects" with no logo, footer disclaimer removed,
      project/pattern rename moved to a pencil icon next to each heading.
- [ ] Real end-to-end test of the third UX pass, especially the parts Claude
      could not verify live: pinch-to-zoom on an actual phone (does a
      2-finger touch ever still trigger an accidental paint stroke on the
      first finger before the second lands?), wheel-zoom over the grid on
      desktop (does it still let you scroll the page normally everywhere
      else?), stitch-guide auto-centering and resume-position across a
      logout/reopen, and the vertical stitch direction's column highlighting.
- [x] Small polish pass: icon-only Delete Project (matching pattern
      delete), replaced the odd hand pan icon with a simple 4-way
      move/cross-arrow, added a "Resuming where you left off" indicator +
      "Revert to beginning" button when the stitch guide resumes a saved
      position.
- [x] **Yarn Stash / Project Yarn / Pattern Yarn system** — built in full:
      a global yarn catalog with optional brand/material/size/hook-size
      (each with editable presets), a per-project selection ("Project
      Yarn", picked at project creation and manageable from the project
      header), and a per-pattern selection ("Pattern Yarn", picked at
      pattern creation and manageable from the toolbar), all with inline
      "add a new yarn" and automatic upgrade of old projects' flat
      palettes. See the dedicated section above for the full shape.
- [ ] Real end-to-end test of the Yarn Stash system, which Claude could not
      exercise live: a pre-existing project opening correctly with its old
      colors now showing as Project Yarn; the full create-project → pick or
      add yarn → create-pattern → pick or add yarn → paint flow; unchecking
      a yarn from Project Yarn correctly clearing it from any pattern that
      had it; a Preset Options "add"/"remove" surviving a refresh.
- [ ] Broader "discard unsaved changes?" sweep beyond Pattern Settings
      (raised alongside the original Yarn Stash notes — not yet scoped).
- [ ] (add more here as we plan upcoming work)

## Update checklist (run through this on every change we ship)

- [ ] Update **Current Functionality** above if behavior changed.
- [ ] Add a dated entry to the **Changelog** below.
- [ ] Update **To Do / backlog** (check off completed items, add newly
      discovered ones).
- [ ] Manually smoke-test in a browser: create a project, create a pattern,
      paint/fill/erase, resize the grid, run the stitch guide, refresh from
      the home screen.
- [ ] Commit with a clear message.

## Changelog

### 2026-09-12 (default preset cleanup)
- **Brand presets emptied** (`defaultYarnPresets().brand = []`) — the
  original seeded list (Red Heart, Lion Brand, etc.) was speculative and
  US/UK-centric; better to let it build up from what Mia actually adds
  rather than guess. The other three categories (material/size/hook size)
  keep sensible defaults.
- **Material presets deduplicated**: dropped `Merino Wool` (redundant with
  `Wool`) and `Acrylic/Wool Blend` / `Silk Blend` (overly specific — a
  plain `Blend` option covers mixed fibers instead).
- **Hook size gap fixed**: the list jumped from 6.5mm straight to 8.0mm,
  skipping the standard 7.0mm size — added.
- **New "Reset to defaults" button** in the Yarn Stash screen's Preset
  Options section (`reset-yarn-presets` action), since changing
  `defaultYarnPresets()` in code only affects a *never-before-seeded*
  account's presets doc — it can't retroactively fix one that already
  auto-seeded the old (weird) defaults into Firestore during earlier
  testing. This button lets that be fixed with one click instead of
  removing each stale option by hand. It fully overwrites the current
  lists (with confirmation), so don't use it if you've already added real
  customizations you want to keep.

### 2026-09-12 (Yarn Stash / Project Yarn / Pattern Yarn system)
- **New Firestore collections**: `users/{uid}/yarnStash/{yarnId}` (the
  global catalog — name, hex, optional brand/material/size/hookSize) and
  `users/{uid}/yarnPresets/lists` (a single doc with the editable
  brand/material/size/hookSize dropdown lists, seeded with crochet-standard
  defaults via `defaultYarnPresets()` the first time it's read).
- **Project and pattern docs gain `yarnIds`** (arrays of Stash ids),
  replacing `project.palette`. `serializeProject` stops writing `palette`;
  `deserializePattern` defaults a missing `yarnIds` to `null` (not `[]`) so
  migration can tell "not yet migrated" apart from "migrated, zero yarns."
- **Migration** (`migrateProjectYarn()`): runs once per project right after
  load. If the project has no `yarnIds` yet, each of its old `palette`
  entries becomes a new Stash doc and the project's `yarnIds` points at
  them. Independently (not gated behind the same check, so a pattern whose
  own write hadn't synced isn't skipped forever once its project is
  already migrated), any of that project's patterns missing `yarnIds`
  default to the project's full list.
- **Yarn Stash screen** (new `state.view='yarnstash'`, via a 🧶 button on
  the home header): a card grid of yarns (add/edit/delete, `openYarnModal`)
  plus a Preset Options section (`YARN_PRESET_CATEGORIES`) for editing the
  four dropdown lists as removable chips + an add-one-option row.
- **Project Yarn**: a "🧶 Project Yarn" button in the project header opens
  a checklist of every Stash yarn (`openProjectYarnModal` /
  `yarnChecklistHtml`) plus an inline quick-add-a-new-yarn row
  (`quickAddYarnRowHtml`); saving updates `project.yarnIds` and strips any
  now-unlisted yarn from that project's patterns. The New Project modal
  gained the same picker (optional — you can create with none checked).
- **Pattern Yarn**: the toolbar's active-yarn dropdown now lists only
  `pattern.yarnIds` (was every `project.palette` entry) plus a "Manage
  Pattern Yarn" action opening a checklist of the *entire* Stash — checking
  a yarn not yet in the project auto-adds it there too via the quick-add
  handler's `data-cascade-project`/`data-scope-project` attributes. This
  collapses the originally-described three-way choice ("pick from Project
  Yarn, or pull one in from the Stash, or add a brand-new one") into one
  list with an auto-expand side effect — approved as a deliberate
  simplification before implementing. The New Pattern modal gained the
  same picker, scoped to the project's yarn and defaulting to all-checked.
- Removed the old `select-palette-color`/`remove-palette-color`/
  `add-palette-color` actions and the now-dead "remove swatch" (`.rm`) CSS;
  the guide bar's per-color breakdown now looks up a yarn's name from
  `db.yarnStash` by hex instead of the removed `project.palette`.
- `.modal` can now scroll (`max-height:calc(100vh - 32px); overflow:auto`)
  since the new pattern/yarn-picker modals can get taller than a phone
  viewport.
- Also folded in from user feedback on the third UX pass: Delete Project
  is now an icon-only trash button (was a labeled danger button); the pan
  tool's hand icon (looked odd) is now a standard 4-way move/cross-arrow;
  the stitch guide shows a "Resuming where you left off" note and "Revert
  to beginning" button when it resumes a saved non-zero position
  (`state.guideResumed`, `guide-revert` action).
- Landed as three sequential commits (data layer + Stash screen; Project
  Yarn; Pattern Yarn/toolbar) per the approved plan, but not pushed/live
  until all three were done — the app was genuinely broken for new
  projects in the intermediate state (toolbar/guide code still read the
  no-longer-seeded `project.palette`).
- Not verified live in a browser this session — see To Do above for the
  specific end-to-end flows to check.

### 2026-09-12 (third UX pass: stitch direction, guide auto-center/resume, gesture zoom, header/rename cleanup)
- **Stitch direction**: new Pattern Settings field (Horizontal/Vertical).
  Horizontal keeps today's behavior (guide steps row-by-row); Vertical steps
  column-by-column instead, with the grid highlighting/dimming whole columns
  rather than rows. Implemented via a generalized `getGuideOrder()` (picks
  row or column order) and renamed the CSS/JS highlight classes from
  `row-dim`/`row-current` to `line-dim`/`line-current` so they can apply to
  either axis.
- **Stitch guide auto-centers**: every Prev/Next step or arrow-key press
  calls `scrollGuideIntoView()`, which finds the current highlighted label
  chip and calls `scrollIntoView({block:'center', inline:'center'})`.
- **Stitch guide resumes your position**: added `pattern.guidePos` (synced
  to Firestore). Starting the guide now resumes from the saved position
  instead of always row/column 1; every step re-saves it. Resets to 0 if you
  change the Stitch direction setting (a saved row position isn't meaningful
  once you're stepping through columns instead).
- **Pinch-to-zoom (touch) and wheel-to-zoom (desktop)**: two-finger touch on
  the grid adjusts cell size by the pinch distance ratio (tracked via
  `state.touchPoints`/`state.pinch`, since pointer events don't bundle
  multi-touch state); a plain mouse wheel or trackpad scroll while hovering
  the grid also zooms, intentionally replacing page-scroll-by-wheel there.
  If a paint stroke had already started on the first finger before a second
  finger landed, it's rolled back rather than left as a stray single-cell
  edit. This is the one part of this pass Claude could not test on a real
  device — see To Do.
- **Fixed a real (not cosmetic) bug**: view mode ("View"/stitch-guide mode)
  had `touch-action:none` on the grid unconditionally, which blocks native
  touch scrolling entirely — meaning panning never worked in view mode on
  touch, regardless of any UI. Fixed by only disabling native touch handling
  in edit mode when a paint tool (not Pan) is selected; view mode always
  allows native pan now.
- **"Start Stitch" relocated**: removed the old bordered "Start Stitch
  Guide" box and its subtext under the grid; added a "Start Stitch" button
  (new arrow-right icon, replacing the 🧶 emoji) into the pattern header's
  action row, shown only when in view mode with the guide not yet active.
  The active-guide Prev/Next/Stop bar is unchanged.
- **Rename moved next to the heading**: project and pattern rename are now
  a small inline pencil icon directly beside the `<h2>`/`<h3>` name, instead
  of a separate button grouped with Delete/Duplicate/Edit.
- **Header simplified further**: the in-app header (Home/Project views) now
  just reads "Crochet Projects" with no logo icon, so it and the "← All
  Projects" back button fit on one line on a phone. (The login screen's
  "Mia's Crochet Patterns" branding is unchanged — this only affects the
  persistent in-app header.)
- **Footer disclaimer removed** ("Your patterns are stored locally in this
  browser...") — stale copy from before the Firebase migration.
- Not verified live in a browser this session, and the gesture-handling
  code (pinch/wheel zoom) specifically has no automated test coverage —
  please exercise it for real before trusting it, especially on the phone.

### 2026-09-12 (second UX pass: zoom/pan rework, settings save/undo, terminology, multi-login, cleanup)
- **Grid label color**: highlighted row/column number chips (`.lbl-hi`) now
  use a neutral grey/white pair instead of the orange accent color, so they
  don't visually compete with painted yarn colors.
- **Scroll-position preservation**: `render()` now records and restores
  `.grid-scroll`'s scroll offset around every re-render. Root cause of the
  "switching Pan→Paint jumps the view back to the top-left" bug: `render()`
  replaces the grid's DOM wholesale on every state change (including a
  plain tool switch), and a freshly-created scroll container always starts
  at `scrollTop/scrollLeft = 0`.
- **Bucket fill undo/redo fix**: bucket fill no longer calls `render()`
  synchronously inside the `pointerdown` handler (replacing the DOM
  mid-gesture, unlike paint/erase which patch the DOM directly and only
  re-render at `pointerup`). It now computes the fill immediately but defers
  render/persist to `pointerup` via the existing `endPaint()` path, matching
  every other tool.
- **Zoom + Pan rework**: removed Pan from the Paint/Fill/Erase tool group.
  Added Zoom −/+ buttons (±4px per click, 12–44px range) that directly
  adjust cell size; Pan is now its own toggle next to them. Removed the
  manual "Cell size" slider from Pattern Settings (zoom owns this now).
  Replaced the free 50–200% "Cell width" slider with a **Cell Height:Width
  Ratio** preset dropdown (Taller 2:3 / Tall 5:6 / Square 1:1 / Wide 6:5 /
  Wider 3:2), still writing the same underlying `aspect` number.
- **Toolbar is now two explicit rows** on every screen size (not just
  mobile): row 1 = Undo/Redo · tools · zoom/pan · active yarn; row 2 = Clear
  Grid (left) / Pattern Settings (right). Avoids the awkward mid-group wraps
  a single flex-wrap row produced on a phone in landscape.
- **Pattern header actions → icons**: Rename/Duplicate/Delete are now
  icon-only buttons (new pencil/copy/trash SVGs, trash redrawn simpler than
  the old 🗑 emoji to match the others' style). "Edit Pattern"/"Done
  Editing" shortened to **"Edit"/"Done"** (kept as the only text buttons in
  that row).
- **"Grid Settings" renamed to "Pattern Settings"** throughout.
- **Pattern Settings Save/Cancel + undo integration**: opening the dropdown
  now drafts `{rows, cols, numbering, aspect}` — edits apply to the draft
  only. Save applies the draft (running the resize-crop logic if rows/cols
  changed) and persists; Cancel discards. Closing any other way (outside
  click, the toggle button, or switching pattern/edit-mode) while the draft
  differs from the live pattern prompts to discard or keep editing.
  Undo/redo snapshots widened from a bare `cells` array to
  `{name, rows, cols, cells}`, so a Pattern Settings save that resizes the
  grid, a pattern rename, and Clear Grid all now share the same single
  undo/redo history (numbering and aspect ratio stay outside undo — they're
  cosmetic, not data-destructive).
- **Home screen**: removed Backup/Restore buttons (redundant now that data
  syncs live to Firestore); added a **Refresh** button that re-pulls from
  Firestore. Project cards drop the color-swatch preview row — just name +
  pattern count + updated date now.
- **Terminology**: user-facing "color" → "yarn" on today's project palette
  feature ("Project Palette" → "Project Yarn", "Active color" → "Active
  yarn", etc.) — internal variable/field names unchanged, this is a display
  string change only ahead of the planned Yarn Stash system.
- **Multi-account login**: added a "Mia" / "Developer" picker on the login
  screen (`LOGIN_ACCOUNTS` in `index.html`), each a separate Firebase Auth
  account with data isolated by uid (already enforced by `firestore.rules`,
  no rules change needed). Requires manually creating the "mia" account in
  the Firebase console — see To Do.
- Not verified live in a browser this session — see To Do above.

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
