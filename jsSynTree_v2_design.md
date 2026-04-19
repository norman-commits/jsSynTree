# jsSynTree v2 — Feature Development Design Document

## Overview

This document specifies incremental feature development for jsSynTree, a browser-based syntactic tree drawing tool. All development is in a single self-contained HTML file. Each session is independently shippable and buildson the previous version. Sessions are ordered to front-load bug fixes and foundational architecture before more complex features.

The working codebase is `jsSynTree_v1_3.html`. Each session should produce a new versioned file.

---

## Session 1 — Bug Fixes (v1.4)

**Goal:** Resolve two existing defects before building further. No new features.

### Bug 1: Small Caps Rendering

**Problem:** The `[SC]...[/SC]` markup tag is parsed correctly but small caps do not render visually on the canvas. The `_applyTextStyles` method uppercases the text, but `font-variant: small-caps` cannot be set on a canvas context — it is a CSS property only. The result is full uppercase rather than true small caps.

**Fix:** Implement a manual small caps simulation in `_drawFormattedText`. When a segment has `styles.smallcaps === true`, render the text in two passes:
- Uppercase letters at the base font size
- Lowercase letters converted to uppercase but rendered at approximately 75% of the base font size

This requires splitting the string character by character, tracking case, and measuring/drawing each run separately. It is fiddly but the only correct approach on a canvas.

The SVG export path already uses `font-variant="small-caps"` which is correct and should not be changed.

### Bug 2: Circle Node Style Intersects Daughter Branches

**Problem:** When a node has `style === 'circle'`, the circle is sized and positioned relative to the node's text bounding box. The branch lines from parent to child use `getConnectionPoint`, which returns `bounds.bottom` for the parent. For a circle node, `bounds.bottom` is the bottom of the text bounding box, not the bottom of the circle, so the line visually starts inside or behind the circle rather than at its edge.

**Fix:** Override `getConnectionPoint` behaviour for circle-style nodes. Calculate the actual circle radius (`Math.max(shapeWidth, nodeHeight) / 2`) and the circle centre (`y + nodeHeight / 2`), then return the point on the circle's circumference in the direction of the child node rather than the raw `bounds.bottom`. This requires knowing the direction to the child at draw time, which `getConnectionPoint` does not currently receive.

The cleanest fix is to add an optional `targetX, targetY` argument to `getConnectionPoint`. For non-circle nodes these are ignored and existing behaviour is unchanged. For circle nodes, compute the angle from circle centre to target and return the circumference point. Update `ConnectionModule.Connection.draw` and `MovementArrowModule` to pass the counterpart node's coordinates when calling `getConnectionPoint`.

---

## Session 2 — Navigation Panel (v1.5)

**Goal:** A collapsible read-only panel on the right side of the canvas displaying a live bracket notation view of the tree, with node highlighting synced to canvas selection.

### Layout Changes

The current layout is a single `<canvas>` inside `.container`. Introduce a flex row wrapper:

```
.container
  .toolbar          (unchanged)
  .info             (unchanged)
  .workspace        (new flex row)
    canvas          (flex-grow: 1)
    .nav-panel      (fixed width, collapsible)
```

The nav panel has a hide/show toggle button. When hidden, only the toggle button remains visible at the right edge, and the canvas expands to fill the space.

### Bracket String Generation

Add a `BracketViewModule` with a `generate(nodes, connections)` function that:

1. Finds root nodes (nodes with no incoming connections), same logic as `LaTeXExportModule`.
2. Recursively builds the bracket string by traversing the connection graph from each root.
3. For each node, outputs `[LABEL]` or `[LABEL [CHILD1] [CHILD2]]`.
4. **Label rendering:** Strip internal formatting markup (`**`, `*`, `~`, `^`, `_`, `[SC]`, `[SIZE=]`) for readability. The nav panel is for navigation, not typesetting. Display plain text labels only.
5. **Node style annotation:** Append a sigil to the label if the node has a non-default style: `TP<circle>` or `TP<box>`. Keep it terse.
6. **Features:** Excluded from the bracket string. They are accessible by clicking the node on canvas and inspecting the edit modal, or in a future tooltip.
7. **Disconnected nodes:** Any node not reachable from a root is listed at the bottom under a `% disconnected:` comment line.
8. **Movement arrows:** Listed at the bottom as `% movement: LABEL_A → LABEL_B`, using node labels. Not interactive.

