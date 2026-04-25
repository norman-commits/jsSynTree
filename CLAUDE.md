# jsSynTree — Claude Code Project File

## Project Overview

Single-file browser application for drawing syntactic trees. No build system, no dependencies, no server. Everything lives in one `.html` file. Read the current source before starting any session.

Full feature specifications are in `jsSynTree_v2_design.md`. Refer to it for detailed implementation instructions for every session below.

The canonical bracket notation format is specified in `jsSynTree_bracket_spec.md`. Any session that touches bracket import or export must implement the full spec.

---

## Working Rules

- Each session produces a **new versioned file**. Do not overwrite the previous version.
- All code is inline in the HTML file. No external scripts, no CDN imports.
- After every session, confirm: existing v1.3 JSON saves load correctly; undo/redo works for all new operations; PNG and SVG export include new visual elements; browser console is clean on load.
- New persistent data (new fields on nodes, arrows, or new object types) must be added to `getCurrentStateObject`, `restoreState`, and the undo/redo stack.
- New saved fields must have safe fallback defaults when loading older JSON files that lack them.
- Add a `version` field to the JSON save format from v1.5 onwards.

---

## To-Do List

### ✅ Session 1 — Bug Fixes → `jsSynTree_v1_4.html`

See design doc **Session 1** for full specification.

- [x] Remove small caps `[SC]...[/SC]` feature (removed entirely — users can use `[SIZE=...]` instead)
- [x] Fix circle node style intersecting daughter branches (added `_getCircleGeometry` helper; updated `getConnectionPoint`, `getEdgeIntersectionPoint`, `Connection.draw`, `distanceToPoint`, and SVG export to compute circumference intersection points)

---

### ✅ Session 2 — Bracket Format + Navigation Panel → `jsSynTree_v1_5.html`

See design doc **Session 2** for full specification.

**Part A — Bracket format upgrade (import + export)**

- [x] Upgrade `BracketParser.parse` to handle `|style` suffix (`box`, `circle`), `{features}` block (`;`-separated), `\n` escapes in labels, and `_id` suffixes (strip from display label, preserve as node ID hint)
- [x] Implement escaped characters: `\|` (literal pipe in label), `\;` (literal semicolon in feature), `\n` (newline in label)
- [x] Pass parsed style and features through `BracketParser.toAppData` so they are set on the resulting node objects
- [x] Implement `BracketExporter` module with `export(nodes, connections, movementArrows)` function:
  - Finds root nodes, recurses depth-first
  - Emits multi-line format when depth > 2 or more than 4 total nodes; single-line otherwise
  - Appends `|style` if non-default, `{features}` if non-empty, `_id` if node is referenced by a movement arrow
  - Escapes `|` and `;` in label/feature content
  - Encodes multiline labels with `\n`
  - Prepends comment: `% geometry, movement arrows, and decorations are not represented`
  - Appends disconnected nodes under `% disconnected:` comment
- [x] Add "Export Bracket" button to toolbar that calls `BracketExporter.export` and copies result to clipboard
- [x] Update the bracket import modal examples to show `|style` and `{features}` syntax
- [x] Update the formatting help text in the bracket import modal to reference the spec

**Part B — Navigation Panel**

- [x] Restructure layout into a flex row `.workspace` containing the canvas and a `.nav-panel`
- [x] Implement `BracketViewModule`: wraps `BracketExporter.export` to generate the bracket string; additionally returns a **span map** — array of `{ nodeId, start, end }` objects recording each node's character span in the output string
- [x] Render bracket string in the nav panel as `<span>` elements (one per node span) to allow per-node highlighting; plain text in between as text nodes
- [x] Sync highlighting: canvas selection → highlight corresponding `<span>`; scroll into view
- [x] Sync selection: clicking a `<span>` in the nav panel → select that node on canvas and redraw
- [x] Hide/show toggle button for the panel (`«` / `»`)
- [x] Hook `navPanel.update()` into end of `redraw()`

---

### ✅ Session 3 — Copy and Paste → `jsSynTree_v1_6.html`

See design doc **Session 3** for full specification.

- [x] Implement internal clipboard (`this.clipboard`) — not system clipboard
- [x] `Ctrl+C`: copy selected node or dominance-selected subtree (nodes + internal connections + internal movement arrows only)
- [x] `Ctrl+V`: paste with new IDs, offset coordinates (+40px), select pasted nodes as dominance group, save state
- [x] `Ctrl+X`: copy then delete
- [x] No-op gracefully when clipboard is empty

---

### ✅ Session 4 — Configurable Keyboard Shortcuts → `jsSynTree_v1_7.html`

See design doc **Session 4** for full specification.

- [x] Implement `ShortcutManager` module with action-name-keyed defaults
- [x] Refactor `handleKeyDown` to delegate to `ShortcutManager.match(event)`
- [x] Persist shortcut overrides to `localStorage` under `jsSynTree_shortcuts`
- [x] Settings modal: shortcut table with Record and Reset buttons per row; Reset All button; conflict highlighting
- [x] Add "Settings" button to toolbar

