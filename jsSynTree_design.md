# jsSynTree Design Specification and Function Reference

This document provides a comprehensive feature specification and detailed function-by-function reference for **jsSynTree v2.2.2**.

---

## 1. Architectural and Design Philosophy

jsSynTree is designed as a **single-file, client-side browser application** for drawing generative syntactic trees. It is built to run entirely locally in any modern browser without a web server, build steps, or external package dependencies.

### Key Architectural Guidelines
- **Zero Dependencies:** All styling (CSS), page layout (HTML), and logical components (JavaScript) are embedded directly within a single `.html` file.
- **Vanilla Technologies:** Built using standard HTML5 Canvas API, native Web APIs (like `FontFace` and `FileReader`), and vanilla JavaScript.
- **Client-Side State Management:** The state of the canvas is maintained in-memory and persisted optionally to the user's local disk via JSON files, or to browser `localStorage` for configuration settings.
- **HiDPI/Retina Support:** High-density displays are handled by scaling physical canvas dimensions by `window.devicePixelRatio` and scaling coordinates back to logical pixels using `ctx.setTransform`.

---

## 2. Feature Specification

### 2.1 Drawing Modes
The application runs in one of five mutually exclusive drawing modes, managed by the toolbar:
1. **Add Node:** Clicking on the canvas instantiates a new syntactic node.
2. **Connect:** Drawing dominance lines (branches). Clicking a parent node and then a daughter node creates a straight branch connecting them.
3. **Select/Move:** 
   - Dragging a single node adjusts its coordinates.
   - **Dominance Subtree Selection:** Ctrl-clicking a parent node selects that node and all its dominated descendants as a group, allowing them to be dragged or deleted together.
   - Hovering over elements dynamically alters the mouse cursor contextually to indicate interaction possibilities (e.g., `grab`, `grabbing`, `crosshair`, resize cursors).
4. **Add Movement:** Drawing movement arrows to represent syntactic movement. Clicking a source node and then a destination node generates a curved or angular arrow.
5. **Add Box:** Creating rectangular highlight boxes (decorations). Dragging on the canvas defines the box dimensions.

### 2.2 Node Styles
Nodes can be rendered in three styles:
- **None:** Text-only label.
- **Circle:** A circle is drawn around the node text. Intersections with daughter branches and movement arrows are computed dynamically to end at the circle's circumference rather than its geometric centre.
- **Box:** A rectangle is drawn surrounding the node text.

### 2.3 Rich Text Formatting Engine
The canvas and exporters support inline markdown-like formatting tags. The formatting logic is cached for efficiency:
- `**bold**`: Renders bold text.
- `*italic*`: Renders italic text.
- `~strikethrough~`: Renders text with a horizontal line drawn through it.
- `^superscript^`: Renders text shifted upwards and scaled down.
- `_subscript_`: Renders text shifted downwards and scaled down.
- `[SIZE=N]text[/SIZE]`: Dynamically overrides the font size to `N` pixels.
- `\n`: Renders multi-line node labels.

### 2.4 Labelled-Bracket Notation (Import and Export)
The application parses and generates tree structures using bracketed notation.
- **Grammar:** `[Label|style {features} children]`
- **Style Suffix:** Appending `|box` or `|circle` to the label specifies the node's visual style.
- **Feature Block:** `{feature1; feature2}` encodes features separated by semicolons.
- **Escapes:** Escaped characters are supported (`\|` for a literal pipe, `\;` for a literal semicolon in a feature, and `\n` for a newline inside labels).
- **ID Hinting:** Suffixes like `_A` or `_B` denote node cross-references, allowing external tools to map movement arrows and box selections back to particular nodes.
- **Layout Engine:** When importing a bracket string, a post-order traversal layout algorithm calculates node coordinates from topology using customisable horizontal and vertical spacing values.

### 2.5 Navigation Panel
A collapsable flex-row panel side-by-side with the canvas shows a live textual representation of the tree in bracketed notation.
- **Bidirectional Sync:** 
  - Selecting a node on the canvas highlights its corresponding bracket token in the navigation panel and smooth-scrolls it into view.
  - Clicking a bracket token in the navigation panel selects the associated node on the canvas.

