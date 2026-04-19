# jsSynTree v2 — Feature Development Design Document

## Overview

This document specifies incremental feature development for jsSynTree, a browser-based syntactic tree drawing tool. All development is in a single self-contained HTML file. Each session is independently shippable and builds on the previous version. Sessions are ordered to front-load bug fixes and foundational architecture before more complex features.

The working codebase is `jsSynTree_v1_3.html`. Each session should produce a new versioned file. The canonical bracket notation format is defined in `jsSynTree_bracket_spec.md`; this document cross-references it but does not duplicate it.

---

## Session 1 — Bug Fixes (v1.4)

**Goal:** Resolve two existing defects before building further. No new features.

### Bug 1: Small Caps Rendering

**Problem:** The `[SC]...[/SC]` markup tag is parsed correctly but small caps do not render visually on the canvas. The `_applyTextStyles` method uppercases the text, but `font-variant: small-caps` cannot be set on a canvas context — it is a CSS property only. The result is full uppercase rather than true small caps.

**Fix:** Remove the `[SC]...[/SC]` feature entirely. Users can approximate small caps using `[SIZE=12]` for the reduced-size portion if needed. Remove the pattern from `TextFormatter.parseFormattedText`, remove the SVG export handling, and remove any UI references.

### Bug 2: Circle Node Style Intersects Daughter Branches

**Problem:** When a node has `style === 'circle'`, the circle is sized and positioned relative to the node's text bounding box. The branch lines from parent to child use `getConnectionPoint`, which returns `bounds.bottom` for the parent. For a circle node, `bounds.bottom` is the bottom of the text bounding box, not the bottom of the circle, so the line visually starts inside or behind the circle rather than at its edge.

**Fix:** Add a `_getCircleGeometry(ctx)` helper method to `Node` that returns `{ centerX, centerY, radius }`. Override `getConnectionPoint` behaviour for circle-style nodes: compute the angle from circle centre to the target and return the circumference point. Add optional `targetX, targetY` arguments to `getConnectionPoint`; for non-circle nodes these are ignored. Update `ConnectionModule.Connection.draw`, `MovementArrowModule`, and SVG export to pass counterpart node coordinates.

---

## Session 2 — Bracket Format + Navigation Panel (v1.5)

**Goal:** Two tightly related deliverables: (1) a full implementation of the bracket notation format defined in `jsSynTree_bracket_spec.md`, covering both import (upgraded parser) and export (new exporter); (2) a collapsible navigation panel displaying the live bracket string with bidirectional selection sync. The nav panel depends on the exporter, so Part A must be implemented first.

### Part A — Bracket Format Upgrade

#### Overview

The existing `BracketParser` handles only basic topology (labels and nesting). The spec adds three extensions that must be parsed on import and emitted on export:

- **Node style:** `[NP|circle]`, `[T|box]` — the `|style` suffix after the label
- **Feature matrix:** `[NP {uD; +past; ~EPP~}]` — a `{...}` block after the label/style, features separated by `;`
- **Multiline labels:** `[**DP**\n*i*]` — `\n` within the label string encodes a real newline character
- **Node IDs:** `[TP_A]` — a `_id` suffix on the label, stripped from the display label, used to cross-reference movement arrows and (future) decorations

The full grammar is in `jsSynTree_bracket_spec.md`. Implement exactly that grammar; do not invent variations.

#### Upgraded `BracketParser.parse`

Rewrite the tokeniser/parser to handle the extended grammar. The parse function must:

1. Accept `[` or `(` as bracket delimiters (existing behaviour).
2. After reading the label token (characters up to the first whitespace, `[`, `]`, `|`, or `{`):
   a. If the next character is `|`, read the style token (`box` or `circle`; empty string treated as default/none).
   b. If the next character is `{`, read the features block up to the matching `}`, then split on unescaped `;` to produce a features array.
3. Strip any trailing `_[A-Za-z0-9]+` from the label and store it separately as `node._bracketId`. The display label is the label without the `_id` suffix.
4. Decode escape sequences in label and feature strings: `\n` → newline character, `\|` → `|`, `\;` → `;`.
5. Continue reading children as before.

The `toAppData` function must be updated to pass `style` and `features` through to the created node objects, not just `text`.

#### `BracketExporter` Module

Add a new `BracketExporter` module alongside `BracketParser`. Its primary function is `BracketExporter.export(nodes, connections, movementArrows)`, which returns a string.