---

### ✅ Session 5 — Custom Fonts → `jsSynTree_v1_8.html`

See design doc **Session 5** for full specification.

- [x] Add font selector dropdown (curated safe-web-font list) to toolbar or Settings modal
- [x] Store selected font in `this.currentFont`; replace all hardcoded `'Arial'` references
- [x] "Load Font..." file picker: reads `.ttf/.otf/.woff/.woff2`, creates `FontFace`, adds to `document.fonts`, adds to dropdown
- [x] Warn user that loaded fonts do not persist across sessions
- [x] Apply font to nav panel display
- [x] Update SVG export to use current font

---

### ✅ Session 6 — Feature Matrix Bracket Styles → `jsSynTree_v1_9.html`

See design doc **Session 6** for full specification.

- [x] Extract bracket drawing into standalone `drawFeatureBrackets(ctx, cx, y, bw, bh, style)` function
- [x] Implement three styles: `'rounded'` (current default), `'angular'`, `'serif'`
- [x] Store selection in `this.featureBracketStyle`; persist to `localStorage`
- [x] Add style selector to Settings modal
- [x] Update SVG export to honour bracket style

---

### ✅ Session 7 — Customisable Arrows → `jsSynTree_v2_0.html`

See design doc **Session 7** for full specification.

- [x] Add new fields to `MovementArrow`: `lineStyle`, `lineShape`, `annotation`, `annotationPosition`, `annotationOffset` (with defaults; safe fallbacks in `restoreState`)
- [x] Implement `lineStyle`: `'dashed'` (default), `'dotted'`, `'solid'`
- [x] Implement `lineShape`: `'curved'` (default bezier), `'angular'` (two straight segments via control point)
- [x] Implement annotation: render text at parameterised position along curve, perpendicular offset; draggable in select mode
- [x] Contextual property panel when arrow is selected (not a modal — appears in-context, disappears on deselect)
- [x] Include all new fields in save/load and undo/redo

---

### ✅ Session 8 — Decorations → `jsSynTree_v2_1.html`

See design doc **Session 8** for full specification.

- [x] Implement `Decoration` data type with fields: `id, type, x, y, width, height, label, labelPosition, strokeStyle, strokeColour, strokeWidth, fillColour, cornerRadius`
- [x] Store in `this.decorations = []`; include in save/load and undo/redo
- [x] Add "Add Box" mode button to toolbar; click-drag to define rectangle
- [x] Render decorations as bottom layer (drawn before connections, nodes)
- [x] Select mode: click on border (not interior) to select; resize handles at corners and midpoints; drag interior to move
- [x] Label renders at specified corner; draggable and snaps to four corner positions
- [x] Contextual property panel when decoration is selected
- [x] Include decorations in SVG export (`<rect>` elements) and PNG export
- [x] Omit decorations from LaTeX export; add comment line noting the omission
- [x] Keep decoration selection logic strictly separate from node selection logic
- [x] Update `BracketExporter` to include decoration-referenced node IDs in `_id` suffixes (now that decorations exist)

---

### ✅ Session 9 — v2.2.x Bug Fixes & Semantic Classification → `jsSynTree_v2_2_2.html`

**v2.2 — Semantic export**

- [x] Add `movementType` field to `MovementArrow`: `'' | 'A' | 'A-bar' | 'Head' | 'Other'`
- [x] Add dropdown selector in arrow property panel for movement type
- [x] Persist `movementType` in save/load and undo/redo
- [x] Machine-readable data with no visual effect (for export processing)

**v2.2.1 — Canvas fix**

- [x] Fix canvas-related bug (details in commit)

**v2.2.2 — Arrow modal fix**

- [x] Fix arrow property panel UI issues

---

## File Index

| File                        | Version  | Status                                |
| --------------------------- | -------- | ------------------------------------- |
| `jsSynTree_v1_3.html`       | v1.3     | Previous working base                 |
| `jsSynTree_v1_4.html`       | v1.4     | Session 1 output                      |
| `jsSynTree_v1_5.html`       | v1.5     | Session 2 output                      |
| `jsSynTree_v1_6.html`       | v1.6     | Session 3 output                      |
| `jsSynTree_v1_7.html`       | v1.7     | Session 4 output                      |
| `jsSynTree_v1_8.html`       | v1.8     | Session 5 output                      |
| `jsSynTree_v1_9.html`       | v1.9     | Session 6 output                      |
| `jsSynTree_v2_0.html`       | v2.0     | Session 7 output                      |
| `jsSynTree_v2_1.html`       | v2.1     | Session 8 output                      |
| `jsSynTree_v2_2_2.html`     | v2.2.2   | Session 9 output (current)            |
| `jsSynTree_v2_design.md`    | —        | Full feature specification            |
| `jsSynTree_bracket_spec.md` | —        | Bracket notation format specification |
| `CLAUDE.md`                 | —        | This file                             |
