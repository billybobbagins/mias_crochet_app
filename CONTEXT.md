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
  - **Firestore's 1MB-per-document hard limit caps grid size.** A pattern's
    entire `cells` grid is serialized into one `cellsJson` string field on
    one document — there's no chunking. Measured actual sizes: a 300×300
    grid is ~0.43MB empty / ~0.86MB fully filled (safely under the limit);
    a 500×500 grid is already ~1.19MB *empty*, over the limit. This is why
    the grid size cap is 300, not higher — going bigger would need
    splitting a pattern's cells across multiple sub-documents (chunked
    storage) rather than just raising a number. A failed save currently
    only logs to the console and shows a generic "Save failed" toast — it
    doesn't specifically detect "this pattern is too big," so a
    theoretical future bug that let rows/cols exceed 300 (e.g. a bypassed
    client-side clamp) would fail silently-ish rather than with a clear
    error, worth keeping in mind if this area changes again.

## Current functionality

### Home screen
- Grid of project cards (name, pattern count, last-updated date — no color
  preview, since projects are moving toward a yarn-based model rather than
  flat color swatches). No "Your Projects" heading above the grid anymore —
  redundant with the page itself. No hover-reveal delete icon on the card
  either — deleting a project happens from inside it (the Delete button in
  the project header), one way rather than two; cards themselves are also
  slimmer now (no more `min-height` left over from the removed
  color-swatch row).
- Create / delete projects. Delete requires confirmation.
- Header buttons: 🧶 Stash (text+emoji), then three icon-only square
  buttons — ⚙ gear (Settings), a refresh-arrows icon (re-pulls the latest
  data from Firestore), and a door-arrow icon (Log out) — grouped in a
  `.header-actions` flex container so they stay right-justified against
  the left-justified "Crochet Projects" title in one row, matching the
  in-pattern header's layout. Shortening "Yarn Stash" to "Stash" alone
  wasn't enough to keep the row from wrapping on narrow phones, so a
  `@media (max-width:420px)` rule also tightens the header's
  padding/gaps and hides the Stash button's text label (leaving just the
  🧶 emoji) below that width.
- In-app header just says "Crochet Projects" (no logo icon) so it and the
  "← All Projects" back button fit on one line on a phone. No footer
  disclaimer text anymore (removed — was leftover copy from the
  `localStorage`-only era and no longer accurate).

### Settings
New screen (`state.view='settings'`, ⚙ button on the home header) with
tabs (`SETTINGS_TABS`):
- **General**: a Light / Dark / System theme picker. Saved to
  `localStorage` (device-local, not synced — a per-viewer UI preference,
  not app data). "System" (the default) follows the OS/browser preference
  via `prefers-color-scheme`, same as before this existed; picking Light or
  Dark sets `data-theme` on `<html>` which now overrides that in the CSS.
- **Yarn Presets**: the brand/material/size/hook-size dropdown-option
  management, moved here from the Yarn Stash screen (chip list per
  category, add/remove, "Reset to defaults").
- **General → Danger Zone**: only rendered when signed in as the
  Developer account (`isDevAccount()`, compares `auth.currentUser.email`
  against the "dev" entry in `LOGIN_ACCOUNTS`) — invisible on Mia's
  account. A "Wipe all my data" button that permanently deletes every
  project/pattern/yarn/preset for *whichever account is currently signed
  in* (`wipeAllMyData()`, scoped by `auth.currentUser`'s uid the same way
  every other read/write already is — it can never touch
  a different account). Meant for clearing out test data on the Developer
  account; shows the signed-in email right on the button so it's clear
  which account is about to be wiped.