**Algorithm:**

1. Find root nodes (nodes with no incoming connections in `connections`).
2. Build a depth map by traversing from each root (BFS or DFS).
3. Determine whether to use multi-line or single-line format: multi-line if tree depth > 2 or total reachable node count > 4.
4. Determine which node IDs to include: a node gets a `_id` suffix if it is referenced as source or destination by any movement arrow (pass `movementArrows` for this check; in Session 8 decorations will also trigger this).
5. Recurse depth-first to build the bracket string for each root tree:
   - Label: encode newlines as `\n`, escape `|` as `\|`
   - Append `_id` suffix if required
   - Append `|style` if style is not `'none'`
   - Append `{features}` if features array is non-empty: join features with `; `, escaping `;` within individual features as `\;`
   - Append children (recursively), indented in multi-line format
6. Prepend a comment line: `% geometry, movement arrows, and decorations are not represented`
7. Append disconnected nodes (nodes with neither incoming nor outgoing connections) after a `% disconnected:` comment, one bracket expression per line.

**Escaping rules** (applied to both labels and individual feature strings):
- `\n` in the stored text → `\n` in the bracket string (literal backslash-n, not a newline character)
- `|` → `\|`
- `;` in feature strings → `\;`

**Multi-line indentation:** use two spaces per depth level.

**Node ID assignment for export:** use a short alphabetical ID derived from position (A, B, C… Z, AA, AB…), not the internal UUID, to keep the bracket string human-readable.

#### Toolbar Button

Add an "Export Bracket" button to the toolbar (adjacent to the existing "From Bracket" button). Clicking it calls `BracketExporter.export` on the current state and copies the result to the clipboard, with a notification alert. If no nodes exist, alert the user that the canvas is empty.

#### Import Modal Updates

Update the bracket import modal:
- Add `|circle`, `|box`, and `{features}` to the examples section.
- Update the help text to explain the `|` style suffix and `{...}` feature block syntax.
- The existing spacing inputs (`hSpacing`, `vSpacing`) are unchanged.

---

### Part B — Navigation Panel

**Goal:** A collapsible read-only panel on the right side of the canvas displaying the live bracket string, with node highlighting synchronised to canvas selection.

#### Layout Changes

Introduce a flex row wrapper around the canvas:

```
.container
  .toolbar          (unchanged)
  .info             (unchanged)
  .workspace        (new flex row, align-items: stretch)
    canvas          (flex-grow: 1, min-width: 0)
    .nav-panel      (fixed width 280px, collapsible)
```

When the nav panel is hidden, the canvas expands to fill the space. The toggle button (`«` / `»`) sits at the top-right corner of the panel (or at the right edge of the canvas when collapsed).

#### `BracketViewModule`

Add a `BracketViewModule` with an `update(nodes, connections, movementArrows, selectedNodeId)` method. This module:

1. Calls `BracketExporter.export(nodes, connections, movementArrows)` to get the bracket string.
2. Builds a **span map**: for each node, records `{ nodeId, start, end }` character positions in the generated string. This requires the exporter to return span information alongside the string — add a second return value or make the exporter return `{ text, spans }`.
3. Renders the bracket string into the nav panel DOM as a sequence of `<span data-node-id="...">` elements for node spans, with plain text nodes for characters between spans.
4. Applies a highlight class to the span matching `selectedNodeId` (if any), and scrolls it into view.
5. Attaches click handlers to each `<span>` that call back into the app to select that node and redraw the canvas.

**Span map construction:** The exporter must track character positions as it builds the string. The simplest approach is to have the recursive node serialiser record `{ nodeId, start, end }` into an accumulating array, where `start` is the index of the `[` and `end` is the index after the closing `]` for the full subtree (including children). This means spans are nested (parent spans contain child spans), which is intentional — clicking anywhere in a subtree that is not covered by a more specific inner span selects the parent. In the DOM, render only leaf-level spans; clicking propagates upward in the tree to find the nearest `data-node-id`.

