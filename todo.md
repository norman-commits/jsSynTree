I have everything I need. Here is a precise action plan.

---

# Action Plan: Semantic JSON Export — `jsSynTree_v2_2.html`

## Working Rules
- New file `jsSynTree_v2_2.html`. Do not overwrite v2.1.1.
- Read the full current source before starting.
- All changes must pass: existing JSON saves load correctly; undo/redo works; browser console clean on load.

---

## Task 1 — Add `movementType` to `MovementArrow`

**In `MovementArrowModule`, class `MovementArrow` constructor**, add one new field after the existing session 7 fields:

```javascript
this.movementType = '';   // '' | 'A' | 'A-bar' | 'Head' | 'Other'
```

**In `getCurrentStateObject()`**, add `movementType` to the arrow serialisation:

```javascript
movementType: arrow.movementType || ''
```

**In `restoreState()`**, add safe fallback when restoring arrows:

```javascript
arrow.movementType = aData.movementType || '';
```

Do the same in the clipboard paste code in `pasteFromClipboard()` where `Object.assign(arrow, {...})` is called.

---

## Task 2 — Add `movementType` dropdown to the arrow property panel

**In the HTML**, inside `#arrowPropertyPanel`, add a new `<label>` block after the "Line shape" label and before the "Annotation" label:

```html
<label>Movement type
  <select id="apMovementType">
    <option value="">— unspecified —</option>
    <option value="A">A</option>
    <option value="A-bar">A-bar</option>
    <option value="Head">Head</option>
    <option value="Other">Other</option>
  </select>
</label>
```

**In `updateArrowPanel()`**, add:

```javascript
document.getElementById('apMovementType').value = arrow.movementType || '';
```

**In `_initArrowPanel()`**, add a change listener:

```javascript
document.getElementById('apMovementType').addEventListener('change', e => {
    if (!this.selectedMovementArrow) return;
    this.selectedMovementArrow.movementType = e.target.value;
    this.saveState();
});
```

---

## Task 3 — Add `SemanticExportModule`

Add a new IIFE module alongside the existing `LaTeXExportModule` and `BracketExporter`. It takes the raw state object from `getCurrentStateObject()` and produces the semantic JSON. It has no knowledge of visual properties.

```javascript
const SemanticExportModule = (() => {

    function exportSemantic(stateObj) {
        const { nodes, connections, movementArrows, decorations } = stateObj;

        // Build child map to determine which nodes fall inside decoration bounds
        const semanticNodes = nodes.map(n => {
            const obj = {
                id: n.id,
                label: n.text,
            };
            if (n.features && n.features.length > 0) obj.features = [...n.features];
            if (n.style && n.style !== 'none') obj.style = n.style;
            return obj;
        });

        const dominance = connections.map(c => ({
            parent: c.fromId,
            child: c.toId
        }));

        const movement = movementArrows.map(a => {
            const obj = {
                from: a.fromId,
                to: a.toId,
            };
            if (a.movementType) obj.type = a.movementType;
            if (a.annotation) obj.annotation = a.annotation;
            return obj;
        });

        // Decorations: use label as semantic role
        // Determine which node IDs fall within each decoration's bounds
        const domains = decorations
            .filter(d => d.label && d.label.trim() !== '')
            .map(d => {
                const enclosed = nodes
                    .filter(n => n.x >= d.x && n.x <= d.x + d.width &&
                                 n.y >= d.y && n.y <= d.y + d.height)
                    .map(n => n.id);
                return {
                    role: d.label.trim(),
                    nodes: enclosed
                };
            });

        const output = {
            schema: 'marq-tree-v1',
            nodes: semanticNodes,
            dominance,
        };

        if (movement.length > 0) output.movement = movement;
        if (domains.length > 0) output.domains = domains;

        return JSON.stringify(output, null, 2);
    }

    return { exportSemantic };
})();
```

---

## Task 4 — Add Export Semantic JSON button to toolbar

**In the HTML toolbar**, add a button alongside the existing export buttons:

```html
<button id="exportSemanticBtn" style="background:#607D8B;">Export Semantic</button>
```

**In the app initialisation** where other toolbar buttons are wired up, add:

```javascript
document.getElementById('exportSemanticBtn').addEventListener('click', () => {
    const state = this.getCurrentStateObject();
    const json = SemanticExportModule.exportSemantic(state);
    navigator.clipboard.writeText(json).then(() => {
        alert('Semantic JSON copied to clipboard.');
    }).catch(() => {
        prompt('Copy this semantic JSON:', json);
    });
});
```

---

## Task 5 — Update version string

In `getCurrentStateObject()`, change:

```javascript
version: '2.2'
```

---

## Verification Checklist

- Load a v2.1 JSON save — confirms backward compatibility and `movementType` falls back to `''` cleanly.
- Draw a tree with movement arrows. Select an arrow — movement type dropdown appears in the panel. Set a type, save, reload — type persists.
- Draw a decoration with a label. Export Semantic JSON — decoration appears in `domains` array with correct `role` and enclosed node IDs.
- Draw a tree with no arrows. Export Semantic JSON — `movement` key is absent from output entirely (not an empty array).
- Browser console clean on load.