Alongside the string, `generate` must also return a **span map**: an array of `{ nodeId, start, end }` objects recording the character positions of each node's full span (opening `[` to closing `]`) in the generated string. This is what enables highlighting.

### Panel Rendering

The nav panel contains:
- A `<pre>` or `<div>` with `white-space: pre; font-family: monospace; font-size: 13px; overflow: auto`.
- The bracket string is rendered as a sequence of `<span>` elements, one per node span, with the plain text in between as text nodes. This allows individual node spans to receive a highlight class without re-rendering the whole string.
- A hide button (`«` / `»`) at the top right of the panel.

### Highlighting Behaviour

When `selectedNode` changes (canvas click, or any selection event), the app calls `navPanel.highlightNode(nodeId)`:
1. Remove any existing highlight class.
2. Look up `nodeId` in the span map.
3. Add a highlight class to the corresponding `<span>` (e.g. yellow background, or matching the canvas blue glow).
4. Scroll the span into view within the panel (`scrollIntoView({ block: 'nearest' })`).

Conversely, clicking a `<span>` in the panel should select the corresponding node on the canvas and redraw. This is a one-way click (panel → canvas selection), not a full edit.

### Update Trigger

The bracket view regenerates whenever `redraw()` is called. Since `redraw` is already the single point of canvas update, hook into it: at the end of `redraw()`, call `this.navPanel.update(this.nodes, this.connections, this.selectedNode)`.

---

## Session 3 — Copy and Paste (v1.6)

**Goal:** Allow nodes (and optionally subtrees) to be copied and pasted onto the canvas.

### Scope

Copy/paste operates on the canvas, not on the bracket string (the nav panel is read-only). The clipboard target is the app's internal state, not the system clipboard — this avoids the complexity of serialising/deserialising a full subtree to and from text and then competing with the user's system clipboard for text content. However, the system clipboard *can* be used as a fallback serialisation target for cross-tab paste, if desired as a stretch goal.

### Copy

When one or more nodes are selected (either a single node or a dominance-selected subtree):
- `Ctrl+C` (or a toolbar Copy button) serialises the selected nodes and any connections *between* them (not connections to unselected nodes) into an internal `this.clipboard` object on the app.
- Structure: `{ nodes: [...], connections: [...] }`, same shape as `getCurrentStateObject`.
- Features, styles, and text are all preserved.
- Movement arrows between copied nodes are also copied.

### Paste

`Ctrl+V` (or a toolbar Paste button):
- Deserialises `this.clipboard`.
- Assigns new IDs to all pasted nodes and connections (to avoid ID collisions).
- Offsets all pasted node coordinates by a fixed amount (e.g. +40px x, +40px y) so the paste is visibly distinct from the original.
- If pasted repeatedly, increment the offset each time.
- Selects all pasted nodes as a dominance group immediately after paste.
- Saves state.

### Cut

`Ctrl+X`: copy then delete selected. Straightforward composition of the above.

### Constraints and Edge Cases

- Pasting when clipboard is empty is a no-op.
- Pasting nodes whose connections reference nodes not in the clipboard (i.e. partial subtree copies that include a child but not its parent) drops those external connections silently. Only internal connections are preserved.
- Movement arrows that reference one node inside and one outside the copied set are dropped.

---

## Session 4 — Configurable Keyboard Shortcuts (v1.7)

**Goal:** Allow users to remap the application's keyboard shortcuts via a settings modal.

### Current Hardcoded Shortcuts

Document all existing shortcuts first:
- `Ctrl+Z` — Undo
- `Ctrl+Y` — Redo
- `Ctrl+C` — Copy (new, from Session 3)
- `Ctrl+X` — Cut (new)
- `Ctrl+V` — Paste (new)
- `Delete` / `Backspace` — Delete selected
- `Escape` — Cancel / deselect
- `Ctrl+Enter` — Save node properties (in modal)

Mode shortcuts are not currently bound to keys but could be added (e.g. `A` for Add Node, `S` for Select, `C` for Connect).

### Architecture

Introduce a `ShortcutManager` module:
- Stores a `shortcuts` map: `{ actionName: { key, ctrl, shift, alt } }`.
- Default shortcuts are defined as a `DEFAULT_SHORTCUTS` constant.
- User overrides are persisted to `localStorage` under the key `jsSynTree_shortcuts`.
- On load, merge defaults with stored overrides.
- The existing `handleKeyDown` method is refactored to delegate to `ShortcutManager.match(event)` which returns an action name or null.

