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
  flat color swatches). No "Your Projects" heading above the grid anymore —
  redundant with the page itself. No hover-reveal delete icon on the card
  either — deleting a project happens from inside it (the Delete button in
  the project header), one way rather than two; cards themselves are also
  slimmer now (no more `min-height` left over from the removed
  color-swatch row).
- Create / delete projects. Delete requires confirmation.
- Header buttons: 🧶 Yarn Stash (text+emoji), then three icon-only square
  buttons — ⚙ gear (Settings), a refresh-arrows icon (re-pulls the latest
  data from Firestore), and a door-arrow icon (Log out). Converted from
  text+emoji buttons since the row was overflowing the screen edge on a
  phone; Yarn Stash keeps its label since it's the most-used of the four.
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
  size are optional. Managed from its own screen (🧶 **Yarn Stash** button
  on the home header): a compact card grid (swatch, name, details if any —
  the details line is omitted entirely rather than showing a placeholder
  when a yarn has none set) sorted alphabetically by name. Click a card
  anywhere to edit it (no separate Edit button — that's the only reason to
  be on this screen). Delete lives inside the Edit Yarn modal itself (a
  Delete button between Cancel and Save, with a confirmation warning) —
  not on the card, so there's one deliberate path to delete rather than a
  quick hover-icon. The dropdown preset-list management moved to
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
- Cell zoom (toolbar +/− buttons, pinch-to-zoom on touch, mouse-wheel/
  trackpad over the grid on desktop, 12–44px) and a Cell Height:Width Ratio
  preset dropdown (Taller/Tall/Square/Wide/Wider) to approximate real stitch
  proportions, since crochet stitches aren't square. Every zoom method
  keeps whatever point you're zoomed in on (cursor position, pinch
  midpoint, or the viewport center for the +/− buttons) visually fixed in
  place (`zoomPatternAroundPoint()`) instead of always anchoring to the
  grid's top-left corner.
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
      read-throughs found nothing wrong (`.yarn-check-row .swatch` styling
      looks correct: 20×20px, `flex-shrink:0`, inside a `display:flex`
      label). Applied a defensive hardening (`display:inline-block` +
      `min-width`/`min-height` added alongside the existing `width`/
      `height`, in case some browser context wasn't sizing it as a flex
      item for an unclear reason) but this is a guess, not a confirmed
      fix — if it still reproduces, the next step is inspecting the actual
      element in browser devtools (right-click the swatch → Inspect →
      check the Computed tab for its real width/height/background) since
      static reading has hit its limit twice now.
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
- [ ] **Grid viewport/zoom rearchitecture**: right now the grid's rendered
      size just grows with zoom and the browser scrolls it — width caps at
      the screen but height keeps growing unbounded. Wanted instead: the
      grid defaults to fitting the screen's width (full pattern width
      visible, no horizontal scroll needed at 100%), with height following
      from that width and the locked cell aspect ratio. The *viewing area*
      (viewport) should then stay a fixed on-screen size, with zoom/pan
      moving and scaling the grid *within* that fixed viewport (like a
      map), rather than the viewport itself growing — and zooming out
      should be able to shrink the grid smaller than the screen width too.
      Also: when zoomed out far enough that row/column numbers would be
      unreadably small, thin them out adaptively (every 5th number, then
      every 10th, etc.) the way axis ticks scale on a graph. This is a
      genuine rendering-architecture change (fixed-size viewport with
      content that scales/pans inside it, vs. today's "container grows to
      fit content, browser scrollbars appear as needed"), not a small
      tweak — needs real design thought before touching.
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

### Planned — new features

- [ ] **Pill-based yarn selection UI**, replacing the checkbox list used by
      every yarn picker (Project Yarn, Pattern Yarn, New Project, New
      Pattern, and the redesigned Manage Pattern Yarn above): an
      "available" pool of pills and a "selected" pool: clicking a pill
      moves it from available → selected (and back), so what's chosen is
      always visibly separated from what isn't, rather than scanning a
      long checklist for checked boxes.
- [ ] **Filter/group the yarn list** by category (brand, size, etc.) in the
      pickers, with the user able to choose how the list is organized/
      displayed. Not yet scoped — depends somewhat on the pill UI above.
- [ ] **Straight-line drawing tool**: click two points on the grid and fill
      every cell between them, alongside Paint/Bucket/Erase. Should ideally
      handle diagonals (not just same-row/same-column lines) — likely a
      Bresenham-line-style algorithm to pick which cells a diagonal line
      "passes through." Not yet scoped.
- [ ] **Image upload** — user asked whether Firebase supports this: yes,
      via **Firebase Storage** (a separate product from Firestore, same
      Firebase project, would need its own SDK include and its own
      security rules file). Not wired into the app at all yet and no
      concrete use case defined (e.g. a reference photo attached to a
      project or pattern) — needs scoping before building, not committed
      to yet.

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