### Yarn Stash / Project Yarn / Pattern Yarn
Three-tier system for curating which colors are offered when picking a
color to paint with. Cells still just store raw hex — this is entirely a
picker/organization layer, not a grid-data change.
- **Yarn Stash**: the global, per-account catalog of every yarn the user
  owns — name + color are required, brand/material/size/recommended hook
  size are optional. Managed from its own screen (🧶 **Stash** button on
  the home header): a compact card grid (swatch, name, details if any —
  the details line is omitted entirely rather than showing a placeholder
  when a yarn has none set) sorted alphabetically by name, with a single
  **"Group / Filter"** collapsible dropdown (`renderYarnGroupFilterDropdown()`,
  `state.yarnGroupFilterOpen`, closes on an outside click the same way the
  active-yarn dropdown does) holding both controls — a **"Group by"**
  `<select>` (None/Brand/Material/Size/Hook Size) that splits the grid into
  sub-headed groups by that field (a yarn missing it groups under "Unset,"
  sorted last), and a **"Filter by"** chip section below it, one clickable
  chip per preset option; a yarn must match *every* chip you've turned on
  (`state.yarnStashFilters`, strict AND). The two are mutually exclusive —
  picking a Group by clears any active filters, and turning on a filter
  chip resets Group by to None — since running both at once wasn't needed
  and this was explicitly the simpler option. The dropdown's button label
  summarizes whichever is active ("Grouped by Material" / "2 filters").
  Whenever turning on one more chip would leave zero yarn matching, that
  chip is greyed out and inert (`yarnStashFilterWouldMatch()`) rather than
  letting you reach a dead-end "no results" state through the filters
  themselves. Click a card anywhere to edit it (no separate Edit button —
  that's the only reason to be on this screen). Delete lives inside the
  Edit Yarn modal itself (a Delete button between Cancel and Save, with a
  confirmation warning) — not on the card, so there's one deliberate path
  to delete rather than a quick hover-icon. The dropdown preset-list
  management moved to
  Settings → Yarn Presets (below).
- **Project Yarn**: a per-project *selection* from the Stash — picked when
  the project is created (alongside its name) and manageable afterward via
  a "🧶 Project Yarn" button in the project header. Unchecking a yarn there
  also removes it from any of that project's patterns that had it selected.
- **Pattern Yarn**: a per-pattern *selection from that project's Project
  Yarn* (not the whole Stash) — picked when the pattern is created
  (alongside rows/cols, defaulting to the project's full yarn list) and
  manageable afterward via "Manage Pattern Yarn" in the toolbar's
  active-yarn dropdown. That checklist's pool is always Project Yarn, so a
  yarn has to be part of the project before it can be part of a pattern.
  To bring in something new, a **"+ Add Project Yarn"** button opens a
  second checklist of Stash yarn *not yet* in the project — pick one or
  more, or use its own "+ Add New Yarn" for a brand-new one (opens the
  same full Add Yarn form) — and "Add Selected" adds them to Project Yarn
  immediately, then returns to Manage Pattern Yarn with them checked
  (merged with whatever was already checked there) so a final Save on
  that dialog puts them in Pattern Yarn too. This replaced an earlier
  "one combined list, auto-expand" simplification of the same idea (a
  single full-Stash checklist where checking a non-project yarn silently
  expanded Project Yarn) — that shortcut turned out to obscure the
  project/pattern distinction, so it's back to the more explicit two-step
  flow originally described. (`state.addProjectYarnReturn` holds Manage
  Pattern Yarn's checked state while the nested picker is open, the same
  way `state.yarnAddReturn` holds it for the Add Yarn form — a second,
  parallel single-level "modal return," since the app can nest at most
  three yarn-related modals deep: Manage Pattern Yarn → Add Project Yarn →
  Add New Yarn.)
- **Adding a new yarn from inside a picker**: every yarn checklist (Project
  Yarn, Pattern Yarn, and both the New Project/New Pattern creation modals)
  has a "+ Add New Yarn" button at the bottom instead of an inline
  name+color-only quick-add — it opens the same full Add Yarn form the
  Stash screen uses (all optional fields included), then returns you to
  the checklist you came from with the new yarn checked and everything
  else you'd already filled in or checked still intact
  (`state.yarnAddReturn`, `reopenYarnReturnModal()`).
- **Picker UI**: every one of these "checklists" is visually a pair of
  pill pools — "Selected" and "Available" — not checkboxes. Clicking a
  pill moves it between the two pools (`toggle-yarn-pill` action, plain
  DOM manipulation with no re-render). `readCheckedYarnIds(root)` is still
  the one function every save/add handler calls to read back what ended
  up in "Selected" (now scanning for `.yarn-pill.selected` instead of
  checked `<input>`s), so none of those call sites needed to change when
  this moved from checkboxes to pills.
- **Migration**: projects/patterns created before this system (which had a
  flat `project.palette` instead) are upgraded automatically the first time
  they're loaded — each old palette entry becomes its own new Stash entry,
  and the project/its patterns get `yarnIds` pointing at them. No prompt,
  nothing lost. The four original seed-palette colors specifically
  (Terracotta/Cream/Sage/Espresso) get illustrative material/size/hook
  details on migration since they were always just placeholder examples,
  never real yarn — a user's own custom colors stay detail-free rather than
  have data fabricated for them. (Known edge case, not engineered around:
  if a client goes offline mid-migration and reloads before the write
  syncs, it could create duplicate Stash entries — low-probability for a
  single-user app.)

### Projects
- Rename / delete project (both live in the project page's header, next to
  the Project Yarn button — see Yarn Stash / Project Yarn / Pattern Yarn
  above).
- A project holds one or more patterns, shown as a **card grid** (was
  tabs) — each card is a row with the name/grid-size/Pattern-Yarn-color-dots
  on the left and a small (44×44px) thumbnail render of the actual grid on
  the right (`patternThumbnailSvg()`). The thumbnail renders every cell at
  full fidelity (no downsampling — it's a lossless vector SVG scaled down
  by the browser) with `shape-rendering="crispEdges"` so adjacent same-size
  rects don't show anti-aliasing seams between them, and draws only the
  filled cells' colors — no grid lines. A dashed "+ New Pattern" tile sits
  alongside the cards, matching the Home screen's "+ New Project" tile. Switched away
  from tabs because the active tab gave no visible "you are here"
  indication until you interacted with something on the page — a plain
  named tab looks identical whether selected or not until its accent
  styling kicks in on hover/interaction, which isn't obvious on load.
- Clicking a card opens a **dedicated pattern page** (`state.view =
  'pattern'`) — everything that used to render below the tabs (pattern
  header, toolbar/guide bar, grid) now lives on its own page, with a
  "← *(project name)*" back button (`renderPatternPageView()`) in place of
  the project view's "← All Projects", returning to that project's card
  grid.

### Patterns
- Create / rename / duplicate / delete patterns within a project. Rename is
  a small pencil icon right next to the pattern name heading (same for
  project names, next to the project heading); Duplicate/Delete are icon
  buttons in the header actions row; Edit/Done Editing are the two text
  buttons there.
- Configurable grid size (1–300 rows × 1–300 cols), resizable via Pattern
  Settings (shrinking prompts a confirmation since it discards out-of-bounds
  cells). Capped at 300, not higher, because of how patterns are stored —
  see Data & storage above.
- **Export to Excel** — a download-icon button in the pattern header
  (`exportPatternToExcel()`) builds a real `.xlsx` workbook client-side via
  **ExcelJS** (loaded from a CDN on first use, not bundled up front, so
  patterns that never export it never pay the ~950KB download) and
  triggers a browser download. Layout: a yarn legend (colored swatch +
  name, one row per Pattern Yarn entry) in the leftmost two columns, then
  the actual grid starting a few columns further right at row 1 — so it
  reads as sitting in the top-right of the used area, next to the legend.
  Every grid cell gets a light grey border (so the shape reads even where
  blank) and filled cells get a solid Excel cell fill matching their yarn's
  hex exactly (`excelArgbFromHex()` — Excel fills use ARGB, so it's the hex
  with an opaque `FF` alpha prefix). Verified end-to-end with the real
  `exceljs` npm package in Node (build a workbook, write it, read it back,
  confirm fills/legend/grid position match) since this environment can't
  open the file in an actual browser or Excel.
- Cell zoom (toolbar +/− buttons, pinch-to-zoom on touch, mouse-wheel/
  trackpad over the grid on desktop, 4–44px) and a Cell Height:Width Ratio
  preset dropdown (Taller/Tall/Square/Wide/Wider) to approximate real stitch
  proportions, since crochet stitches aren't square. Every zoom method
  keeps whatever point you're zoomed in on (cursor position, pinch
  midpoint, or the viewport center for the +/− buttons) visually fixed in
  place (`zoomPatternAroundPoint()`) instead of always anchoring to the
  grid's top-left corner.
- The grid's viewing area (`.grid-scroll`) is height-capped (`70vh`)
  rather than growing without limit, so it acts as a fixed viewport you
  scroll/pan within (native browser scroll — mouse drag via the Pan tool,
  touch drag, or scrollbars) instead of pushing the whole page down as a
  pattern gets tall. New patterns (and any Pattern Settings resize) pick
  their starting cell size via `fitCellSize(cols, aspect)`, sized so the
  full pattern width fits the actual device width — no horizontal scroll
  needed at the default zoom level, for patterns that reasonably can. Row/
  column numbers thin out adaptively as cells shrink (every number, then
  every 5th, then every 10th — always still showing the first and last)
  via `labelStepForSize()`, the way a graph's axis ticks thin out when you
  zoom out.
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
- One-row toolbar above the grid: Undo/Redo · Paint/Fill/Line/Erase ·
  Zoom−/Zoom+/Pan · active-yarn button · Clear Grid (pinned to the far
  right, `margin-left:auto`, so there's a visible gap between it and the
  active-yarn button rather than sitting right next to it). Pattern
  Settings moved out of the toolbar entirely — see below — freeing it up
  to fit in one row on desktop; it still wraps to two on a narrow phone
  via the existing flex-wrap, just with more room before that happens
  than the old two-row layout had.
- Tools: Paint, Bucket (flood fill), **Line**, **Circle**, **Rectangle**,
  Erase — icon-only buttons. Line: press a start cell, drag, release on an
  end cell, and every cell along a straight line between them gets painted —
  including true diagonals, via Bresenham's line algorithm
  (`bresenhamLine()`), not just same-row/same-column. Circle: press to set
  the center, drag out to set the radius, release to fill — radius is
  measured in on-screen pixels (`circleCells()`, using the pattern's own
  `cellSize`/`aspect`) so it renders as a true circle even on non-square
  cells, not an ellipse warped by the stitch aspect ratio. Rectangle: press
  one corner, drag to the opposite corner, release to fill every cell in
  the bounding box between them (`rectangleCells()`) — deliberately filled,
  not just an outline, matching Circle's approach. All three share a
  live outline preview while dragging (`markShapePreview()`/
  `clearShapePreview()`) without touching `pattern.cells` until release,
  the same "compute now,
  commit/persist at pointerup" pattern Bucket follows, and all need a
  yarn selected, same as Paint/Bucket. **Pan** is a
  separate hand-icon toggle next to the zoom buttons rather than grouped
  with the paint tools, since it plays a different role (viewport, not
  drawing) — click it to scroll a grid wider/taller than the viewport,
  mainly for touch.
- Scroll position (pan/scroll offset within the grid) survives every
  re-render, including switching tools — previously any state change reset
  the grid's scroll to the top-left because the grid DOM was fully replaced
  each time.
- Active yarn is a single button showing the current color, with a small
  white yarn/skein icon (drop-shadowed so it stays visible against any
  background color, including the empty dashed swatch) always overlaid on
  top so the button reads as "the yarn picker" even before any color is
  set. Clicking it opens a dropdown listing the pattern's yarn ("Pattern
  Yarn") to pick from, plus a "Manage Pattern Yarn" action — see Yarn
  Stash / Project Yarn / Pattern Yarn above. The pan icon is a simple 4-way
  move/cross-arrow, not a hand — the original hand icon read oddly at this
  size.
- **Two-finger pan**: alongside pinch-to-zoom, a two-finger touch drag now
  also pans the grid — each `pointermove` shifts `.grid-scroll`'s
  `scrollLeft`/`scrollTop` by however far the two fingers' midpoint moved
  since the previous move event, before the existing pinch-zoom math (which
  re-anchors around that same midpoint) runs. The two gestures are additive
  and independent: panning works whether or not the pinch distance is also
  changing that frame, and zooming still keeps the same on-screen point
  fixed under your fingers exactly as before.
- **Default active yarn**: opening/creating/duplicating a pattern (or
  auto-selecting one when you first land on a project) sets the active
  yarn to whichever yarn is *first* in that pattern's Pattern Yarn
  (`syncActiveColorToPattern()`), instead of carrying over whatever was
  active in a previously-open pattern or falling back to the app's accent
  color. If a pattern has no Pattern Yarn selected at all, `state.
  activeColor` is `null` — the color button shows an empty dashed swatch,
  and Paint/Bucket (not Erase, which needs no color) render greyed out in
  the toolbar. They're still clickable (not natively `disabled`, so a
  click can be intercepted rather than silently doing nothing): clicking
  either one, or trying to paint/fill directly on the grid, shows a
  **toast** ("Select a yarn first") instead of switching tools or
  painting. This is the app's first toast/transient-notification UI
  (`showToast()`, `#toast-root` — a small pill that fades in/out at the
  bottom of the screen), alongside the existing blocking `alert()`/
  `confirm()` used elsewhere.