### Settings Modal

A new "Settings" button in the toolbar opens a modal with a shortcut table. Each row shows:
- Action name (human-readable)
- Current key binding (displayed as e.g. `Ctrl+Z`)
- A "Record" button: when clicked, the next keypress (excluding modifier-only presses) is captured and assigned to that action.
- A "Reset" button per row to restore the default.
- A "Reset All" button at the bottom.

Conflicts (two actions assigned the same shortcut) should be highlighted with a warning but not blocked — the user may intentionally want to unbind something.

### Persistence

Save the full shortcuts map to `localStorage` on every change. Load on app initialisation. This survives page refresh without any server or file dependency.

---

## Session 5 — Custom Fonts (v1.8)

**Goal:** Allow users to select from browser-available fonts or load a custom font file, applied globally to node labels.

### Part A: Font Selector

Add a font dropdown to the toolbar (or the Settings modal from Session 4). Populate it with a curated list of known safe web fonts:
- Arial (current default)
- Georgia
- Times New Roman
- Courier New
- Palatino
- Helvetica

The selected font is stored in `this.currentFont` on the app and substituted wherever `Arial` is currently hardcoded in canvas font strings and SVG output.

The nav panel font should match.

### Part B: Custom Font Loading via File

Add a "Load Font..." button that opens a file picker accepting `.ttf`, `.otf`, `.woff`, `.woff2`. On selection:
1. Read the file as an `ArrayBuffer`.
2. Create a `FontFace` object: `new FontFace(derivedName, arrayBuffer)`.
3. Call `.load()` and add to `document.fonts`.
4. Add the font name to the dropdown and select it.
5. Trigger a redraw.

**Constraints:**
- Font files are not persisted across sessions (no `localStorage` for binary data without base64 encoding, which is heavy). Warn the user that the font must be reloaded on next session.
- The font name shown in the dropdown is derived from the filename (strip extension).
- If `FontFace` is not supported (old browsers), show a graceful error.

### Part C: Font Application

Audit all locations in the code where font strings are constructed and ensure they all reference `this.currentFont` rather than the literal `'Arial'`. This includes:
- `Node._getFontString`
- `Node._measureFormattedText`
- SVG export `textToSVG`
- LaTeX export (font is irrelevant to LaTeX output, no change needed)

---

## Session 6 — Feature Matrix Bracket Styles (v1.9)

**Goal:** Allow users to choose from three visual styles for the feature matrix brackets drawn below nodes.

### The Three Styles

**Style A — Rounded (current default):** The existing square bracket shape with slightly rounded corners. Small horizontal serifs at top and bottom of each bracket arm. Rendered with `ctx.beginPath` line segments.

**Style B — Angular:** Strict right-angle square brackets, no rounding, no serifs, heavier stroke weight. Clean and minimal.

**Style C — Serif/Linguistic:** Styled after the brackets used in formal phonology and syntax literature. Taller serifs, slightly thinner arm, overall more typographic. This is achieved by adjusting the serif length and stroke width parameters.

### Architecture

Extract the bracket drawing code from `Node.draw` into a standalone `drawFeatureBrackets(ctx, cx, y, bw, bh, style)` function, where `style` is `'rounded'`, `'angular'`, or `'serif'`. Each branch of a switch statement implements the relevant path commands.

The current bracket style is a global setting stored in `this.featureBracketStyle` on the app, defaulting to `'rounded'`. Persist to `localStorage`.

Add a three-way toggle or dropdown to the Settings modal (Session 4) or toolbar.

Update the SVG export to honour the bracket style using equivalent SVG path commands.

---

## Session 7 — Customisable Arrows (v2.0)

**Goal:** Movement arrows gain a property panel allowing per-arrow customisation of line style, shape, and text annotations.

### New Arrow Properties

Each `MovementArrow` object gains the following new fields with defaults:

```javascript
{
  lineStyle: 'dashed',      // 'dashed' | 'dotted' | 'solid'
  lineShape: 'curved',      // 'curved' | 'angular'  
  annotation: '',           // text string
  annotationPosition: 0.5,  // 0.0–1.0 along the curve
  annotationOffset: 15      // px perpendicular offset from curve
}
```

### Line Style

`dashed` is the existing default. `dotted` uses a tighter `setLineDash([2, 4])`. `solid` uses `setLineDash([])`.