### 2.6 Clipboard Operations
A custom internal clipboard supports standard hotkeys:
- **Ctrl+C (Copy):** Copies the selected node or dominance-selected subtree (retaining internal branches and movement arrows).
- **Ctrl+X (Cut):** Copies the selected items to the internal clipboard and deletes them from the canvas.
- **Ctrl+V (Paste):** Instantiates the copied elements with new IDs, shifted downward-right by 40 pixels.

### 2.7 Customisable Keyboard Shortcuts
Hotkeys are handled by a dedicated manager:
- Default bindings (Undo = `Ctrl+Z`, Redo = `Ctrl+Y`, Copy = `Ctrl+C`, Cut = `Ctrl+X`, Paste = `Ctrl+V`, Delete = `Delete`/`Backspace`, Cancel = `Escape`, Save modal = `Ctrl+Enter`).
- All shortcuts can be customised in the Settings modal by recording keystrokes.
- Mappings are persisted to `localStorage` under `jsSynTree_shortcuts`.
- Conflicting shortcut bindings are highlighted in amber.

### 2.8 Custom Font Engine
- Users can choose from a list of safe web fonts (Arial, Georgia, Times New Roman, Courier New, Palatino, Helvetica).
- A custom font loader reads `.ttf`, `.otf`, `.woff`, or `.woff2` files using the browser's `FontFace` API and registers them dynamically.

### 2.9 Movement Arrow Customisation
Selected movement arrows reveal a contextual property panel allowing configuration of:
- **Line Style:** `dashed`, `dotted`, or `solid`.
- **Line Shape:** `curved` (quadratic Bezier) or `angular` (two straight lines).
- **Movement Type:** Semantic labels (`A`, `A-bar`, `Head`, `Other`) for external processor parsing.
- **Annotations:** An annotation text label drawn along the arrow line. The label's position along the curve (0.0 to 1.0) and perpendicular offset are adjustable via panel sliders or by direct dragging on the canvas.

### 2.10 Box Decorations
Rectangular highlight boxes render on the bottom layer:
- Contextual properties support custom label, corner label alignment, line styles, border colours, border width, fill colours, fill opacity, and corner radii.
- Node reference detection: Nodes spatially enclosed by the box boundaries on creation are associated with the box in bracket exports.

### 2.11 Exporters
- **PNG Export:** Generates a PNG download after hiding user-interface markers (gridlines, selection borders).
- **SVG Export:** Compiles all active tree elements into a clean vector file, translating text styles into SVG `<text>` and `<tspan>` tags.
- **LaTeX Export:** Generates code for the LaTeX `forest` package. Multi-line labels translate to `\\` breaks, and strikethroughs map to the `soul` package's `\st` command.

---

## 3. Function and Module Reference

The following sections document every module, class, helper function, and method in `jsSynTree_v2_2_2.html` in sequential order of their definition.

---

### 3.1 TextFormatter Module
An IIFE module that parses and caches rich text markdown strings.

#### `parseFormattedText(text)`
- **Purpose:** Parses a string with markdown-like syntax into styled text segments.
- **Arguments:** `text` (String) - Raw node or feature text.
- **Returns:** Array of segment objects: `[{ text: String, styles: Object }]`.
- **Logic:**
  - Checks if the string is cached in `parseCache` and returns it immediately if found.
  - Matches regular expressions for bold (`**`), italic (`*`), strikethrough (`~`), superscript (`^`), subscript (`_`), and font size (`[SIZE=N]`).
  - Sorts matches by position and length, filters out overlaps (keeping the longest match), and tokenises the remaining parts.
  - Caches results in `parseCache` (limited to 100 entries; evicts oldest on overflow).

---

### 3.2 Utils Module
An IIFE module providing math, coordinate mapping, and grid snapping utilities.

#### `snapToGrid(coord, gridSize, enabled)`
- **Purpose:** Snaps a pixel coordinate to a grid increment.
- **Arguments:**
  - `coord` (Number) - The raw pixel coordinate.
  - `gridSize` (Number) - Dimension of grid squares (defaults to 20px).
  - `enabled` (Boolean) - Whether grid snapping is toggled on.
- **Returns:** Snapped coordinate (Number).

#### `getCanvasCoordinates(canvas, event)`
- **Purpose:** Maps browser client coordinates to logical canvas space.
- **Arguments:**
  - `canvas` (HTMLCanvasElement) - Target canvas.
  - `event` (MouseEvent) - The triggered mouse event.