- **Pattern Settings** (renamed from "Grid Settings") is now a gear-icon
  button in the pattern header (next to Duplicate/Delete, visible in both
  Edit and View mode) that opens a real modal, rather than a toolbar
  dropdown. Still Save/Cancel-gated: opening it drafts the current
  rows/cols/numbering/ratio/direction; edits only apply to the draft. Save
  commits (running the resize-crop logic if rows/cols changed) and
  persists; Cancel discards. Simplified when it became a modal: closing it
  via the backdrop or Cancel now just discards silently, the same as every
  other modal in the app, instead of the old dropdown's discard-changes
  confirmation prompt — one less special case, consistent with how New
  Project/Add Yarn/etc. already behave.
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
- Entry point moved: a button (arrow-right icon) lives in the pattern
  header's action row (next to Duplicate/Delete), replacing the old
  bordered "Start Stitch Guide" box with subtext below the grid. It reads
  **"Start Stitch"** the first time, or **"Resume Stitch"** if
  `pattern.guidePos > 0` (i.e. you've stepped through this pattern
  before) — same condition that drives `state.guideResumed` below. Once
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
  When resuming from a non-zero position, a "Continuing from last session"
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
Reorganized 2026-09-13 (was one long chronological list) into open work up
top, grouped by kind, and a **Shipped** log at the bottom for history.

### Known bugs

- [ ] **Yarn checklist color swatches not showing** — confirmed reproducing
      after a full data wipe (so not stale cached data), shows as a thin
      horizontal line where the color square should be. Two static-code
      read-throughs of the old checkbox-based markup (`<label
      class="yarn-check-row">`) found nothing wrong, and a defensive CSS
      hardening didn't confirm-fix it either. The Pass 4 pill rework
      (below) replaced that markup entirely (`<button class="yarn-pill">`,
      a different element and flex context) — please retest against the
      new pill picker specifically, since the bug may or may not still
      reproduce against genuinely different markup. If it does, the next
      step is inspecting the actual `.yarn-pill .swatch` element in
      browser devtools (right-click → Inspect → Computed tab) since static
      code reading has hit its limit here.
- [x] **Bucket fill undo failing intermittently** — root cause found:
      bucket fill responded to `pointermove`, not just `pointerdown`, so
      any drag/tremor during a "tap" (very common on touchscreens) fired
      the fill repeatedly, each firing pushing its own undo snapshot onto
      the stack. Pressing Undo once only popped the last (usually
      redundant) one, making Undo look broken even though the stack itself
      wasn't corrupted. Fixed: bucket now only fires on `pointerdown`,
      never on `pointermove` for the rest of that gesture — a flood fill
      isn't a drag-repeatable action the way paint/erase are. Also added:
      a no-op guard (skip pushing an undo snapshot at all if the clicked
      cell is already the target color, mirroring the existing paint/erase
      guard) and a fix for a related edge case where a bucket fill
      interrupted mid-gesture by a second touch landing (pinch start)
      would leave its already-computed mutation stuck unpersisted in
      memory instead of finishing normally.

### Needs live verification (not yet confirmed working on a real device)

- [ ] Zoom +/− and Pan on a phone with a wide pattern; toolbar as two clean
      rows on a phone in landscape; Pattern Settings Save/Cancel/discard
      prompt and its undo/redo; pattern rename undo; Refresh pulling fresh
      data; each login account seeing separate projects.
- [ ] Pinch-to-zoom on an actual phone (does a 2-finger touch ever still
      trigger an accidental paint stroke on the first finger before the
      second lands?); wheel-zoom over the grid on desktop (does it still
      let you scroll the page normally everywhere else?); stitch-guide
      auto-centering and resume-position across a logout/reopen; the
      vertical stitch direction's column highlighting.
- [ ] Full Yarn Stash end-to-end flow: a pre-existing project opening
      correctly with its old colors now showing as Project Yarn; the full
      create-project → pick/add yarn → create-pattern → pick/add yarn →
      paint flow; unchecking a yarn from Project Yarn correctly clearing it
      from any pattern that had it; a Preset Options add/remove surviving a
      refresh.
- [ ] **This pass's changes**, none of which could be exercised in a real
      browser/touchscreen this session: the home header staying one row
      down to actual narrow-phone widths; the pattern-card thumbnail
      rendering crisp (no seams) and legible at 44×44px for a high-res
      grid; the Group/Filter dropdown opening/closing correctly and
      actually being mutually exclusive in practice; the active-yarn glyph
      being legible against light *and* dark yarn colors; two-finger pan
      working smoothly alongside pinch-zoom (and not fighting it); and the
      new Rectangle tool's drag/preview/commit feel on both mouse and touch.

### Planned — refinements to existing features

- [x] **Manage Pattern Yarn redesign**: now shows just the current Project
      Yarn as the pick list (not the whole Stash), with a "+ Add Project
      Yarn" button opening a picker of Stash yarn not yet in the project
      (itself with its own "+ Add New Yarn" for a brand-new one) — picking
      or creating one there adds it to Project Yarn immediately and
      returns it checked in Pattern Yarn. Walks back the "one combined
      list, auto-expand" simplification in favor of this more granular
      flow.
- [x] **Default active yarn**: opening/editing a pattern now sets the
      active yarn to whichever is first in that pattern's Pattern Yarn,
      not the app's accent color. No Pattern Yarn selected → Paint/Bucket
      render greyed out (Erase stays enabled) and clicking them, or trying
      to paint directly on the grid, shows a toast ("Select a yarn first")
      instead of doing anything. Added the app's first toast/transient-
      notification component for this (`showToast()`).
- [x] **iPhone header overflow**: Settings/Refresh/Log out are now
      icon-only square buttons (SVG gear/refresh/logout icons); Yarn Stash
      kept its text+emoji label.
- [x] **Danger Zone now only shows for the Developer account** — gated by
      `isDevAccount()`, comparing `auth.currentUser.email` against the
      "dev" entry in `LOGIN_ACCOUNTS`.