### Line Shape

`curved` is the existing quadratic bezier. `angular` renders as two straight line segments meeting at the control point: `from → controlPoint → to`. The arrowhead calculation is unchanged (always uses the final segment direction).

### Annotation

A text label rendered at a point along the arrow path. Position is parameterised 0–1 along the curve. The label is offset perpendicularly from the curve by `annotationOffset` pixels. The user can drag the annotation by clicking and dragging it in select mode (hit-test against a bounding box around the text). Dragging updates `annotationPosition` and `annotationOffset`.

### Arrow Property Panel

When a movement arrow is selected, a small floating property panel appears adjacent to it (or in a fixed sidebar position) with:
- Line style selector (radio or dropdown)
- Line shape selector
- Annotation text input
- Annotation position slider (0–1)

This panel is not a modal — it appears in-context and disappears when the arrow is deselected, similar to how design tools show contextual property panels.

### Persistence

All new properties are included in `getCurrentStateObject` and `restoreState`, maintaining save/load compatibility.

---

## Session 8 — Decorations: Boxes and Annotations (v2.1)

**Goal:** Allow users to draw freeform rectangular boxes over parts of the tree, with movable text annotations, to indicate constituent groupings, ellipsis sites, or other structural annotations.

### New Data Type: Decoration

A `Decoration` object is independent of nodes and connections:

```javascript
{
  id: string,
  type: 'box',              // extensible: 'ellipse' later if desired
  x: number,               // top-left
  y: number,
  width: number,
  height: number,
  label: string,
  labelPosition: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right',
  strokeStyle: 'solid' | 'dashed' | 'dotted',
  strokeColour: string,     // hex
  strokeWidth: number,
  fillColour: string,       // hex with alpha, default transparent
  cornerRadius: number      // 0 = sharp, >0 = rounded
}
```

Decorations are stored in `this.decorations = []` on the app and included in save/load state.

### Drawing Mode

Add an "Add Box" mode button to the toolbar. In this mode, the canvas cursor changes to crosshair. The user clicks and drags to define the box rectangle, releasing the mouse to confirm. The box appears immediately. If the drag area is too small (under ~20px), the box is discarded.

### Selection and Manipulation

In Select mode, decorations are selectable by clicking on their border (not interior, to avoid blocking node interaction). Selected decorations show resize handles at corners and midpoints. Dragging a handle resizes the box. Dragging the interior moves it. Decorations render behind nodes and connections (drawn first in the render order).

### Label Annotation

Each decoration has an optional text label. The label renders outside the box at the specified corner position. The user can drag the label independently by clicking and dragging it (in select mode), which snaps to the four corner positions rather than freeform, to keep it tidy.

### Property Panel

When a decoration is selected, a contextual property panel (same pattern as Session 7 arrow properties) shows:
- Label text input
- Label position selector (four corners)
- Stroke style, colour, width
- Fill colour and opacity
- Corner radius slider

### Render Order

Decorations are drawn first (bottom layer), then connections, then movement arrows, then nodes. This ensures boxes appear behind the tree structure.

### SVG and PNG Export

Decorations must be included in SVG export using `<rect>` elements with appropriate attributes. PNG export inherits this automatically since it renders the canvas.

### LaTeX Export

Decorations cannot be trivially represented in the `forest` package. Omit them from LaTeX export. Add a comment line at the top of the generated output: `% Note: box decorations are not included in LaTeX export.`

---

## General Notes for All Sessions

**File naming:** Each session produces a new file, e.g. `jsSynTree_v1_4.html`, `jsSynTree_v1_5.html` etc. No build system; everything is inline in the single HTML file.

**Backwards compatibility:** The JSON save format must remain loadable by newer versions. Add a `version` field to the saved JSON from Session 2 onwards. New fields introduced in later sessions should be given safe defaults when loading older files that lack them.

**No external dependencies:** Do not introduce CDN-loaded libraries. All code must work offline from the local file.

**State management:** All new persistent data types (decorations, arrow properties) must be included in `getCurrentStateObject`, `restoreState`, and the undo/redo history stack.

**Testing note for Claude Code:** After each session, verify the following manually before closing the session:
1. Existing trees saved in v1.3 JSON format load correctly.
2. Undo/redo works for all new operations introduced in the session.
3. PNG and SVG export include all new visual elements.
4. The browser console is free of errors on a clean load.