- **Returns:** Coordinates object: `{ x: Number, y: Number }`.

#### `distanceToLineSegment(x, y, x1, y1, x2, y2)`
- **Purpose:** Computes the shortest distance from a point `(x, y)` to a straight line segment between `(x1, y1)` and `(x2, y2)`.
- **Returns:** Distance (Number).
- **Logic:** Projects the point onto the segment line vector, clamping the parameter to `[0, 1]` to handle endpoints, and calculates the vector magnitude.

---

### 3.3 StateManager Module
An IIFE module handling undo/redo history stacks.

#### `class Manager`
Manages history state arrays.
- **Methods:**
  - `constructor()`: Instantiates empty `historyStack` and `redoStack` arrays.
  - `saveState(state)`: Deep-clones the app state, pushes it to `historyStack` (capping length at 30), and clears `redoStack`.
  - `undo()`: Pops the top state from `historyStack`, pushes it to `redoStack`, and returns the previous state on `historyStack`.
  - `redo()`: Pops the top state from `redoStack`, pushes it to `historyStack`, and returns it.
  - `canUndo()`: Returns boolean indicating if `historyStack` has more than 1 entry.
  - `canRedo()`: Returns boolean indicating if `redoStack` is not empty.
  - `getLastState()`: Returns the most recent state entry without popping it.
  - `clearRedoStack()`: Empties the `redoStack` array.

---

### 3.4 Global Helper: `drawFeatureBrackets`
A global utility function for drawing feature matrix brackets.

#### `drawFeatureBrackets(ctx, cx, y, bw, bh, style)`
- **Purpose:** Draws brackets around node features on the canvas.
- **Arguments:**
  - `ctx` (CanvasRenderingContext2D) - Drawing context.
  - `cx` (Number) - Centre horizontal coordinate of node features.
  - `y` (Number) - Starting vertical coordinate.
  - `bw` (Number) - Width of brackets.
  - `bh` (Number) - Height of brackets.
  - `style` (String) - Bracket style: `'angular'`, `'serif'`, or `'rounded'`.
- **Logic:** Calculates left and right line limits and issues drawing path lines (`moveTo` / `lineTo` / `arcTo`) based on the selected bracket style.

---

### 3.5 DecorationModule
An IIFE module handling the creation, rendering, and hit testing of box highlights.

#### `makeDecoration(x, y, width, height)`
- **Purpose:** Creates a default box decoration object.
- **Arguments:** Start coordinates and dimensions.
- **Returns:** Serialization-friendly decoration object with a random unique ID.

#### `applyDefaults(dec)`
- **Purpose:** Sanitises imported decoration objects by populating missing property fields with default values.
- **Returns:** Sanitised decoration object.

#### `isOnBorder(x, y, dec)`
- **Purpose:** Determines if point `(x, y)` is on or near the outer border line of a box.
- **Returns:** Boolean.

#### `isInInterior(x, y, dec)`
- **Purpose:** Determines if point `(x, y)` lies inside the box boundaries.
- **Returns:** Boolean.

#### `getHandles(dec)`
- **Purpose:** Computes pixel coordinates for the eight resize handles (NW, N, NE, E, SE, S, SW, W).
- **Returns:** Key-value object mapping names to coordinates: `{ NW: { x, y }, ... }`.

#### `hitHandle(x, y, dec)`
- **Purpose:** Checks if coordinates `(x, y)` intersect any of the resize handles.
- **Returns:** Intersected handle name (String) or `null`.

#### `getLabelAnchor(dec)`
- **Purpose:** Calculates the coordinate, horizontal alignment, and vertical baseline for rendering a box's text label.
- **Returns:** Anchor object: `{ x: Number, y: Number, align: String, baseline: String }`.

#### `getLabelHitBox(ctx, dec, fontSize)`
- **Purpose:** Calculates bounding box coordinates and dimensions of the rendered label text for mouse hit detection.
- **Returns:** Rect object: `{ x, y, w, h }` or `null`.

#### `snapLabelToCorner(dropX, dropY, dec)`
- **Purpose:** Identifies the closest corner (top-left, top-right, bottom-left, bottom-right) to a dragged label's coordinates.
- **Returns:** Snapped corner identifier (String).