- [x] **Grid viewport/zoom rearchitecture** — implemented via a
      *conservative* approach rather than a full custom-canvas rewrite (see
      the Pass 5 changelog entry for the reasoning): `.grid-scroll` is now
      height-capped (`max-height:70vh`) instead of growing unbounded, so it
      behaves as a fixed-size viewport the grid scrolls/pans within using
      the browser's existing (already-proven) native scroll — no new pan
      mechanism. New patterns default their `cellSize` to
      `fitCellSize(cols, aspect)`, which sizes cells so the full pattern
      width fits the actual device width at creation time (was a fixed
      "assume ~420px" guess); a Pattern Settings resize does the same.
      Lowered `CELL_SIZE_MIN` from 12 to 4 so very wide/tall patterns can
      still shrink small enough to mostly or fully fit. Row/column numbers
      thin out adaptively as cells shrink (every number down to 12px cells,
      every 5th from 6-11px, every 10th below that — always still showing
      the first and last) via `labelStepForSize()`, the same way a graph's
      axis ticks thin out when you zoom out. **Not implemented**: true
      transform-based canvas panning/zooming (content scaling within a
      viewport via CSS `transform`, independent of native scroll) — the
      literal "like a map" framing. That would need custom drag-panning to
      replace native touch/mouse scroll everywhere, a much larger and
      riskier change; this conservative version produces the same
      practical outcomes (fixed viewing area, pan within it, shrink below
      screen width, adaptive labels) using mechanics that were already
      working and tested. Revisit as its own pass if the native-scroll
      version doesn't feel right in practice.
- [x] **"Start Stitch" → "Resume Stitch"** once `pattern.guidePos > 0`;
      guide bar's resumed-position note reworded to "Continuing from last
      session."
- [x] **Yarn Stash card delete** moved from the card into the Edit Yarn
      modal (a Delete button between Cancel and Save), with the same
      confirmation warning it always had.
- [x] **Home screen project cards**: removed the hover-reveal delete icon
      (rely solely on the Delete button inside the project page) and
      slimmed the card padding/height now that it's not reserving space
      for the old color-swatch row.
- [ ] Yarn Stash reorder option (today it's always alphabetical by name) —
      raised earlier, still not scoped.
- [ ] Broader "discard unsaved changes?" sweep beyond Pattern Settings —
      raised earlier, still not scoped.
- [ ] **Grid default size still a bit odd** — `fitCellSize()` gets close
      but user feedback after testing is it's "not quite right yet." Left
      as-is for now (working well enough), logged here rather than guessed
      at again blind — needs the user's specifics on what looks off before
      touching it further.
- [x] **Max grid size raised from 100 to 300** (rows and cols) — capped at
      300 rather than the originally-requested 1000 because of Firestore's
      1MB-per-document limit; see Data & storage above for the actual
      measured numbers and the user's explicit choice of this tradeoff over
      a bigger storage-architecture rework. Added a save-failure toast
      alongside this (previously silent, console-only) as a cheap safety
      net regardless of the cap.
- [x] **Export to Excel** — a new download-icon button in the pattern
      header exports the current pattern as a real `.xlsx` file (colored
      cell fills matching yarn hex, plus a yarn-name legend) using ExcelJS,
      loaded on demand from a CDN. See Patterns above for the full shape
      and layout, and the user's explicit sign-off on the legend-left/
      grid-top-right layout.
- [x] **Home header genuinely one row on mobile** — shortening "Yarn
      Stash" to "Stash" alone wasn't enough; added a `.header-actions` flex
      wrapper (title left-justified, buttons right-justified, one row) plus
      a `max-width:420px` media query that tightens padding/gaps and hides
      the Stash button's text label on very narrow phones. See Home screen
      above.
- [x] **Pattern card thumbnail redesign** — full-fidelity render (no
      downsampling), `shape-rendering="crispEdges"` to avoid seams, no grid
      lines drawn, laid out as a small (44×44px) thumbnail right-justified
      against the name/grid-size/yarn-dots on the left. See Projects above.
- [x] **Yarn Stash Group by / Filter by merged into one dropdown**,
      mutually exclusive (picking one clears the other) per explicit user
      request that this was fine to simplify. See Yarn Stash section above.
- [x] **Active-yarn button always shows a yarn/skein glyph** overlaid on
      the color swatch (white, drop-shadowed for visibility on any
      background) so it reads as "the yarn picker" even when empty. See
      Pattern editor above.
- [x] **Toolbar down to one row + Pattern Settings moved to a modal**:
      Pattern Settings is no longer a toolbar dropdown — it's a gear-icon
      button in the pattern header (next to Duplicate/Delete) that opens a
      real modal with the same fields. Clear Grid moved into the same row
      as the rest of the toolbar, pinned to the far right
      (`margin-left:auto`) so there's a gap between it and the active-yarn
      button instead of sitting flush against it. This freed up enough
      space that the toolbar is one row on desktop (was always two), still
      wrapping to two on a narrow phone via the existing flex-wrap. See the
      "Pattern editor" section above for the toolbar shape and the
      "Pattern Settings" bullet for the modal's simplified close behavior.

### Planned — new features

- [x] **Pill-based yarn selection UI**, replacing the checkbox list used by
      every yarn picker (Project Yarn, Pattern Yarn, Add Project Yarn, New
      Project, New Pattern): an "Available" pool of pills and a "Selected"
      pool — clicking a pill moves it between them, so what's chosen is
      always visibly separated from what isn't.
- [x] **Group the yarn list by category** — implemented on the Yarn Stash
      screen specifically (a "Group by" dropdown: None/Brand/Material/
      Size/Hook Size, `state.yarnStashGroupBy`, sub-headed card groups),
      not inside the pill pickers themselves — see the Pass 6 changelog
      entry for why. A yarn missing the chosen field groups under "Unset,"
      sorted last.
- [x] **Straight-line drawing tool**: a new Line tool (alongside Paint/
      Bucket/Erase) — press on a start cell, drag, release on an end cell,
      and every cell the line passes through gets painted, including true
      diagonals (`bresenhamLine()`). Shows a live preview outline while
      dragging (`updateLinePreview()`, pure DOM/CSS, not committed to
      `pattern.cells` until release) and needs a yarn selected like Paint/
      Bucket do.
- [ ] **Image upload** — user asked whether Firebase supports this: yes,
      via **Firebase Storage** (a separate product from Firestore, same
      Firebase project, would need its own SDK include and its own
      security rules file). Not wired into the app at all yet and no
      concrete use case defined (e.g. a reference photo attached to a
      project or pattern) — needs scoping before building, not committed
      to yet.
- [ ] **Two-finger pan** alongside pinch-to-zoom — shipped this pass, see
      Pattern editor above and the changelog entry below. Flagged for live
      device testing since it can't be exercised in this environment
      (a real two-finger drag on a touchscreen).
- [x] **Rectangle drawing tool**: press one corner, drag to the opposite
      corner, release to fill the bounding box between them — same
      architecture as Line/Circle (`rectangleCells()`,
      `updateRectanglePreview()`, `commitRectangle()`). Filled, not just an
      outline, for consistency with Circle.
- [ ] **Yarn yardage/length calculator** — user asked how feasible it'd be
      to estimate how much of each yarn a pattern will use, given the
      stitch type and "other variables it needs." This is a real,
      buildable feature (yardage-per-stitch is a known, publishable
      constant per stitch type + yarn weight — e.g. a single crochet in
      worsted-weight yarn uses roughly a fixed length per stitch, scaled by
      hook size/tension), **but it needs real inputs before it can be
      built**, not a guess: which stitch type(s) to support first, whether
      tension/gauge should be a user-entered override or a fixed table per
      stitch+weight, and whether "how much yarn a pattern uses" means per
      filled cell (assuming one stitch per cell, which matches how this
      app already models a graphgan) or something more granular. Logged
      here as a scoped-but-not-started feature rather than attempted blind
      this pass — worth a short follow-up conversation on the exact
      stitch-to-yardage assumptions before writing any code.
- [x] **Filter the yarn list by category** — added alongside Group by on
      the Yarn Stash screen: chip-based, multiple categories/values at
      once, strict AND, with a chip greyed out the moment turning it on
      would leave zero yarn matching (`yarnStashFilterWouldMatch()`).
- [x] **Circle drawing tool**: press to set the center, drag out to set the
      radius, live preview of the cells that would get painted, release to
      commit (`circleCells()`, `updateCirclePreview()`, `commitCircle()`) —
      alongside Paint/Bucket/Line/Erase. Radius is measured in on-screen
      pixels using the pattern's own `cellSize`/`aspect` (not raw row/col
      distance), so it renders as a true circle even when cells aren't
      square — verified in an isolated Node simulation (a 3-column drag on
      2:1-aspect cells produced a ±6-row/±3-col bounding box, i.e. equal
      120px diameters on both axes). Shares its live-preview mechanism and
      "compute now, commit at pointerup" shape with the Line tool
      (generalized `clearShapePreview()`/`markShapePreview()`, was
      Line-specific `clearLinePreview()`).
