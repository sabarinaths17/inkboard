# inkboard — Claude context

Single-file whiteboard canvas app: `inkboard.html`. All logic, styles, and markup are in that one file.

## Architecture

- **State**: `const S = { objects, view, tool, color, sz, bg, sel, selSet, name, pages, pageIdx }` — single global state object
- **Rendering**: canvas-based, `paint()` → `drawObj()` per object; selection overlays drawn after
- **History**: simple array-based undo/redo via `pushUndo()` / `hist.u` / `hist.r`
- **Persistence**: auto-save to `localStorage` on every change (saves all pages); file system API (`dirHandle`) for folder saves; two export formats:
  - **PNG** — visual export that also embeds the full board JSON (all pages) invisibly in the file metadata; re-importing restores a fully editable multi-page board
  - **`.inkboard.json`** — JSON snapshot of `{ app, v, name, pages, pageIdx }`; two save options in the Save menu:
    - **All pages** — full board including every page
    - **Current page only** — single-page snapshot wrapped in the same format (`pages` array with one entry, `pageIdx: 0`)
- **Theme**: defaults to light mode on first open; `setTheme(dark, save?)` toggles UI; board bg follows separately via `setBg()`

## Object model

Each object in `S.objects` has a `type` field: `pen | rect | ellipse | line | arrow | text`. Objects are plain JS objects — no classes. `clone()` deep-copies them (note: `srcConn`/`dstConn` object refs are lost on JSON round-trip), `moveObj()` translates them, `bbox()` returns their bounding box.

## Selection model

- `S.sel` — single selected object (for resize handles, property editing)
- `S.selSet` — Set of objects for multi-selection
- When `selSet.size > 1`, `sel` is null; when `selSet.size === 1` or 0, `sel` is the primary (or null)
- `selArr()` returns `[...selSet]` if multi, else `[sel]` if single, else `[]`
- Multi-select operations: delete, copy, paste, duplicate, move all use `selArr()`

## Tools

`select | pen | eraser | rect | ellipse | arrow | line | text | hand`

Keybindings: V P E R O A L T H respectively.

## Keyboard shortcuts implemented

| Shortcut | Action |
|---|---|
| Ctrl+Z / Ctrl+Y | Undo / Redo |
| Ctrl+A | Select all |
| Ctrl+C / Ctrl+V | Copy / Paste selected |
| Ctrl+D | Duplicate selected |
| Ctrl+S / Ctrl+O | Save PNG / Open board |
| Ctrl+0 | Reset view |
| Ctrl+Delete | Clear board |
| Delete / Backspace | Remove selected |
| Alt+drag | Drag a copy |
| [ / ] | Change tool size |
| 1–6 | Set colour |
| Space | Hold to pan |
| Tab | Presenter mode |
| Esc | Deselect / exit presenter |
| ? | Help |

## Drag modes

`drag.mode`: `pan | draw | shape | erase | move | resize | marquee | connect`

- `pan`: dragging empty canvas in select mode (or space+drag, middle-mouse, hand tool) pans the view; cursor shows `grab`/`grabbing`; two-finger trackpad scroll also pans (wheel event without ctrlKey); Ctrl+scroll or pinch zooms via `zoomAt()`
- `marquee`: rubber-band rect drawn on canvas while dragging empty space with select tool; on release selects all objects whose bbox overlaps the rect (legacy — currently empty-space drag starts pan instead)
- `move` with `drag.moveAll`: set when moving within a multi-selection; each item's original position stored in `drag.moveAll` as `{ obj, snap }`
- `connect`: started by dragging from a port handle on any rect (no pre-selection needed); draws a draft arrow from that port; snaps endpoint to nearest port on any other rect within 40px; on release creates an arrow object if long enough (>8px); dropped on empty space auto-spawns a new node in the arrow's travel direction

## Port handles

Port handles are the 4 blue dots at the N/E/S/W midpoints of a selected `rect` object, visible only when the select tool is active. Used to drag out arrow connections (Obsidian Canvas style).

- `portsFor(o)` — returns `[{id, x, y}]` for the 4 midpoints of `o`'s bbox
- `portAt(p)` — scans **all** rects for a port within tolerance 10/k; returns `{id, x, y, obj}` or null; tool must be `'select'`
- `paintPorts(o)` — draws the 4 blue filled circles with white stroke
- Cursor changes to `crosshair` when hovering any port regardless of current selection
- Multiple arrows can originate from the same port; clicking a port starts a new connect drag even when the source rect isn't selected

## Curved arrows

Arrow objects support an optional `cx, cy` control point (quadratic bezier). When absent the arrow is straight.

- `arrowD(o)` — draws `Q cx cy` bezier path when control point exists; arrowhead tangent computed from (cx,cy)→(x2,y2)
- `handlesFor(o)` for `arrow` type — adds `{id:'mid'}` handle at the control point (or geometric midpoint when straight)
- Drag the mid handle (solid blue circle) to bend the arrow; drag it back near the geometric midpoint to straighten
- `applyResize` h='mid' — sets `o.cx/cy` or deletes them when within 10px of straight midpoint
- `hitEdge` for curved arrows — samples 20 points along the bezier
- `bbox` includes cx,cy so selection/marquee works on curved arrows
- `moveObj` propagates cx,cy offset when moving arrows
- `nearestPortSnap(p, exclude)` — returns nearest `{id, x, y, obj}` port within 40 world-px on any rect except `exclude`, or null
- During connect drag, endpoint snaps live to nearest port and a blue glow ring highlights the target rect
- Arrows created via port-snap store `arrow.srcConn = { obj, port }` and `arrow.dstConn = { obj, port }` as direct object references
- **Reconnecting existing arrows**: dragging an arrow's p1/p2 handle also snaps to ports (same glow ring); on release sets/clears `srcConn`/`dstConn` for that endpoint
- `updateConnectedArrows(movedObjs[])` — called during move drag; scans all arrows and updates x1/y1 or x2/y2 for any whose srcConn/dstConn points to a moved rect; uses `portsFor()` to get current port positions
- Auto-spawn direction: new node appears in the direction the arrow was dragged; `drag.srcPort` determines offset direction and opposite dstConn port (`OPP = {e:'w', w:'e', n:'s', s:'n'}`)