#### `draw(ctx, dec, isSelected)`
- **Purpose:** Renders the box fill, boundary strokes (dashed, dotted, solid), label text, and selection handles (if selected).

#### `hexToRgb(hex)`
- **Purpose:** Converts a hex colour string (e.g., `#ffff64`) to an RGB object.
- **Returns:** `{ r: Number, g: Number, b: Number }`.

---

### 3.6 NodeModule
An IIFE module enclosing the core syntactic node model.

#### `class Node`
Represents an individual node in the tree.
- **Methods:**
  - `constructor(x, y, showGrid)`: Instantiates a Node with snapped coordinates, unique ID, default text `'Node'`, empty features list, `'none'` style, and empty boundary caches.
  - `_getFontString(baseSize, styles)`: Formats font strings (e.g., `'italic bold 16px Arial'`) based on current styles.
  - `_getYOffset(baseSize, styles)`: Calculates vertical offsets for rendering subscripts and superscripts.
  - `_applyTextStyles(text, styles)`: Dummy placeholder returning text unmodified.
  - `_measureFormattedText(formattedText, baseSize, ctx)`: Calculates horizontal width and maximum height of formatted text by summing segment metrics.
  - `_calculateBounds(ctx)`: Calculates node height and width, incorporating the heights of multiple label lines and bracketed features.
  - `getBounds(ctx)`: Returns node bounds, retrieving from cache if input properties have not changed.
  - `_getCircleGeometry(ctx)`: Computes the centre coordinate and radius of a node styled as a circle.
  - `getConnectionPoint(isParent, ctx, targetX, targetY)`: Computes coordinates where branch lines connect. Returns perimeter points for circles, bottom centre for parents, and top centre for daughters.
  - `_drawFormattedText(ctx, formattedText, x, y, baseSize, isSelected)`: Draws a line of text on the canvas, handling strikethroughs, superscripts, subscripts, and font sizes.
  - `draw(ctx, isSelected, isDominanceSelected, bracketStyle)`: Renders selection shadows, shape borders (circles/boxes), label text lines, feature brackets, and feature text.
  - `isPointInside(x, y, ctx)`: Determines if point `(x, y)` lies within the node bounds.
  - `getEdgeIntersectionPoint(externalX, externalY, ctx)`: Computes intersection coordinates on the node's perimeter from an external point (used to end movement arrows cleanly at node borders).

---

### 3.7 ConnectionModule
An IIFE module enclosing dominance branch connections.

#### `class Connection`
Represents a dominance line (branch) connecting parent and child nodes.
- **Methods:**
  - `constructor(from, to)`: Initialises a connection pointing from parent node `from` to daughter node `to`.
  - `draw(ctx, isSelected)`: Renders a line between connection points of parent and child. Displays in blue when selected.
  - `distanceToPoint(x, y, ctx)`: Calculates distance from point `(x, y)` to the connection line.

---

### 3.8 MovementArrowModule
An IIFE module enclosing syntactic movement arrows.

#### `class MovementArrow`
Represents a curved or angular movement arrow with text annotations.
- **Methods:**
  - `constructor(fromNode, toNode, initialCurveDirection, ctx)`: Instantiates a movement arrow, sets default offsets, calculates default control point positions, and initialises styling options.
  - `_calculateInitialControlPoint(ctx, forCurveDirection)`: Computes default control point coordinates positioned perpendicular to the midpoint of the line connecting source and destination nodes.
  - `getPoints(ctx)`: Returns boundary coordinates of connected nodes and control points. Caches calculations.
  - `_getPointAtT(fromPoint, toPoint, controlPoint, t)`: Evaluates coordinate positions along the arrow at parametric parameter `t` (0.0 = start, 1.0 = end). Handles curved Bezier paths and two-segment angular paths.
  - `_getTangentAtT(fromPoint, toPoint, controlPoint, t)`: Computes the tangent vector direction at parameter `t`.
  - `drawAnnotation(ctx, isSelected)`: Draws the annotation text at parameter `t` along the path line with a perpendicular offset, drawing a light background for legibility.
  - `getAnnotationHitBounds()`: Returns bounding coordinates of the annotation text.
  - `draw(ctx, isSelected)`: Renders curved or angular path lines (solid, dotted, dashed), arrowheads, control points (if selected), and annotations.
  - `distanceToPoint(x, y, ctx)`: Computes the distance from coordinate `(x, y)` to the arrow path by evaluating 20 points along its line.
  - `isControlPointHit(x, y)`: Checks if mouse coordinate matches the arrow's Bezier control point.