Alternatively, and more robustly: render each node as a `<span>` covering only its own label/style/features token (not its children's brackets). This avoids nesting issues entirely and is the recommended approach.

#### Nav Panel DOM Structure

```html
<div class="nav-panel" id="navPanel">
  <div class="nav-panel-header">
    <span>Tree</span>
    <button id="navPanelToggle">«</button>
  </div>
  <div class="nav-panel-content" id="navPanelContent">
    <!-- bracket string rendered here as mixed span/text nodes -->
  </div>
</div>
```

Style: `font-family: monospace; font-size: 13px; white-space: pre; overflow: auto; padding: 10px;`

Highlighted node: `.nav-highlight { background: #fff3cd; border-radius: 2px; }` (warm yellow, distinct from the canvas blue selection glow).

Disconnected nodes and comment lines render as plain text (no span, not clickable).

#### Update Trigger

At the end of `SyntacticTreeApp.redraw()`, call:

```javascript
this.navPanel.update(this.nodes, this.connections, this.movementArrows, 
                     this.selectedNode ? this.selectedNode.id : null);
```

This means the nav panel regenerates on every redraw. Since `BracketExporter.export` traverses the graph, it is O(n) in node count — acceptable for trees of the sizes this tool targets. No caching is needed at this stage.

#### Interaction

- **Canvas → panel:** When a node is selected on canvas (by click in select mode, or by dominance selection), the corresponding span is highlighted and scrolled into view in the nav panel.
- **Panel → canvas:** Clicking a node span sets `this.selectedNode` to the corresponding node and calls `this.redraw()`. The mode does not change (the user stays in whatever mode they were in). This is a selection-only interaction; it does not open the edit modal.
- **Deselection:** When `selectedNode` is null, no span is highlighted.

---

## Session 3 — Copy and Paste (v1.6)

**Goal:** Allow nodes (and optionally subtrees) to be copied and pasted onto the canvas.

### Scope

Copy/paste operates on the canvas, not on the bracket string (the nav panel is read-only). The clipboard target is the app's internal state, not the system clipboard.

### Copy

When one or more nodes are selected (either a single node or a dominance-selected subtree):
- `Ctrl+C` serialises the selected nodes and any connections *between* them (not connections to unselected nodes) into `this.clipboard`.
- Structure: `{ nodes: [...], connections: [...], movementArrows: [...] }`, same shape as `getCurrentStateObject`.
- Features, styles, and text are all preserved.

### Paste

`Ctrl+V`:
- Assigns new IDs to all pasted nodes and connections.
- Offsets all pasted node coordinates by +40px x, +40px y; increment offset on repeated paste.
- Selects all pasted nodes as a dominance group immediately after paste.
- Saves state.

### Cut

`Ctrl+X`: copy then delete selected.

### Constraints

- Pasting when clipboard is empty is a no-op.
- Connections referencing nodes outside the copied set are dropped silently.
- Movement arrows with one endpoint outside the copied set are dropped.

---

## Session 4 — Configurable Keyboard Shortcuts (v1.7)

**Goal:** Allow users to remap the application's keyboard shortcuts via a settings modal.

### Current Hardcoded Shortcuts

- `Ctrl+Z` — Undo
- `Ctrl+Y` — Redo
- `Ctrl+C` — Copy (from Session 3)
- `Ctrl+X` — Cut
- `Ctrl+V` — Paste
- `Delete` / `Backspace` — Delete selected
- `Escape` — Cancel / deselect
- `Ctrl+Enter` — Save node properties (in modal)

### Architecture

Introduce a `ShortcutManager` module:
- Stores a `shortcuts` map: `{ actionName: { key, ctrl, shift, alt } }`.
- Default shortcuts defined as `DEFAULT_SHORTCUTS` constant.
- User overrides persisted to `localStorage` under `jsSynTree_shortcuts`.
- `handleKeyDown` refactored to delegate to `ShortcutManager.match(event)`.

### Settings Modal

A new "Settings" button in the toolbar opens a modal with a shortcut table. Each row shows:
- Action name (human-readable)
- Current binding (e.g. `Ctrl+Z`)
- "Record" button: next keypress is captured and assigned
- "Reset" button per row

Conflicts highlighted with a warning but not blocked. "Reset All" button at the bottom. Persist on every change.

---

## Session 5 — Custom Fonts (v1.8)

**Goal:** Allow users to select from browser-available fonts or load a custom font file, applied globally to node labels.

### Part A: Font Selector

Font dropdown in the toolbar or Settings modal. Curated list:
- Arial (default), Georgia, Times New Roman, Courier New, Palatino, Helvetica

Selected font stored in `this.currentFont`. Nav panel font must match.

### Part B: Custom Font Loading

"Load Font..." button opens file picker (`.ttf`, `.otf`, `.woff`, `.woff2`). On selection: read as `ArrayBuffer`, create `FontFace`, call `.load()`, add to `document.fonts`, add to dropdown, select, redraw. Warn that loaded fonts do not persist across sessions.

### Part C: Font Application

Audit all font string construction: `Node._getFontString`, `Node._measureFormattedText`, SVG export `textToSVG`. All must reference `this.currentFont`.

---

## Session 6 — Feature Matrix Bracket Styles (v1.9)

**Goal:** Allow users to choose from three visual styles for the feature matrix brackets drawn below nodes.

### The Three Styles

- **Rounded** (current default): square bracket shape with small horizontal serifs at top and bottom of each arm.
- **Angular**: strict right-angle square brackets, no rounding, no serifs, heavier stroke weight.
- **Serif/Linguistic**: typographic style with taller serifs, thinner arm — closer to brackets in formal syntax literature.

### Architecture

Extract bracket drawing from `Node.draw` into `drawFeatureBrackets(ctx, cx, y, bw, bh, style)`. Global setting `this.featureBracketStyle` defaults to `'rounded'`, persisted to `localStorage`. Add selector to Settings modal. Update SVG export.

---

## Session 7 — Customisable Arrows (v2.0)

**Goal:** Movement arrows gain per-arrow properties and a contextual property panel.

### New Arrow Properties

```javascript
{
  lineStyle: 'dashed',      // 'dashed' | 'dotted' | 'solid'
  lineShape: 'curved',      // 'curved' | 'angular'
  annotation: '',
  annotationPosition: 0.5,  // 0.0–1.0 along the curve
  annotationOffset: 15      // px perpendicular offset
}
```

### Implementation

`dashed` = existing default. `dotted` = `setLineDash([2, 4])`. `solid` = `setLineDash([])`.

`angular` renders as two line segments: `from → controlPoint → to`.

Annotation rendered at parameterised position along curve, draggable in select mode (updates `annotationPosition` and `annotationOffset`).

Contextual property panel appears when arrow is selected; disappears on deselect. All new fields in save/load and undo/redo.

---

## Session 8 — Decorations (v2.1)

**Goal:** Allow users to draw freeform rectangular boxes over parts of the tree, with movable text annotations.

### Data Type

```javascript
{
  id: string,
  type: 'box',
  x: number, y: number,
  width: number, height: number,
  label: string,
  labelPosition: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right',
  strokeStyle: 'solid' | 'dashed' | 'dotted',
  strokeColour: string,
  strokeWidth: number,
  fillColour: string,
  cornerRadius: number
}
```

Stored in `this.decorations = []`. Included in save/load and undo/redo.

### Drawing Mode

"Add Box" mode button. Click-drag to define rectangle; discard if too small (< 20px in either dimension).

### Selection and Manipulation

In Select mode: click on border (not interior) to select. Resize handles at corners and midpoints. Drag interior to move. Decorations render behind nodes and connections (drawn first).

### Label Annotation

Optional text label renders outside the box at the specified corner. Dragging the label in select mode snaps to four corner positions.

### Property Panel

Contextual panel (same pattern as Session 7): label text, label position selector, stroke style/colour/width, fill colour/opacity, corner radius slider.

### Render Order

Decorations first → connections → movement arrows → nodes.

### Export

SVG and PNG: include decorations. LaTeX: omit; prepend `% Note: box decorations are not included in LaTeX export.`

### Bracket Exporter Update

Now that decorations exist, update `BracketExporter.export` to also include `_id` suffixes on nodes referenced by decorations (in addition to movement arrows). The exporter signature should accept `decorations` as a fourth argument.

---

## General Notes for All Sessions

**File naming:** Each session produces a new file. No build system; everything is inline in the single HTML file.

**Backwards compatibility:** The JSON save format must remain loadable by newer versions. Add a `version` field to saved JSON from Session 2 onwards. New fields must have safe defaults when loading older files.

**No external dependencies:** Do not introduce CDN-loaded libraries. All code must work offline from the local file.

**State management:** All new persistent data types must be included in `getCurrentStateObject`, `restoreState`, and the undo/redo stack.

**Testing note for Claude Code:** After each session, verify the following before closing:
1. Existing trees saved in v1.3 JSON format load correctly.
2. Undo/redo works for all new operations introduced in the session.
3. PNG and SVG export include all new visual elements.
4. Browser console is free of errors on a clean load.
5. *(From Session 2 onwards)* Bracket export round-trips correctly: export a tree, import the exported string, confirm topology, styles, and features are preserved.
