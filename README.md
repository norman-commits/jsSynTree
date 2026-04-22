# jsSynTree v2.1

A single-file, browser-based syntactic-tree drawing tool aimed at generative-syntax frameworks. Drag-and-drop interface, no bracketed-string entry required (though bracket import and export are supported). Runs locally — open the HTML file in any modern browser; no server, no build step, no external dependencies.

Free to use, distribute, and modify.

---

## What's new in v2.1

v2.1 closes out the eight-session roadmap defined in `jsSynTree_v2_design.md`. Compared with v1.3, this release adds:

- **Box decorations** with stroke/fill controls, corner radius, and movable corner labels
- **Bracket-notation export** plus a live nav-panel view of the tree as bracket notation
- **Configurable keyboard shortcuts** with conflict highlighting, persisted to `localStorage`
- **Custom font selection**, including loading `.ttf`/`.otf`/`.woff`/`.woff2` from disk
- **Three feature-matrix bracket styles**: rounded, angular, and serif/linguistic
- **Per-arrow customisation**: line style (dashed/dotted/solid), shape (curved/angular), and a draggable text annotation
- **Copy / cut / paste** of single nodes or whole subtrees
- **Bug fixes** — see "Bug fixes since v1.3" below

The bracket-notation grammar is specified in full in `jsSynTree_bracket_spec.md`.

---

## Quick start

1. Open `jsSynTree_v2_1.html` in a browser.
2. The toolbar offers five drawing modes, exports, and a settings button.
3. Click "Add Node" then click on the canvas to create nodes; the edit dialog opens for each new node so you can set text, features, and shape.
4. Switch to "Connect" and click parent then daughter to draw a dominance line.
5. "Select/Move" lets you drag nodes, double-click to edit, or Ctrl-click to grab a whole subtree.

---

## Toolbar reference

| Button | What it does |
|---|---|
| Add Node | Click to place a node; opens the edit dialog. |
| Connect | Click parent then daughter to draw a dominance line. |
| Select/Move | Drag nodes; Ctrl-click to select a subtree; double-click to edit. |
| Add Movement | Click source then destination to draw a movement arrow. |
| Add Box | Drag on the canvas to draw a rectangular decoration. |
| Show Grid | Toggles a 20-px snap grid. |
| Undo / Redo | Step through the last 30 changes. |
| Delete Selected | Removes whatever is currently selected. |
| Clear All | Wipes the canvas (with confirmation). |
| Width / Height / Auto Fit | Resize the canvas, or shrink-wrap it to current content. |
| From Bracket | Open the import dialog for labelled-bracket notation. |
| Export Bracket | Copies the current tree as bracket notation to the clipboard. |
| Export PNG / SVG / LaTeX | Three export formats. LaTeX uses the `forest` package. |
| Save Tree / Load Tree | Save and load the full canvas state as JSON. |
| Settings | Font, feature-bracket style, keyboard shortcuts. |

---

## Node text formatting

Inside the node-text and feature fields, the following inline markers render as expected on the canvas, in SVG export, and in LaTeX export:

| Markup | Renders as |
|---|---|
| `**text**` | **bold** |
| `*text*` | *italic* |
| `~text~` | strikethrough |
| `^text^` | superscript |
| `_text_` | subscript |
| `[SIZE=14]text[/SIZE]` | sized text |

Newlines inside a node label are real `\n` characters in the text field; in bracket notation they are encoded as the two-character sequence `\n`.

The small-caps marker `[SC]…[/SC]` is **not supported**. It was removed in v1.4 because canvas does not honour `font-variant: small-caps`. Use `[SIZE=12]…[/SIZE]` for a reduced-size approximation.

---

## Bracket notation

The full grammar is in `jsSynTree_bracket_spec.md`. The short version:

```
[Label|style {features} children]
```

- Either `[ ]` or `( )` brackets are accepted on import; `[ ]` is canonical on export.
- `|style` is optional and may be `box` or `circle`.
- `{features}` is optional; features are separated by `;`. Inline text formatting works inside features.
- Children are nested nodes, separated by whitespace. Bare words (no brackets) are leaf nodes.
- A trailing `_id` on a label (e.g. `[TP_A]`) is stripped from the display label and reserved for cross-references from movement arrows and decorations.
- Escapes: `\|` for a literal pipe, `\;` for a literal semicolon in a feature, `\n` for a newline inside a label.