- [x] **Replaced pattern tabs with pattern cards + a dedicated per-pattern
      page** — see the Projects/Patterns sections above and the Pass 7c
      changelog entry for the full shape. The project page's header
      (Rename pencil, Project Yarn button, Delete Project) didn't need to
      move — it already lived on what's now the card-grid page, not on
      the old tab-strip.

### Shipped

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
- [x] Third UX pass: stitch direction setting (horizontal/vertical) driving
      the stitch guide's step axis, auto-centering the highlighted
      row/column in the viewport while guiding, per-pattern saved guide
      position (resume where you left off), pinch-to-zoom (touch) and
      wheel-to-zoom (desktop) on the grid, view-mode pan/scroll fix
      (previously blocked by a stray `touch-action:none`), "Start Stitch"
      moved into the pattern header actions and re-iconed, header simplified
      to "Crochet Projects" with no logo, footer disclaimer removed,
      project/pattern rename moved to a pencil icon next to each heading.
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
- [x] Yarn Stash follow-up pass: smaller click-to-edit cards (no separate
      Edit button), dropped the "No extra details yet" placeholder,
      illustrative details on the migrated seed-palette examples, "+ Add
      New Yarn" opening the full form from inside any picker (instead of a
      name+color-only inline quick-add), new Settings screen (General theme
      picker + Yarn Presets management moved there from the Stash screen),
      zoom now anchors to cursor/pinch-point/viewport-center instead of the
      grid's top-left, "Your Projects" heading removed.
- [x] Default preset cleanup (emptied speculative Brand list, deduped
      Material, fixed a Hook Size gap, added a "Reset to defaults" button).
- [x] Self-service "Wipe all my data" Danger Zone in Settings, scoped to
      whichever account is signed in — added since Claude has no direct
      Firestore access to clear test data itself.

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

