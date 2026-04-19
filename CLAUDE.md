# jsSynTree — Claude Code Project File

## Project Overview

Single-file browser application for drawing syntactic trees. No build system, no dependencies, no server. Everything lives in one `.html` file. Read the current source (`jsSynTree_v1_3.html`) before starting any session.

Full feature specifications are in `jsSynTree_v2_design.md`. Refer to it for detailed implementation instructions for every session below.

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

### ✅ Session 2 — Navigation Panel → `jsSynTree_v1_5.html`
See design doc **Session 2** for full specification.
- [ ] Restructure layout into a flex row `.workspace` containing the canvas and a `.nav-panel`
- [ ] Implement `BracketViewModule`: generates bracket string + span map from current tree state; strips formatting markup for display; annotates circle/box styles; lists disconnected nodes and movement arrows as comments
- [ ] Render bracket string as `<span>` elements (one per node) to allow per-node highlighting
- [ ] Sync highlighting: canvas selection → highlight corresponding span; scroll into view
- [ ] Sync selection: clicking a span in the nav panel → select that node on canvas and redraw
- [ ] Hide/show toggle button for the panel
- [ ] Hook `navPanel.update()` into end of `redraw()`

---

### ✅ Session 3 — Copy and Paste → `jsSynTree_v1_6.html`
See design doc **Session 3** for full specification.
- [ ] Implement internal clipboard (`this.clipboard`) — not system clipboard
- [ ] `Ctrl+C`: copy selected node or dominance-selected subtree (nodes + internal connections + internal movement arrows only)
- [ ] `Ctrl+V`: paste with new IDs, offset coordinates (+40px), select pasted nodes as dominance group, save state
- [ ] `Ctrl+X`: copy then delete
- [ ] No-op gracefully when clipboard is empty

---

### ✅ Session 4 — Configurable Keyboard Shortcuts → `jsSynTree_v1_7.html`
See design doc **Session 4** for full specification.
- [ ] Implement `ShortcutManager` module with action-name-keyed defaults
- [ ] Refactor `handleKeyDown` to delegate to `ShortcutManager.match(event)`
- [ ] Persist shortcut overrides to `localStorage` under `jsSynTree_shortcuts`
- [ ] Settings modal: shortcut table with Record and Reset buttons per row; Reset All button; conflict highlighting
- [ ] Add "Settings" button to toolbar

---

### ✅ Session 5 — Custom Fonts → `jsSynTree_v1_8.html`
See design doc **Session 5** for full specification.
- [ ] Add font selector dropdown (curated safe-web-font list) to toolbar or Settings modal
- [ ] Store selected font in `this.currentFont`; replace all hardcoded `'Arial'` references
- [ ] "Load Font..." file picker: reads `.ttf/.otf/.woff/.woff2`, creates `FontFace`, adds to `document.fonts`, adds to dropdown
- [ ] Warn user that loaded fonts do not persist across sessions
- [ ] Apply font to nav panel display
- [ ] Update SVG export to use current font

---

### ✅ Session 6 — Feature Matrix Bracket Styles → `jsSynTree_v1_9.html`
See design doc **Session 6** for full specification.
- [ ] Extract bracket drawing into standalone `drawFeatureBrackets(ctx, cx, y, bw, bh, style)` function
- [ ] Implement three styles: `'rounded'` (current default), `'angular'`, `'serif'`
- [ ] Store selection in `this.featureBracketStyle`; persist to `localStorage`
- [ ] Add style selector to Settings modal
- [ ] Update SVG export to honour bracket style

---

### ✅ Session 7 — Customisable Arrows → `jsSynTree_v2_0.html`
See design doc **Session 7** for full specification.
- [ ] Add new fields to `MovementArrow`: `lineStyle`, `lineShape`, `annotation`, `annotationPosition`, `annotationOffset` (with defaults; safe fallbacks in `restoreState`)
- [ ] Implement `lineStyle`: `'dashed'` (default), `'dotted'`, `'solid'`
- [ ] Implement `lineShape`: `'curved'` (default bezier), `'angular'` (two straight segments via control point)
- [ ] Implement annotation: render text at parameterised position along curve, perpendicular offset; draggable in select mode
- [ ] Contextual property panel when arrow is selected (not a modal — appears in-context, disappears on deselect)
- [ ] Include all new fields in save/load and undo/redo

---

### ✅ Session 8 — Decorations → `jsSynTree_v2_1.html`
See design doc **Session 8** for full specification.
- [ ] Implement `Decoration` data type with fields: `id, type, x, y, width, height, label, labelPosition, strokeStyle, strokeColour, strokeWidth, fillColour, cornerRadius`
- [ ] Store in `this.decorations = []`; include in save/load and undo/redo
- [ ] Add "Add Box" mode button to toolbar; click-drag to define rectangle
- [ ] Render decorations as bottom layer (drawn before connections, nodes)
- [ ] Select mode: click on border (not interior) to select; resize handles at corners and midpoints; drag interior to move
- [ ] Label renders at specified corner; draggable and snaps to four corner positions
- [ ] Contextual property panel when decoration is selected
- [ ] Include decorations in SVG export (`<rect>` elements) and PNG export
- [ ] Omit decorations from LaTeX export; add comment line noting the omission
- [ ] Keep decoration selection logic strictly separate from node selection logic

---

## File Index

| File | Version | Status |
|------|---------|--------|
| `jsSynTree_v1_3.html` | v1.3 | Current working base |
| `jsSynTree_v2_design.md` | — | Full feature specification |
| `CLAUDE.md` | — | This file |
| `jsSynTree_v1_4.html` | v1.4 | Session 1 output |
| `jsSynTree_v1_5.html` | v1.5 | Session 2 output |
| `jsSynTree_v1_6.html` | v1.6 | Session 3 output |
| `jsSynTree_v1_7.html` | v1.7 | Session 4 output |
| `jsSynTree_v1_8.html` | v1.8 | Session 5 output |
| `jsSynTree_v1_9.html` | v1.9 | Session 6 output |
| `jsSynTree_v2_0.html` | v2.0 | Session 7 output |
| `jsSynTree_v2_1.html` | v2.1 | Session 8 output |