## Markdown text rendering

Text objects render Obsidian-style inline markdown via `drawMdText(c, o)`:

| Syntax | Rendered |
|---|---|
| `- text` or `. text` | `• text` (bullet) |
| `[ ] text` | unchecked checkbox □ + text |
| `[x] text` | checked checkbox ✓ + dimmed text |

- `drawMdText(c, o)` — called from `drawObj` text case; splits on `\n`, renders each line with its markdown prefix
- Checkbox square size: `cs = round(o.size * 0.72)`; positioned vertically centered on the line
- `checkboxLineAt(o, p)` — returns line index if world point `p` hits the checkbox symbol area of a `[ ]`/`[x]` line, else -1
- `toggleCheckbox(o, li)` — flips `[ ]` ↔ `[x]` on line `li`; calls `pushUndo()` + `changed()`
- Single click on a checkbox symbol in select mode toggles it without entering edit mode
- `measure(o)` accounts for checkbox/bullet width offsets in bbox calculation

## Clipboard

`clipboard` is an array of cloned objects (supports multi-select copy/paste). Paste offsets each by 24px.

## Templates

Templates panel is in the topright bar ("Templates" chip). Two templates available:

### Kanban Column
- Single empty column (200×420) — no pre-filled cards
- Column outline (rect) + "Column" title text + horizontal divider line; all 3 share a `group: 'kb-...'` id so they move together
- Colors: accent blue (`#3b6cf6`) for column chrome
- Dragging an arrow from any kanban port to empty space auto-spawns a new linked kanban column in that direction
- Double-clicking inside the kanban body (below the divider at y+44) creates a text element at that position
- `makeKanban(cx?, cy?)` accepts optional world-coordinate center; defaults to `viewCenter()`

### Mind Map Node
- Single rectangular node (160×56) + "Main Topic" text — no pre-built branches
- Both objects share a `group: 'mm-...'` id; moving the rect moves the text with it
- Rect shows 4 connection ports (N/E/S/W midpoints) when selected with the select tool; drag from a port to create an arrow
- Dropping an arrow on empty space auto-spawns a new linked node in the arrow's travel direction; new node's facing port aligns with the arrow tip
- `makeMindmap(cx?, cy?)` accepts optional world-coordinate center; defaults to `viewCenter()`

### Stamp logic (`stampTemplate(objs)`)
- Pushes undo snapshot, adds all objects, selects them all via `selSet`, switches to select tool, shows toast.
- Colors in template objects are resolved at stamp time — they won't auto-adapt if the board theme changes later.

## Multi-page support

Each board can hold multiple pages (slides). `S.pages` is the array; `S.pageIdx` is the index of the active page. `S.objects` and `S.bg` always hold the **live values** for the current page — they are synced to/from `S.pages[S.pageIdx]` via `syncPageState()`.

- Each page: `{ id: 'pg-<timestamp>', name: string, bg: string, objects: [] }`
- `syncPageState()` — writes `S.bg` and `S.objects` back into `S.pages[S.pageIdx]`; must be called before switching pages or saving
- `switchPage(idx)` — syncs current, swaps `S.objects`/`S.bg`, resets selection and undo history, re-renders
- `addPage()` — appends a new blank page, switches to it
- `deletePage(idx)` — removes a page (blocked if only one remains); switches to nearest neighbour
- `renderPageBar()` — rebuilds the bottom tab bar DOM from `S.pages`
- `togglePageBar()` — toggles `body.bar-off` class; bar slides out via `translateY(100%)`; a persistent `#page-restore` button (`Pages ▴`) appears at the bottom-left when the bar is hidden

### Page rename
Double-click any page tab to rename it via a `prompt()` dialog. The new name is stored in `S.pages[i].name` and the bar re-renders.

### Page bar layout
- `#page-bar` — fixed bottom strip; has `padding-right: 200px` to clear the Netlify badge
- `#page-toggle` button is on the **left** of the bar (▾ hides, ▴ shows)
- `#page-add` button (+) is on the right of the tab list
- `#page-restore` — fixed bottom-left; shown only when `body.bar-off` is set; clicking it restores the bar

### File format
`payload()` always returns `{ app: 'inkboard', v: 1, name, pages, pageIdx }`. `loadFile()` handles both:
- New format: `data.pages` array present → loaded directly
- Legacy format: `data.objects` present → wrapped in a single-entry `pages` array

## About modal

An About dialog accessible via the **About** button in the top-right bar. Same overlay pattern as the help modal.

- `#about` — fixed full-screen overlay, `display:grid; place-items:center`; hidden by default (`hidden` attribute)
- `.aboutcard` — glass card, `width:min(680px,calc(100vw-32px))`, scrollable
- `openAbout()` / `closeAbout()` — toggle the `hidden` attribute
- Closes on: × button (`#aboutClose`), Escape key, clicking the backdrop
- Content: tagline, six capability sections (Drawing Tools, Connections, Multi-page, Save & Export, Templates, Workflow), footer note on auto-save and AI-generated boards

## Color system

Named colors (`'ink'`, `'accent'`, etc.) resolved at draw time via `resolveColor()`. Swatches in `SWATCHES[]`.
`inkNow()` returns the current ink hex (white on dark canvas, black on light) — used in templates to match the board background at stamp time.
