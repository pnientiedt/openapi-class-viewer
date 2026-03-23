# OpenAPI Class Viewer — Design Spec

**Date:** 2026-03-23
**Project:** fft-classes
**Status:** Approved

---

## Overview

A static React website that serves as an interactive reference for the fulfillmenttools Java model classes. Developers and new team members can browse all 70 domain entities, view class diagrams, and drill into complex types by clicking fields. Diagrams are fully interactive — nodes are draggable, edges route dynamically with bezier curves, and multiple related classes can be expanded on the canvas simultaneously.

---

## Audience

Both internal developers (integration reference while coding) and new team members (onboarding/understanding the model structure).

---

## Tech Stack

| Concern | Choice |
|---|---|
| Framework | React 18 + Vite 5 |
| Diagrams | React Flow v11 |
| Data | Parse `api.swagger.yaml` → `model.json` at build time |
| Styling | Plain CSS (dark theme) |
| Deployment | Static `dist/` — no server required |
| Data script | Python + PyYAML (`parse-swagger.py`) |

**Runtime dependencies:** `react`, `react-dom`, `reactflow` only. No backend.

---

## Data Pipeline

A Python script (`website/scripts/parse-swagger.py`) reads `../api.swagger.yaml` and writes `website/src/data/model.json`. Run once at setup; re-run when the spec changes. The script is idempotent.

### `model.json` schema

```json
{
  "classes": {
    "PickJob": {
      "description": "Represents a pick job in the fulfillment process.",
      "fields": [
        { "name": "facilityRef",   "type": "String",         "kind": "primitive", "array": false },
        { "name": "status",        "type": "PickJobStatus",  "kind": "enum",      "array": false },
        { "name": "pickLineItems", "type": "PickLineItem",   "kind": "complex",   "array": true  }
      ]
    },
    "PickJobStatus": {
      "kind": "enum",
      "values": ["OPEN", "IN_PROGRESS", "CLOSED", "CANCELLED"]
    }
  },
  "domains": {
    "Fulfillment":    ["PickJob", "PickRun", "PackJob", "HandoverJob", "StowJob", "InboundProcess"],
    "Commerce":       ["Order", "Article", "Listing", "Category", "Brand"],
    "Infrastructure": ["Facility", "Carrier", "LoadUnit", "InterfacilityConnection"],
    "Returns":        ["ItemReturnJob", "InboundReceipt"],
    "Configuration":  ["DomainConfiguration", "GdprConfiguration", "Fence"],
    "Common":         ["...all classes not assigned to another domain"]
  }
}
```

`"Common"` is populated by the parser with any class not explicitly listed in another domain group.
```

Field `kind` values: `"primitive"` (String, Integer, Boolean, OffsetDateTime, Object), `"complex"` (another class), `"enum"` (inner or top-level enum).

---

## Project Layout

```
website/
  scripts/
    parse-swagger.py        ← generates src/data/model.json
  src/
    data/
      model.json            ← generated, committed to repo
    components/
      Sidebar.jsx           ← domain groups + entity list + search
      DiagramCanvas.jsx     ← React Flow wrapper
      ClassNode.jsx         ← custom expandable class node
      EnumNode.jsx          ← custom enum node (amber)
    hooks/
      useDiagram.js         ← node/edge/history state
    App.jsx                 ← root layout (sidebar + canvas)
    index.css               ← global dark theme styles
  index.html
  package.json
  vite.config.js
```

---

## UI Layout

Single-page app with two regions:

- **Left sidebar (220px, fixed):** collapsible domain groups → entity list → search bar at top. Active entity highlighted with left border accent.
- **Right canvas (flex-1):** React Flow diagram area. Breadcrumb bar at the top of the canvas tracks drill-down history.

### Sidebar behaviour

- Domain group headers are clickable to collapse/expand their entity list
- Search filters entity names across all domains live
- Clicking an entity resets the canvas and loads that entity as the focus node
- Each entity shows its name and sub-class count

---

## Diagram Canvas

React Flow handles pan (drag), zoom (scroll), and node dragging out of the box. Edges use smooth bezier curves and re-route automatically when nodes are moved.

### Node types

**ClassNode** (regular classes)

| Role | Border | Header background |
|---|---|---|
| Focus (entry point) | `#6366f1` | `#6366f1` (purple) |
| Expanded related | `#10b981` | `#059669` (green) |
| Context (ancestor, collapsed) | `#2d2d44` | `#1e1e2e` (grey, 40% opacity) |

- Header shows class name + collapse/expand toggle
- Body lists fields as `name : Type` — primitive types in grey, complex/enum types in purple + underlined
- Clicking a complex/enum field adds that class as a new node on the canvas and draws an edge
- If the class is already on canvas, only the edge is added (no duplicate nodes)
- Node body is collapsible — collapsed state shows field count only

**EnumNode**

- Amber header (`#d97706`) with `«enum»` badge
- Lists all constant values in amber text
- Connected via dashed amber edges from referencing fields

### Edge types

- **Solid bezier** — class → class relationships (purple or green tint)
- **Dashed bezier** — class → enum relationships (amber tint)

### Breadcrumb

Tracks the chain of clicked drill-downs: `All Entities / PickJob / PickLineItem`. Clicking any crumb reloads that entity as the focus node.

### Canvas controls

- Drag to pan, scroll to zoom (React Flow default)
- Nodes are freely draggable; edges re-route dynamically
- "Reset layout" button re-arranges all visible nodes to their default computed positions

---

## Interactions

1. **Open entity** — click entity in sidebar → canvas resets, entity loads as purple focus node (expanded)
2. **Drill into type** — click a complex/enum field → new node appears on canvas, bezier edge drawn from source field's node
3. **Expand multiple** — clicking several fields of the same node opens all referenced types on canvas simultaneously, each with its own edge
4. **Collapse/expand node** — toggle button in node header hides/shows fields
5. **Move nodes** — drag any node; edges follow automatically
6. **Navigate back** — click breadcrumb segment resets the canvas to that entity as the sole focus node (same as clicking it in the sidebar); nodes expanded after that point are discarded
7. **Duplicate edges** — if a field pointing to a class already on canvas is clicked again, the action is ignored (no duplicate edge drawn)

---

## Visual Theme

Dark canvas throughout:

```
Canvas background:  #1a1a2e
Sidebar background: #12121f
Node body:          #1e1e2e
Node borders:       per role (see above)
Text (fields):      #cbd5e1
Primitive types:    #64748b
Complex types:      #a78bfa  (purple, underlined, clickable)
Enum values:        #fbbf24  (amber)
```

---

## Build & Development

```bash
# First-time setup
cd website
npm install
python scripts/parse-swagger.py    # generates src/data/model.json

# Development
npm run dev                         # Vite dev server → http://localhost:5173

# Production
npm run build                       # outputs to website/dist/
```

The `dist/` folder is fully static and can be opened directly as `index.html` or deployed to any static host (Nginx, GitHub Pages, S3, etc.).

---

## Out of Scope

- Authentication / access control
- Backend / API server
- Editing or annotating the model
- Showing API endpoints (only model classes)
- Search within field values or descriptions