---

### 3.9 DominanceModule
An IIFE module enclosing dominance tree queries.

#### `findDominatedNodes(rootNode, allConnections)`
- **Purpose:** Finds all nodes dominated by `rootNode`.
- **Arguments:**
  - `rootNode` (Node) - The parent node.
  - `allConnections` (Array) - List of all branches.
- **Returns:** Set of dominated nodes.
- **Logic:** Performs a queue-based breadth-first search traversing from parent to child connections, tracking visited nodes in a set.

---

### 3.10 LaTeXExportModule
An IIFE module converting tree structures to LaTeX `forest` syntax.

#### `convertFormatting(text)`
- **Purpose:** Replaces inline markdown formats with LaTeX macro equivalents (`\textbf`, `\textit`, `\st`, `\textsuperscript`, `\textsubscript`). Replaces newlines with `\\`.
- **Returns:** Converted LaTeX string.

#### `buildTree(nodeId, jsonData)`
- **Purpose:** Recursively parses a flat JSON list of connections and nodes into a nested tree structure.
- **Returns:** Tree node object: `{ node: Node, children: Array }`.

#### `toForest(treeNode, indent)`
- **Purpose:** Recursively converts a tree node structure to LaTeX `forest` code block formatting.
- **Returns:** LaTeX code string.

#### `convertToLaTeX(jsonData)`
- **Purpose:** Entry point for generating LaTeX code. Finds root nodes, builds tree hierarchies, maps them using `toForest`, and prepends comments about box decorations.
- **Returns:** LaTeX code block.

---

### 3.11 BracketParser Module
An IIFE module handling parsing, layout, and data translation of bracketed tree strings.

#### `parse(raw)`
- **Purpose:** Normalises brackets and comments, and parses a bracketed string into a nested tree node structure.
- **Arguments:** `raw` (String) - The raw bracket notation text.
- **Returns:** Nested node structure: `{ label: String, style: String, features: Array, children: Array, ... }`.
- **Logic:**
  - Replaces all parentheses with brackets and strips commented lines.
  - Parses nodes recursively.
  - Parses labels, optional style overrides (`|box` or `|circle`), feature lists (`{...}` blocks separated by semicolons), and cross-reference ID suffixes (`_ID`).
  - Converts bare words into leaf nodes.

#### `layout(tree, hSpacing, vSpacing)`
- **Purpose:** Calculates horizontal and vertical layout coordinates for a parsed tree structure.
- **Logic:**
  - `computeWidths(node)`: Traverses tree post-order. Leaves have width 1; parents have width equal to the sum of their children's widths.
  - `assignCoords(node, leftEdge, depth)`: Centers parents over the horizontal span of their children, setting vertical coordinates based on depth.

#### `toAppData(tree, offsetX, offsetY)`
- **Purpose:** Flattens a nested tree layout into arrays of nodes and connections for the canvas, applying spatial offsets.
- **Returns:** `{ nodes: Array, connections: Array }`.

---

### 3.12 BracketExporter Module
An IIFE module for exporting trees to bracket notation.

#### `escapeLabel(text)`
- **Purpose:** Escapes newlines and pipe characters.
- **Returns:** Escaped string.

#### `escapeFeature(text)`
- **Purpose:** Escapes semicolons.
- **Returns:** Escaped string.

#### `generateShortIds(count)`
- **Purpose:** Generates short alphabetical identifiers (`A` to `Z`, then `AA` to `AZ` etc.).
- **Returns:** Array of short IDs.

#### `exportTree(nodes, connections, movementArrows, decorations)`
- **Purpose:** Main tree export function.
- **Returns:** Object containing exported text and character span arrays: `{ text: String, spans: Array }`.
- **Logic:**
  - Map connections to child-parent lookup maps.
  - Resolves root nodes and isolated nodes.
  - Identifies nodes referenced by movement arrows or box decorations, assigning them short cross-reference IDs.
  - Renders a multi-line format if depth > 2 or if there are more than 4 nodes.
  - Performs depth-first traversal to generate brackets, tracking start and end indices of node tokens.
  - Appends comments and appends isolated nodes under a `% disconnected:` section.