### 2026-09-13 (Pass 9: grid size cap raised to 300, save-failure toast, Excel export)
- **Grid size cap**: user asked to raise the max from 100 to 1000. Before
  implementing, measured the actual serialized size of `cellsJson` at
  various sizes (see the Bash-computed table in this pass's conversation):
  a 300×300 grid stays safely under Firestore's 1MB-per-document limit even
  fully filled (~0.86MB), while a 500×500 grid is already over the limit
  *empty* (~1.19MB), and 1000×1000 would be 5-10MB — 5-10x over the limit,
  which would make saves fail (currently silently, console-only). Flagged
  this to the user with options (cap at 300, cap at 200, restructure
  storage to chunk cells across multiple documents, or cap at 300 + add a
  save-failure toast) rather than shipping something that would look like
  it worked but silently lose data on real, large patterns. User picked
  "cap at 300 + add a save-failure toast." Implemented: raised all six
  rows/cols `max`/`clamp` call sites from 100 to 300, and added a
  `showToast('Save failed — changes may not be backed up')` call to both
  `saveProjectDoc()`'s and `savePatternDoc()`'s existing `.catch()`
  handlers (previously `console.error` only, no user-facing signal at
  all).
- **Excel export**: new `exportPatternToExcel()`, wired to a download-icon
  button in the pattern header. Loads **ExcelJS** from a CDN on first use
  (`loadExcelJS()`, a plain dynamic `<script>` injection with a callback
  queue for concurrent calls — not bundled up front, so the ~950KB library
  is never fetched unless someone actually exports). Builds a workbook with
  a yarn-name legend (colored swatch cell + name) in columns A-B and the
  grid itself starting a few columns to the right at row 1, so it visually
  reads as being in the top-right of the used area next to the legend —
  matching what the user asked for and confirmed. Every grid cell gets a
  light grey border regardless of fill (so blank areas still read as part
  of the grid); filled cells get a solid Excel fill in the exact yarn hex
  (Excel fills are ARGB, so `excelArgbFromHex()` just prepends an opaque
  `FF`). Downloads via a Blob + temporary `<a download>` + object URL,
  standard browser download — not sandboxed the way an Artifact preview
  would be, so this works normally on the live site. Verified the whole
  pipeline (legend placement, grid offset, fill colors, blank-cell
  no-fill) end-to-end using the real `exceljs` npm package in Node — built
  a workbook, wrote it, read it back, and confirmed cell-by-cell — since
  this environment has no way to open the resulting file in an actual
  browser or Excel.
- Not verified live: the actual download/open-in-Excel experience on a
  real device, and the exported grid's readability/proportions when
  opened (column width `2.6` and row height `14` were chosen to look
  roughly square in Excel's default view but weren't visually confirmed).

### 2026-09-13 (Pass 8: header/thumbnail polish, Group/Filter dropdown, yarn glyph, two-finger pan, Rectangle tool)
- **Home header, genuinely one row**: shortening "Yarn Stash" to "Stash"
  in Pass 7a wasn't enough on narrow phones. Wrapped the header's button
  group in a `.header-actions` flex container (title stays left-justified
  via `justify-content:space-between` on `.app-header`, buttons stay
  grouped and right-justified) and added a `@media (max-width:420px)` rule
  that tightens `.app-header` padding/gaps, shrinks the title, and hides
  the Stash button's `<span class="btn-label">` text (leaving just 🧶) —
  the actual fix, not just shorter text.
- **Pattern card thumbnail redesign**: `patternThumbnailSvg()` no longer
  downsamples to a capped sample grid — it draws one `<rect>` per actual
  cell (a lossless vector, scaled down by the browser, so a 100×100
  pattern costs the same to describe as it always did, just renders
  smaller) and adds `shape-rendering="crispEdges"` so adjacent same-color
  rects don't show faint anti-aliased seams between them. Only filled
  cells get a rect — no grid lines drawn. `renderPatternCard()` restructured
  from a stacked layout to a row (`.pattern-card-row`): name/grid-size/
  yarn-dots in `.pattern-card-info` on the left, a small fixed 44×44px
  `.pattern-card-thumb` on the right (was `width:100%;aspect-ratio:1`).
- **Yarn Stash Group by / Filter by merged into one dropdown**: replaced
  the always-visible `<select>` + always-visible filter-chip section with
  a single collapsible `.dropdown-wrap`/`.dropdown-panel` (reusing the same
  pattern as the active-yarn dropdown), toggled by
  `state.yarnGroupFilterOpen` and closed on an outside click via the
  existing generic outside-click handler (extended to check this flag
  alongside `state.colorPickerOpen`). Made the two mutually exclusive per
  explicit user sign-off that this was fine to simplify: picking a non-
  "None" Group by clears `state.yarnStashFilters`; turning on any filter
  chip resets `state.yarnStashGroupBy` to `'none'`. `renderYarnFilterSection()`
  was folded into a new `renderYarnGroupFilterDropdown()` that renders both
  controls and a one-line summary in the toggle button's label.
- **Active-yarn glyph**: added `ICONS.yarnGlyph`, a small skein/yarn-ball
  icon rendered white with a dark `drop-shadow` (via CSS, `.color-btn
  svg`) so it stays legible over any active color, including the empty
  dashed swatch — always shown on top of the active-yarn button so its
  purpose is clear even before any yarn is picked.
- **Two-finger pan**: the existing pinch-to-zoom pointer tracking
  (`state.pinch`, `state.touchPoints`) now also records the two-finger
  midpoint each move (`state.pinch.lastMid`) and shifts `.grid-scroll`'s
  `scrollLeft`/`scrollTop` by the midpoint's on-screen delta before running
  the existing zoom-around-anchor math. The two are independent and
  additive: `zoomPatternAroundPoint()` always re-derives its scroll target
  fresh from the *current* scroll position and anchor point, so the manual
  pan adjustment and the zoom's own re-anchoring never fight each other,
  and panning still works on a move where the pinch distance doesn't
  change (no zoom that frame) since it's applied unconditionally.
- **Rectangle tool**: new tool alongside Paint/Bucket/Line/Circle/Erase —
  press one corner, drag, release on the opposite corner to fill the
  bounding box between them (`rectangleCells()`, a plain min/max row-col
  span, verified against both drag directions in an isolated Node check).
  Filled, not outlined, matching Circle's approach for consistency. Built
  by mirroring the Line/Circle architecture exactly: `state.rectStart`/
  `rectCurrent`, `updateRectanglePreview()`/`commitRectangle()` sharing the
  same `markShapePreview()`/`clearShapePreview()` live-preview mechanism,
  added to `TOOLS_NEEDING_COLOR` and the toolbar's tools array, and folded
  into the pinch-interrupts-a-drag abandonment logic (rolls back the
  pushed undo snapshot and clears `rectStart`/`rectCurrent`, same as
  Line/Circle) and the `pointerup`/`pointercancel`/`pointerleave` commit
  chain.
- **Assessed, not built**: a yarn-yardage/length calculator (estimate how
  much of each yarn a pattern uses given stitch type). Real and buildable,
  but needs the user's input on stitch type(s) to support, gauge/tension
  handling, and what "usage" should be measured against before writing any
  code — logged as a scoped-but-not-started feature rather than guessed at
  blind. See "Planned — new features" above.
- Not verified live in a browser this session — every item above needs a
  real device/touchscreen pass; see "Needs live verification" above.

### 2026-09-13 (Pass 7c: pattern cards replace tabs, dedicated pattern page)
- **New navigation level**: `state.view` gains a `'pattern'` value,
  rendered by a new `renderPatternPageView()` — a project's page
  (`renderProjectView()`) now only ever shows that project's header
  (name/rename, Project Yarn, Delete) plus a card grid of its patterns;
  everything that used to render directly below the tab strip (pattern
  header, toolbar/guide bar, grid — the existing `renderPattern()`, kept
  as-is and just called from one place now instead of inline) moved to
  its own page reached by clicking a card.
- **New pattern cards** (`renderPatternCard()`): name, `rows × cols`, small
  dots for the pattern's yarn colors, and a live SVG thumbnail of the
  actual grid (`patternThumbnailSvg()`) — downsampled to at most 24×24
  sampled cells regardless of the real pattern size (a 100×100 pattern's
  thumbnail costs the same to render as a 20×20 one), using
  `preserveAspectRatio="none"` to fill a fixed square thumbnail box
  regardless of the pattern's actual row/col ratio. A dashed "+ New
  Pattern" tile sits alongside the cards, mirroring the Home screen's
  "+ New Project" tile — same empty-state-only-when-zero-patterns
  behavior as Home has for projects.
- **New pattern-page header**: "← *(project name)*" (`renderPatternPageView`
  reuses the existing `open-project` action for its back button, which
  already resets to that project's card grid — no new action needed).
  `open-pattern`/`newPattern()`/`duplicate-pattern` now all set
  `state.view='pattern'` when they navigate into a pattern; `delete-pattern`
  sets `state.view='project'` and clears `state.patternId` instead of
  auto-selecting another pattern to fall into, since deleting can now only
  ever happen from that pattern's own page (there's nowhere else left to
  land once it's gone).
- Root cause this replaces (from the original report): the old tab strip
  gave no visible "you are here" indication until something was clicked or
  the grid was interacted with — a named tab looks the same selected or
  not until hover/interaction styling kicks in, easy to miss on first
  load.
- Removed the pattern-tabs-only `.tab-add` CSS (dead now); `.tabs`/`.tab`/
  `.tab.active` are kept since the Settings screen's General/Yarn Presets
  tab bar still uses them.
- Not verified live in a browser this session, including the thumbnail
  rendering specifically — no way to visually confirm the SVG sampling
  looks right without seeing it rendered.

### 2026-09-13 (Pass 7b: Circle tool)
- **New Circle tool**, alongside Paint/Bucket/Line/Erase: press to set the
  center, drag to set the radius, release to commit. `circleCells()`
  computes the fill using on-screen pixel distance (via the pattern's own
  `cellSize * aspect` for width, `cellSize` for height) rather than raw
  row/column distance, so the result reads as an actual circle even when
  cells aren't square — a naive row/col-distance circle would render as an
  ellipse on any pattern with a non-1:1 Cell Height:Width Ratio. Verified
  the pixel math in an isolated Node simulation (dragging 3 columns on
  2:1-aspect cells produced a ±6-row/±3-col bounding box — both axes work
  out to the same 120px diameter).
- **Generalized the Line tool's preview mechanism** for reuse: renamed
  `clearLinePreview()`/the inline pointermove marking to
  `clearShapePreview()`/`markShapePreview(cellsList)` and the CSS class
  from `.cell-line-preview` to `.cell-shape-preview`, so Circle could
  reuse the exact same "outline the cells that would be painted" behavior
  instead of duplicating it. Caught and fixed one leftover reference to
  the old `clearLinePreview()` name (in the pinch-interrupts-a-drag
  handling) that the rename would otherwise have silently broken.
- `TOOLS_NEEDING_COLOR`/`toolNeedsColor()` replaces three separate
  `tool==='paint' || tool==='bucket' || tool==='line'`-style checks that
  were already drifting apart — one now needs updating per new tool
  instead of three.
- Not verified live in a browser this session — the geometry was checked
  in isolation, not the actual drag gesture or preview rendering.

### 2026-09-13 (Pass 7a: header row fix, yarn filtering, Pattern Settings → modal, one-row toolbar)
- **Header fits one row on mobile**: shortened "Yarn Stash" to "Stash" on
  the home header button — combined with last round's icon-only Settings/
  Refresh/Log out, the row is short enough to stop wrapping on a phone.
- **Yarn Stash "Filter by"**: added alongside Group by — a chip per preset
  option, multiple selectable, strict AND across every chip turned on
  (`state.yarnStashFilters`, `yarnMatchesFilters()`). A chip that would
  leave zero yarn matching if turned on is greyed out and inert
  (`yarnStashFilterWouldMatch()`, checked against the *would-be* combined
  filter set, not just its own category) — this is also what stops
  picking two values from the same category in practice, without needing
  separate same-category-OR logic: a yarn only has one Material, so a
  second Material chip always fails the "would still match something"
  check once the first is active.
- **Pattern Settings is now a modal, not a toolbar dropdown**: a gear-icon
  button in the pattern header (`open-pattern-settings` action) opens
  `patternSettingsFieldsHtml()`'s same fields inside a real
  `openModal()` call instead of a `.dropdown-panel`. This let a chunk of
  now-unreachable code get deleted: `patternSettingsDirty()` and
  `closePatternSettingsPrompting()` existed only to intercept an
  outside-click/toggle-button close on the old dropdown and prompt to
  discard changes — impossible to trigger anymore since a real modal's
  backdrop already blocks interaction with the rest of the page while
  open. Closing via Cancel or the backdrop now just discards silently,
  matching every other modal in the app (New Project, Add Yarn, etc.)
  rather than being the one special case with a confirm-to-discard prompt.
  Also removed the `open-pattern`/`toggle-mode`/`toggle-color-picker`
  guards that used to call `closePatternSettingsPrompting()` first (same
  reason — unreachable once the settings UI can't coexist on-screen with
  those actions), and the now-dead `.dropdown-toggle`/`.dropdown-panel.right`
  CSS and unused `ICONS.chevronDown`.
- **Toolbar is one row on desktop** (was always two): Clear Grid moved
  into the main toolbar row, pinned right (`margin-left:auto`) for a
  visible gap from the active-yarn button, now that Pattern Settings no
  longer needs its own row.
- Logged (not fixed): grid default sizing still isn't quite right per
  user testing feedback — needs specifics before touching `fitCellSize()`
  again rather than guessing blind a second time.
- Not verified live in a browser this session.

### 2026-09-13 (Pass 6: yarn grouping + straight-line tool)
- **Yarn Stash "Group by"**: implemented grouping on the main Yarn Stash
  catalog screen only, not inside the pill pickers (Project Yarn, Pattern
  Yarn, etc.) — those already fully re-render on every state change
  through the normal `render()` cycle, so adding a `<select>` there was
  simple; doing the same live-regrouping *inside a modal* without a full
  re-render would have needed real new plumbing (the modals currently
  patch the DOM directly for pill toggling rather than re-rendering).
  Scoped it to the one place it was cheap to do well rather than force a
  partial version everywhere. `state.yarnStashGroupBy` (None/Brand/
  Material/Size/Hook Size); groups sorted alphabetically with "Unset"
  (yarn missing that field) always last.
- **New Line tool**: added to the toolbar's tool group (Paint/Bucket/
  Line/Erase). `bresenhamLine(r0,c0,r1,c1)` — standard Bresenham's line
  algorithm — returns every cell a straight line between two points
  passes through, diagonals included (verified against horizontal,
  vertical, 45°, shallow, steep, single-point, and reversed-direction
  cases in an isolated Node simulation, since live dragging can't be
  tested here). Follows the same "compute now, commit at pointerup"
  shape Bucket already established: `pointerdown` snapshots undo state
  and records the start cell; `pointermove` only updates a visual preview
  (`updateLinePreview()`/`clearLinePreview()`, CSS outline via
  `.cell-line-preview`, no `pattern.cells` mutation yet); `pointerup`/
  `pointercancel`/document-`pointerleave` call the new `commitLine()`,
  which paints the final path and lets the existing `endPaint()` persist
  and render it. Needs a yarn selected, same guard as Paint/Bucket.
  Also hardened the pinch-interrupts-a-gesture handling (already updated
  for bucket in Pass 1) for this third case: an in-progress Line drag
  interrupted by a second touch landing (pinch start) is abandoned
  (preview cleared, the undo snapshot pushed at its `pointerdown` popped
  back off) rather than partially persisted, since nothing was actually
  painted yet to keep.
- Not verified live in a browser this session — the Bresenham path logic
  was checked in isolation (see above), but not the actual drag gesture,
  live preview rendering, or interaction with the grid's pointer-event
  plumbing.

### 2026-09-13 (Pass 5: grid viewport/zoom rearchitecture — conservative version)
- **Design decision, made without checking back first** (auto-run pass,
  documenting the reasoning here instead): implemented the requested
  outcomes — a fixed-size viewing area, fit-to-width default, ability to
  zoom smaller than the screen, adaptive label thinning — using the
  existing native-scroll architecture rather than the literal "content
  scales/pans inside a fixed viewport like a map" framing, which would
  need custom transform-based drag-panning to replace native touch/mouse
  scroll everywhere (a much larger, riskier change touching code that
  already works and is already fairly well exercised). The chosen approach
  gets the same practical behavior with far less blast radius. Flagged in
  To Do as the "conservative version" in case the native-scroll feel
  doesn't match what was pictured, with the fuller rewrite as a named
  follow-up rather than something silently dropped.
- **`.grid-scroll` is now height-capped** (`max-height:70vh`, was
  unbounded) — acts as a fixed viewport the grid scrolls within instead of
  growing the whole page vertically as a pattern gets tall.
- **New `fitCellSize(cols, aspect)`**: sizes cells so a pattern's full
  width fits the actual device width (`window.innerWidth`), replacing the
  old flat "assume ~420px available" heuristic. Used at pattern creation
  and by Pattern Settings' resize (previously both hardcoded the same
  420px guess independently).