### Examples

Simple tree:

```
[TP [NP [D the] [N cat]] [T' [T sat] [VP [V t]]]]
```

With styles and features:

```
[TP
  [NP|circle {uφ; +nom}
    [D the]
    [N cat]
  ]
  [T'
    [T|box {EPP}]
    [VP [V saw] [NP|box {+acc} [D the] [N dog]]]
  ]
]
```

A multi-line node label uses the literal two-character sequence `\n`:

```
[**DP**\n*i*|circle {uφ; EPP}]
```

### What bracket export does and doesn't capture

Bracket notation captures topology, labels, styles, and features faithfully. It does **not** capture node coordinates, movement-arrow geometry/style, or box decorations. Round-tripping a tree through export then import yields the same structure but the layout engine repositions every node from scratch (this is the same trade-off the import dialog has always made). The exporter prepends a comment line saying so.

Truly disconnected nodes (no parent and no children) are emitted at the bottom under a `% disconnected:` comment.

---

## Movement arrows

After drawing an arrow, click it to reveal the contextual property panel in the top right:

- **Line style** — dashed, dotted, or solid
- **Line shape** — smooth quadratic curve, or angular two-segment polyline
- **Annotation** — optional text label drawn along the arrow
- **Position along curve** — slider, 0 to 1
- **Offset** — perpendicular displacement of the annotation, in px

The annotation is also draggable directly on the canvas in select mode; dragging snaps the position and offset values to whatever you've moved it to.

---

## Box decorations

Switch to "Add Box" mode and drag to draw a rectangle. Boxes render behind everything else, so they're a natural way to highlight a constituent without obscuring it.

Click a box's border (not its interior) in select mode to select it. The contextual property panel offers:

- Label text and corner position (top-left, top-right, bottom-left, bottom-right)
- Stroke style (solid, dashed, dotted), colour, and width
- Fill colour and opacity
- Corner radius

Drag the interior to move the box; drag a handle to resize; drag the label to move it (it snaps to the nearest corner). Boxes appear in PNG and SVG export but **not** in LaTeX export — the LaTeX file gets a `% Note: box decorations are not included in LaTeX export.` comment instead.

When you draw a box, the IDs of any nodes whose centres fall inside it are recorded; bracket export will then attach `_A`-style IDs to those nodes so external tooling can relate them back.

---

## LaTeX export

The exported code uses the `forest` package; strikethrough requires `soul`. Add this to your LaTeX preamble:

```latex
\usepackage{forest}
\usepackage{soul}   % for strikethrough only
```

The exporter handles bold, italic, strikethrough, super/subscript, and multi-line labels (with `\\`). `[SIZE=…]` markers are stripped, since `forest` doesn't have a clean per-token size primitive. Node styles (circle/box) and box decorations are not currently encoded in the LaTeX output.

---

## Custom fonts

Settings → "Font" lets you pick from a curated list of safe web fonts (Arial, Georgia, Times New Roman, Courier New, Palatino, Helvetica). The choice is persisted to `localStorage` and applied to canvas, nav panel, and SVG/PNG export.

"Load Font…" lets you load a `.ttf`/`.otf`/`.woff`/`.woff2` file from disk via the `FontFace` API. **Loaded fonts do not persist across sessions** — you'll need to load again next time. There's an alert reminding you of this when you load.

---

## Keyboard shortcuts

Defaults:

| Action | Default |
|---|---|
| Undo | Ctrl+Z |
| Redo | Ctrl+Y |
| Copy | Ctrl+C |
| Cut | Ctrl+X |
| Paste | Ctrl+V |
| Delete selected | Delete (or Backspace) |
| Cancel / deselect | Escape |
| Save node properties (in modal) | Ctrl+Enter |

All shortcuts are remappable in Settings. Click "Record" next to any binding and press the key combination you want; "Reset" restores that binding to its default; "Reset All" wipes all overrides. Conflicts (two actions bound to the same combination) are highlighted in amber but not blocked. Overrides persist to `localStorage` under `jsSynTree_shortcuts`.

---

## Save / load file format

Tree files are JSON with a `version` field (currently `"2.1"`). The shape:

```json
{
  "version": "2.1",
  "nodes": [{ "id": "...", "x": 100, "y": 100, "text": "...", "features": [...], "style": "none|box|circle" }],
  "connections": [{ "id": "...", "fromId": "...", "toId": "..." }],
  "movementArrows": [{ "id": "...", "fromId": "...", "toId": "...", "controlPointX": 0, "controlPointY": 0, "lineStyle": "dashed", "lineShape": "curved", "annotation": "", "annotationPosition": 0.5, "annotationOffset": 15 }],
  "decorations": [{ "id": "...", "type": "box", "x": 0, "y": 0, "width": 0, "height": 0, "label": "", "labelPosition": "top-left", "strokeStyle": "solid", "strokeColour": "#333333", "strokeWidth": 2, "fillColour": "#ffff64", "fillOpacity": 0.15, "cornerRadius": 0 }],
  "settings": { "showGrid": true, "canvasWidth": 1200, "canvasHeight": 600 }
}
```

Files saved in v1.3 still load — missing fields fall back to safe defaults.

---

## Browser requirements

Anything modern: Chrome, Firefox, Safari, or Edge from roughly 2021 onwards. The code uses `FontFace`, `navigator.clipboard.writeText`, `requestAnimationFrame`, `Set`, `Object.assign`, and template literals. There is one optional use of `CanvasRenderingContext2D.roundRect`, gracefully falling back to `fillRect` on older engines.

---

## Bug fixes since v1.3

- **v1.4** — Removed the broken `[SC]` small-caps markup (canvas can't render `font-variant: small-caps`). Fixed circle-styled nodes having branch lines that intersected the circle instead of meeting its edge.
- **v1.5** — Bracket parser upgraded to handle `|style`, `{features}`, multi-line labels, and `_id` suffixes. Bracket exporter and live nav-panel added.
- **v1.6** — Internal copy/cut/paste added.
- **v1.7** — Configurable keyboard shortcuts.
- **v1.8** — Custom fonts.
- **v1.9** — Three feature-matrix bracket styles.
- **v2.0** — Per-arrow line style, line shape, and draggable annotation.
- **v2.1** — Box decorations.

### Additional fixes in this v2.1 build

- **Nav panel toggle no longer becomes unreachable when collapsed.** The collapsed panel previously had `opacity: 0; pointer-events: none`, hiding the toggle button along with the content. The collapsed state now keeps a thin header strip (32 px) visible so the toggle stays clickable.
- **Bracket exporter now correctly emits a `% disconnected:` section** for truly isolated nodes (no parent, no children). The previous filter contained a logical contradiction — it required `!parentMap[n.id] && !roots.includes(n)`, but isolated nodes were already in `roots`, so the disconnected section never appeared.
- **Copy/paste now preserves arrow customisations.** `lineStyle`, `lineShape`, `annotation`, `annotationPosition`, and `annotationOffset` were previously dropped during copy and silently re-defaulted on paste.
- **`setMode` no longer has dead-code box cancellation.** The cancellation block was guarded by `if (this.mode !== newMode)` placed *after* the assignment, so it was always false. Mouseup recovery covered for it in practice, but the intent is now correctly expressed.
- **SVG-export formatting check no longer references the removed `[SC]` marker.**
- **Bracket exporter feature separator is now `; ` (with space)** rather than just `;`, matching the spec examples and improving readability.
- **`scrollIntoView` call in the nav panel is now defensively guarded**, making the code portable to environments where that DOM method is missing (e.g. some test harnesses).

---

## Files

| File | Purpose |
|---|---|
| `jsSynTree_v2_1.html` | The whole application, single-file. Open this. |
| `jsSynTree_v2_design.md` | Design document tracking the eight-session roadmap. |
| `jsSynTree_bracket_spec.md` | Full bracket-notation grammar specification. |
| `readme.md` | This file. |

---

## Known limitations

- **LaTeX export does not encode node styles or box decorations.** A comment is added to the output explaining the omission.
- **Loaded custom fonts don't persist** across browser sessions — this is a `FontFace` API constraint.
- **Bracket round-trips lose geometry.** The layout engine regenerates positions from topology on import. Movement arrows and decorations are also lost on bracket round-trip — they live in the JSON save format only.
- **The nav panel re-renders on every redraw.** No caching. For trees with hundreds of nodes this could become a perceptible cost; for the trees this tool actually targets (tens of nodes) it is fine.
- **The annotation font on movement arrows uses a global app reference** (`app.currentFont`). This is functionally correct but architecturally a code smell; if you ever want to instantiate `MovementArrow` outside the main app context, you'd need to thread a font parameter through.