---

### 3.13 BracketViewModule
An IIFE module managing the live navigation panel.

#### `create()`
- **Purpose:** Instantiates a navigation controller object exposing lifecycle methods.
- **Methods:**
  - `init(panelEl, contentEl)`: Saves DOM references.
  - `update(nodes, connections, movementArrows, selectedNodeId, decorations)`: Generates current bracket text using `BracketExporter.export`, clears the panel, inserts styled interactive `<span>` nodes for each bracket token, and scrolls the highlighted span into view.
  - `setOnSelectNode(fn)`: Registers a callback to execute when a bracket token is clicked.
  - `toggle()`: Collapses or expands the navigation panel.
  - `isVisible()`: Checks panel visibility.

---

### 3.14 SVGExporter Module
An IIFE module compiling the canvas components into raw SVG vector files.

#### `svgEsc(str)`
- **Purpose:** Replaces HTML entities (`&`, `<`, `>`, `"`) with safe XML representations.
- **Returns:** Escaped string.

#### `textToSVG(formattedText, baseSize, cx, cy, fill, fontFamily)`
- **Purpose:** Translates formatted text labels into SVG `<text>` elements, generating nested `<tspan>` sub-nodes for bold, italic, strikethrough, subscript, and superscript segments.
- **Returns:** SVG string.

#### `featureBrackets(cx, y, bw, bh, style)`
- **Purpose:** Generates SVG bracket outlines.
- **Returns:** SVG path string.

#### `generate(nodes, connections, movementArrows, canvasWidth, canvasHeight, ctx, fontFamily, featureBracketStyle, decorations)`
- **Purpose:** Generates the complete SVG markup.
- **Returns:** String containing the SVG file.
- **Logic:** Compiles a white background rect, box decorations, connection paths, movement arrows with arrowheads and annotations, and nodes with feature brackets and formatted text labels.

---

### 3.15 shortcutManager Module
An IIFE module managing user keybindings.

#### `init()`
- **Purpose:** Initialises shortcuts by loading configurations from `localStorage` under `jsSynTree_shortcuts`, falling back to default mappings.

#### `_persist()`
- **Purpose:** Serialises and writes current shortcut configurations to `localStorage`.

#### `match(event)`
- **Purpose:** Checks if a keyboard event matches a registered shortcut.
- **Returns:** Matching action name (String) or `null`.

#### `getAll()`
- **Purpose:** Returns the current shortcut settings object.

#### `setBinding(action, key, ctrl, shift, alt)`
- **Purpose:** Overrides the key configuration for an action and persists it.

#### `resetBinding(action)`
- **Purpose:** Restores an action's key configuration back to defaults and persists it.

#### `resetAll()`
- **Purpose:** Resets all actions to default key configurations and persists them.

#### `formatBinding(action)`
- **Purpose:** Formats shortcut settings into user-friendly strings (e.g., `Ctrl+Alt+S`).

#### `findConflicts()`
- **Purpose:** Scans all key configurations to identify actions bound to the same keystroke combinations.
- **Returns:** Set of conflicting action names.

---

### 3.16 SyntacticTreeApp Class
The core application controller that orchestrates state, canvas interactions, event listeners, and drawing.

#### `constructor()`
- Initializes application models (`stateManager`), keyboard shortcuts, and fetches DOM references for canvas.
- Configures default mode (`'add'`), states, internal clipboard, grid settings, and features.
- Registers DOM event listeners, canvas handlers, and sets up the live navigation panel.
- Renders the initial layout.

#### `snapToGridCoord(coord)`
- **Purpose:** Snaps logical coordinates to grid sizes.
- **Returns:** Snapped value (Number).

#### `getCanvasCoordinates(e)`
- **Purpose:** Maps browser client coordinates to logical coordinates.
- **Returns:** Coordinates object: `{ x, y }`.

#### `initializeCanvasSize()`
- **Purpose:** Synchronises the values in the toolbar canvas width and height input elements with current configurations.

#### `resizeCanvas()`
- **Purpose:** Resizes the canvas element. Scales backing store dimensions by device pixel ratio to maintain crisp rendering on HiDPI displays.