- **Lowered `CELL_SIZE_MIN` from 12 to 4** — needed so very wide/tall
  patterns can actually shrink small enough to fit or nearly fit, and
  applies uniformly to the fit computation, zoom buttons, wheel, and pinch
  (one shared constant, not a separate floor just for fitting).
- **Adaptive row/column label thinning**: new `labelStepForSize(px)` —
  every number shown at ≥12px cells, every 5th from 6–11px, every 10th
  below that (always still showing the first and last row/column
  regardless). Root cause of "numbers become unreadable when zoomed out"
  made concrete: a row/column's *label* box is only as wide as the
  fixed 20-30px label column, but its *height* (for a row label) or
  *width* (for a column label) matches that row/column's own cell
  size — so at a 4px cell size, a row's label is 4px tall regardless of
  the label column being 20-30px wide, nowhere near enough room for an
  11px-tall number. Thinning which numbers render (not their box size)
  is what relieves that.
- Considered and rejected centering short patterns horizontally in
  `.grid-scroll` (`display:flex;justify-content:center`) — plain
  (non-`safe`) `justify-content:center` on an overflowing flex container
  has a real cross-browser history of clipping the scrollable start edge,
  which would regress the common case (a pattern wider than the viewport)
  for a cosmetic nicety on the uncommon one (a pattern narrower than it).
  Left top/left-anchored instead.
- Not verified live in a browser this session — sanity-checked
  `fitCellSize`/`labelStepForSize`'s output numbers against a phone-width
  (390px) and desktop-width (1400px) viewport in an isolated Node
  simulation instead, since visual/gesture verification isn't possible
  here.

### 2026-09-13 (Pass 4: pill-based yarn picker)
- **Replaced the checkbox list with pills** in `yarnChecklistHtml()`:
  renders two labeled pill pools, "Selected" and "Available," instead of
  one scrollable list of `<label><input type=checkbox>...</label>` rows.
  Clicking a pill (`toggle-yarn-pill` action) moves it between the two
  pools directly in the DOM (no re-render) — toggles its `.selected`
  class, relocates the element, and swaps in/out each pool's "None"/"None
  yet" placeholder as it empties/fills.
- **`readCheckedYarnIds(root)` kept its exact signature and contract**
  (container element in, array of selected yarn ids out) — only its
  internals changed, from querying checked `<input>`s to querying
  `.yarn-pill.selected[data-yarn-id]`. Every save/add handler that calls
  it (Project Yarn, Pattern Yarn, Add Project Yarn, New Project, New
  Pattern) needed zero changes as a result.
- Removed the now-dead `.yarn-checklist`/`.yarn-check-row` CSS; added
  `.yarn-pill-picker`/`.yarn-pill-row`/`.yarn-pill` in its place.
- Not verified live in a browser this session.

### 2026-09-13 (Pass 3: Manage Pattern Yarn redesign)
- **Manage Pattern Yarn's checklist is now Project-Yarn-scoped**:
  `openPatternYarnModal()`'s pool changed from every Stash yarn to
  `db.yarnStash` filtered by `project.yarnIds` — a yarn now has to be in
  the project before it can be in a pattern, closing the gap the earlier
  "auto-expand" shortcut had opened.
- **New "+ Add Project Yarn" nested picker**: `openAddProjectYarnModal()`
  shows Stash yarn *not yet* in the project as checkboxes, plus its own
  "+ Add New Yarn" (via the existing `addYarnButtonHtml`/
  `state.yarnAddReturn` machinery, now handling a third `kind`:
  `'add-project-yarn'`) for a brand-new one. "Add Selected" adds whatever's
  checked to `project.yarnIds` immediately, then reopens Manage Pattern
  Yarn with those ids merged into whatever was already checked there
  (captured beforehand in the new `state.addProjectYarnReturn`) — so a
  final Save on the outer dialog puts them in Pattern Yarn too, matching
  "added to both Project Yarn and Pattern Yarn."