#### `updateCanvasSize()`
- **Purpose:** Validates values entered in the canvas size inputs (enforcing minimum boundaries), updates properties, and resizes the canvas.

#### `autoFitCanvas()`
- **Purpose:** Sizes the canvas boundaries to fit the layout of nodes and control points with padding. Adjusts node positions to avoid negative coordinate clipping.

#### `selectDominanceSubtree(rootNode)`
- **Purpose:** Selects a node and all descendants it dominates as a group.
- **Logic:** Calls `DominanceModule.findDominatedNodes`, populates `dominanceSelectedNodes`, sets primary selection, and redraws.

#### `clearSelection()`
- **Purpose:** Resets node, branch, movement arrow, and decoration selections. Hides contextual panels.

#### `hasDominanceSelection()`
- **Purpose:** Returns boolean indicating if a dominated group is selected.

#### `getCurrentStateObject()`
- **Purpose:** Serialises the canvas data and user configuration properties.
- **Returns:** State object containing nodes, branches, movement arrows, decorations, and settings.

#### `saveState()`
- **Purpose:** Pushes a copy of the current state onto the history stack and refreshes the undo/redo button states.

#### `restoreState(stateToRestore)`
- **Purpose:** Restores canvas elements and settings from a state object.
- **Logic:** Regenerates nodes, connections, movement arrows, and decorations, sanitising objects to ensure compatibility with older save formats. Re-sizes canvas and triggers a redraw.

#### `undo()`
- **Purpose:** Restores the canvas to the previous history state.

#### `redo()`
- **Purpose:** Restores the canvas to the next history state.

#### `updateUndoRedoButtonState()`
- **Purpose:** Disables or enables toolbar undo and redo buttons based on history availability.

#### `setMode(newMode)`
- **Purpose:** Switches the drawing mode. Resets mode-specific variables, toggles toolbar button styling, updates cursor icons, and redraws.

#### `initializeEventListeners()`
- **Purpose:** Binds DOM event listeners to toolbar buttons, canvas events (mouse clicks, drags, double clicks), settings inputs, and window key events.

#### `handleCanvasClick(e)`
- **Purpose:** Dispatches canvas click coordinates to handlers based on active drawing modes.

#### `handleAddNode(x, y)`
- **Purpose:** Spawns a new node at coordinates `(x, y)`, sets it as pending, and opens node properties modal.

#### `handleConnect(x, y)`
- **Purpose:** Handles dominance connection logic. Clicking a parent and then a child creates a connection branch.

#### `handleAddMovement(x, y)`
- **Purpose:** Handles movement arrow creation. Clicking a source and then a destination creates a curved or angular movement arrow.

#### `handleSelect(x, y, isDominanceModifier)`
- **Purpose:** Coordinates element selection. Ctrl-clicking a node initiates dominance subtree selection. Otherwise, checks for intersection with nodes, movement arrows, branches, or decoration borders.

#### `handleCanvasDoubleClick(e)`
- **Purpose:** Opens the node properties modal if a node is double-clicked in Select mode.

#### `handleCanvasMouseDown(e)`
- **Purpose:** Handles coordinate down-press actions. Starts box selection previews, node drags, annotation movements, control point adjustments, decoration resizing, or decoration drags.

#### `handleCanvasMouseMove(e)`
- **Purpose:** Handles mouse movements. Adjusts coordinates of dragged nodes (and dominated subtrees), Bezier control points, arrow annotations, resize boundaries, and decoration boxes using throttled updates.

#### `updateCursor(x, y)`
- **Purpose:** Updates the cursor style contextually depending on whether the mouse is hovering over handle control points, node labels, or borders.

#### `handleCanvasMouseUp()`
- **Purpose:** Ends mouse drag states, finalises box decoration scaling and node offsets, saves updated canvas state, and resets cursor styles.

#### `handleCanvasMouseLeave()`
- **Purpose:** Resets pointer cursor back to default when mouse exits canvas coordinates.

#### `handleKeyDown(e)`
- **Purpose:** Listens for shortcut bindings, modal controls, copy, cut, paste, delete, and cancel keys, and calls their actions.

#### `handleEscape()`
- **Purpose:** Cancels active drawing operations, connection branches, box previews, or selections.

#### `copyToClipboard()`
- **Purpose:** Serialises the selected node or dominance subtree and saves it to the internal clipboard.