- Since Pattern Yarn's pool is now always a strict subset of Project Yarn,
  removed the `save-pattern-yarn` cascade that used to auto-expand
  `project.yarnIds` for a checked-but-not-in-project yarn — no longer
  reachable, since nothing outside the pool can get checked.
- Not verified live in a browser this session.

### 2026-09-13 (Pass 2: default active yarn + toast, header icons, dev-only Danger Zone, wording, card cleanup)
- **Default active yarn**: added `syncActiveColorToPattern()`, called
  whenever the open pattern changes (opening a tab, creating, duplicating,
  or auto-selecting the first pattern on landing in a project) — sets
  `state.activeColor` to the hex of the first entry in that pattern's
  `yarnIds`, or `null` if it has none. `state.activeColor` can now
  legitimately be `null`; audited every read of it for null-safety
  (`.toLowerCase()` calls, the color-btn's inline `background` style, the
  Pattern Yarn dropdown's "selected" check, the `save-pattern-yarn` re-pick
  logic).
- **Paint/Bucket disabled with no yarn selected**: greyed out
  (`.icon-btn-greyed`, not native `disabled` — needs to stay clickable so
  it can respond) plus a dashed empty color button (`.color-btn-empty`).
  Clicking either tool, or attempting to paint/fill directly on the grid,
  calls the new `showToast('Select a yarn first')` instead. Erase is
  unaffected (doesn't need a color).
- **New toast component**: `showToast()` + `#toast-root` — a small pill
  that fades in at the bottom of the screen and auto-dismisses after
  ~2.2s. The app's first non-blocking notification; existing
  `alert()`/`confirm()` usage elsewhere is unchanged.
- **iPhone header overflow fixed**: Settings/Refresh/Log out are now
  icon-only square buttons (new `ICONS.refresh`/`ICONS.logout` SVGs,
  reusing the existing `ICONS.gear`) instead of text+emoji buttons; Yarn
  Stash keeps its label since it's the most-used of the four.
- **Danger Zone gated to the Developer account** (`isDevAccount()`) —
  invisible when signed in as Mia.
- **Wording**: pattern header's stitch-guide button reads "Resume Stitch"
  instead of "Start Stitch" once `pattern.guidePos > 0`; the guide bar's
  resumed-position note now reads "Continuing from last session" (was
  "Resuming where you left off").
- **Yarn delete moved into the Edit Yarn modal** (a Delete button between
  Cancel and Save, same confirmation warning as before) instead of a
  hover icon on the Yarn Stash card — the card is click-anywhere-to-edit
  only now, no in-card actions.
- **Home project cards**: removed the hover-reveal delete icon (delete
  now only happens from inside the project page, one path instead of two)
  and slimmed the card's padding/`min-height`, which had been left over
  from before the color-swatch row was removed from these cards.
- Not verified live in a browser this session.

### 2026-09-13 (Pass 1: bucket-fill undo root-cause fix, swatch defensive CSS)
- **Found and fixed the real bucket-fill undo bug**: `pointermove` called
  `paintCell` for whichever tool was active, with no exception for bucket —
  so a flood fill fired again on every cell the pointer crossed during the
  same gesture, not just once at `pointerdown`. A touchscreen "tap" is
  rarely perfectly stationary, so this stacked several near-duplicate undo
  snapshots per fill in normal use; pressing Undo once only popped the
  last (usually a no-op re-fill of the same already-filled color), making
  Undo look broken. Fixed by having `pointermove` early-return for the
  bucket tool — a flood fill is a single discrete action, not something
  that should repeat across a drag the way paint/erase strokes do.
- Added a no-op guard to bucket fill itself (skip pushing an undo snapshot
  at all if the clicked cell is already the target color), mirroring the
  guard paint/erase already had.
- Hardened the pinch-interrupts-a-bucket-fill edge case: previously, a
  bucket fill interrupted mid-gesture by a second touch landing (pinch
  start) would have its already-computed cell mutation left stuck in
  memory, unpersisted and unrendered, until the pinch gesture happened to
  end. Now it's finished (persisted) immediately when the pinch begins,
  since a bucket fill's mutation + undo-push already completed atomically
  in `pointerdown` — there's nothing partial to roll back the way there is
  for an in-progress paint/erase drag (which still gets rolled back, as
  before).
- Applied a defensive CSS hardening to the yarn checklist's color swatch
  (`display:inline-block` + explicit `min-width`/`min-height` alongside
  the existing `width`/`height`) for the still-unresolved "shows as a
  horizontal line" report — a guess, not a confirmed fix, since two
  static-code read-throughs found nothing structurally wrong. See To Do.

### 2026-09-12 (Yarn Stash follow-up: cards, add-yarn flow, Settings screen, zoom anchoring)
- **Yarn Stash cards redesigned**: smaller card grid (`.yarn-card-grid`/
  `.yarn-card`), no separate Edit button — clicking anywhere on a card opens
  it for editing (the delete icon still stops propagation so it doesn't
  also trigger edit). Dropped the "No extra details yet" placeholder text;
  the details line just doesn't render when a yarn has none set.
- **Migration gives illustrative details to the seed-palette examples
  only**: `SEED_PALETTE_EXAMPLE_DETAILS` maps the four original hardcoded
  colors (Terracotta/Cream/Sage/Espresso) to plausible material/size/hook
  values when they're migrated into Stash entries, since they were always
  placeholder examples rather than real yarn. Any of a user's own custom
  palette colors still migrate with no fabricated details.
- **"+ Add New Yarn" replaces the inline quick-add**: every yarn checklist
  (Project Yarn, Pattern Yarn, New Project, New Pattern) previously had a
  bare name+color mini-form for adding a new yarn on the spot, with no way
  to set brand/material/size/hook. It's now a button that opens the same
  full Add Yarn form the Stash screen uses, then returns to the checklist
  you came from with the new yarn checked and anything else you'd already
  entered (name, rows/cols, other checked boxes) preserved
  (`state.yarnAddReturn`, `reopenYarnReturnModal()` — a lightweight
  single-level "modal return" mechanism, since the app only ever shows one
  modal at a time). `state.yarnAddReturn` is cleared on any modal
  cancel/backdrop-click so a later, unrelated yarn save can't accidentally
  trigger a stale return. Removed the now-dead `quickAddYarnRowHtml()` and
  `quick-add-yarn-to-list` action.
- **New Settings screen** (`state.view='settings'`, ⚙ button on the home
  header) with General and Yarn Presets tabs. General holds a Light/Dark/
  System theme picker, saved to `localStorage` (device-local — a per-viewer
  preference, not synced app data) and applied via a `data-theme` attribute
  on `<html>` that the CSS now checks before falling back to
  `prefers-color-scheme`. Yarn Presets is the preset-list management moved
  here from the Yarn Stash screen.
- **Zoom now anchors to a point, not the top-left corner**: factored out
  `zoomPatternAroundPoint(project, pattern, newSize, anchorX, anchorY)`,
  which keeps the content under that viewport point visually fixed by
  adjusting scroll offset after the resize (`scale = newSize/oldSize`,
  reproject the old scroll+anchor through it). Wheel-zoom anchors to the
  cursor; pinch-zoom anchors to the pinch midpoint (recomputed every
  `pointermove`, not just once at gesture start, so panning while pinching
  keeps working); the toolbar +/− buttons anchor to the viewport center.
  Root cause of the old top-left-only behavior: zoom only ever changed
  `cellSize` and re-rendered, and `render()`'s scroll-preserving logic just
  restores the same numeric `scrollLeft`/`scrollTop`, which — with the
  content now a different size — points at a different, shifted spot.
- **Removed the "Your Projects" heading** above the home screen's project
  grid — redundant with the page itself.
- Not verified live in a browser this session. One thing flagged as a
  possible false alarm rather than fixed: the user reported New Project/
  New Pattern's yarn checklist not showing color swatches (unlike the
  Project/Pattern Yarn management checklists) — both go through the same
  `yarnChecklistHtml()`, so no code-level difference was found; noted in To
  Do to recheck after a hard refresh.

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