#### `pasteFromClipboard()`
- **Purpose:** Deserialises components from the internal clipboard, offsetting coordinates by 40 pixels, assigning fresh unique IDs, selecting them, and saving state.

#### `cutToClipboard()`
- **Purpose:** Copies the selected elements to the clipboard and deletes them.

#### `deleteSelected()`
- **Purpose:** Deletes the selected node (and its descendants if subtree selected), branch, movement arrow, or box decoration.

#### `clearAll()`
- **Purpose:** Clears all canvas components after user confirmation.

#### `drawGrid()`
- **Purpose:** Draws vertical and horizontal grid lines on the canvas if grid display is toggled on.

#### `redraw()`
- **Purpose:** Clears the canvas, and redraws all layers (grid, box decorations, branches, movement arrows, and nodes) in correct order. Triggers navigation panel updates.

#### `exportCanvas()`
- **Purpose:** Clears background to white, temporarily hides grid lines and selection handles, redraws, and downloads the canvas contents as a PNG image.

#### `exportLaTeX()`
- **Purpose:** Converts the tree structure into LaTeX `forest` code and copies it to the system clipboard.

#### `exportSVG()`
- **Purpose:** Generates vector representation elements, packs them into an SVG string, and prompts download.

#### `exportBracket()`
- **Purpose:** Compiles the tree structure into bracketed notation and copies it to the system clipboard.

#### `openBracketModal()`
- **Purpose:** Opens the bracket notation import dialog and focuses input.

#### `parseBracketInput()`
- **Purpose:** Imports bracketed syntax. Parses input, runs layout, adds nodes and branches (offsetting if appending), and updates canvas.

#### `saveTreeToFile()`
- **Purpose:** Serialises the canvas state to a JSON file download.

#### `loadTreeFromFile(event)`
- **Purpose:** Reads a JSON file, parses its data, and calls `restoreState` to reload the tree onto the canvas.

#### `updateArrowPanel()`
- **Purpose:** Shows or hides the movement arrow property panel, populating inputs with the active arrow's properties.

#### `_initArrowPanel()`
- **Purpose:** Connects input change events in the arrow property panel to save and redraw handlers.

#### `updateDecorationPanel()`
- **Purpose:** Shows or hides the box decoration property panel, populating inputs with the active box's properties.

#### `_initDecorationPanel()`
- **Purpose:** Connects input change events in the box decoration property panel to save and redraw handlers.

#### `openPropertiesModal()`
- **Purpose:** Opens the node properties dialog, loading text, features, and style configurations.

#### `closeTextModal()`
- **Purpose:** Closes the node properties modal. Cancels pending nodes if they were newly created.

#### `saveNodeProperties()`
- **Purpose:** Saves changes from the node modal to the node object, recalculates bounds, saves state, and redraws.

#### `openSettingsModal()`
- **Purpose:** Opens the global settings dialog.

#### `closeSettingsModal()`
- **Purpose:** Closes the settings modal.

#### `_buildSettingsTable()`
- **Purpose:** Populates the shortcuts table in the Settings modal with active bindings and record/reset buttons.

#### `_updateSettingsTable()`
- **Purpose:** Updates the settings table UI and highlights shortcut conflicts.

#### `_startRecording(action)`
- **Purpose:** Configures keyboard capture to record new shortcut bindings.

---

### 3.17 Global UI Helper Functions

These helper functions are defined outside classes to interact with inline HTML elements:

#### `insertFormatting(textareaId, openTag, closeTag)`
- **Purpose:** Wraps selected text in a textarea with markdown tags (e.g., `**`). Injects tags at cursor position if no text is selected.

#### `applySizeFormatting(textareaId, fontSize)`
- **Purpose:** Wraps selected text with size override tags: `[SIZE=fontSize]...[/SIZE]`.

#### `closeTextModal()`
- **Purpose:** Global bridge calling `app.closeTextModal()`.

#### `saveNodeProperties()`
- **Purpose:** Global bridge calling `app.saveNodeProperties()`.

#### `closeBracketModal()`
- **Purpose:** Closes the bracket notation modal dialog.

#### `closeSettingsModal()`
- **Purpose:** Global bridge calling `app.closeSettingsModal()`.

#### `parseBracketInput()`
- **Purpose:** Global bridge calling `app.parseBracketInput()`.
