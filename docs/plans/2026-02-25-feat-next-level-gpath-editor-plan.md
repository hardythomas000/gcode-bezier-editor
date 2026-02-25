---
title: "feat: Next-Level G-Path Editor"
type: feat
date: 2026-02-25
deepened: 2026-02-25
---

# Next-Level G-Path Editor

## Enhancement Summary

**Research agents used:** Canvas 2D Performance, Frontend Design UX, Web Audio API, G-code Editor Patterns, Single-File Architecture

### Key Architecture Decisions (from research)
1. **3-layer canvas stack** (grid / toolpath / interaction) — only redraw dirty layers. 100x fewer draw calls during mouse movement.
2. **Event bus + revealing module pattern** — all modules communicate via pub/sub, not direct calls. Single `App` namespace.
3. **3-tier state** — `doc` (undo-able), `view` (persisted, not undo-able), `ui` (transient). Undo operates ONLY on `doc`.
4. **structuredClone snapshots** for undo — 0.1-0.3ms for 500 segments. Simple, fast enough.
5. **Spatial hash grid** for hit-testing — O(1) instead of O(n) per mouse move.
6. **Batch draw calls by style** — one `beginPath/stroke` per color group, not per segment.

## Current State → Target

| | Now | Target |
|---|---|---|
| Lines | ~1060 | ~2500-3000 |
| Features | Drag-edit, audio, code panel | Full editor + simulation |
| Architecture | Global `S`, single canvas | Event bus, 3 canvas layers, modules |
| Undo | None | 50-step structuredClone stack |
| File I/O | Paste textarea only | Drag-drop, open, save, clipboard |
| Drawing | None | Click-to-draw G1, drag-to-draw arcs |
| Selection | Single segment | Multi-select, marquee, group transform |
| Simulation | Auto-loop dot | Play/pause/scrub + material removal |
| Stats | Segment count + length | Full machining time, feeds, bounds |

## Implementation Plan

### Phase 1: Architecture Refactor + Foundation

**Refactor the existing 1060 lines into the module architecture FIRST**, then add features on top.

#### 1.1 Module Architecture

Restructure into 18 sections (dependencies flow downward):

```
CSS → HTML → Constants → Bus → State → Undo → Parser → FileIO →
Render → Geometry → Interaction → Audio → Simulation → DrawMode →
SegmentOps → CommandPalette → UIComponents → Init
```

**Event Bus** (nervous system):
```js
const Bus = (() => {
    const listeners = {};
    return {
        on(e, fn) { (listeners[e] ||= []).push(fn); },
        off(e, fn) { listeners[e] = (listeners[e]||[]).filter(f=>f!==fn); },
        emit(e, d) { (listeners[e]||[]).forEach(fn => fn(d)); }
    };
})();
```

**State** (3-tier):
```js
const State = ((bus) => {
    let S = {
        doc: { lines: [], segs: [], selectedIndices: new Set() },
        view: { px: 0, py: 0, zoom: 1, snap: false, sound: true, mode: 'select' },
        ui: { hover: null, drag: null, simT: 0 }
    };
    // ... controlled mutation with bus.emit('state:changed') ...
    return { get: () => S, update, batch };
})(Bus);
```

#### 1.2 Three-Layer Canvas Stack

```html
<div id="viewport" style="position:relative">
    <canvas id="layer-grid"></canvas>     <!-- z:0, redraws on pan/zoom only -->
    <canvas id="layer-toolpath"></canvas>  <!-- z:1, redraws on data change -->
    <canvas id="layer-interact"></canvas>  <!-- z:2, redraws on hover/select/animate -->
</div>
```

Dirty flag system — only redraw layers that changed:
- `grid` dirty on: pan, zoom, resize, theme change
- `toolpath` dirty on: data edit, undo/redo, file load, feed map toggle
- `interaction` dirty on: hover, select, drag, animation tick

#### 1.3 Undo/Redo (structuredClone snapshots)

- Snapshot `S.doc` (excluding view/ui) before each meaningful action
- Max 50 states, circular buffer
- `Ctrl+Z` / `Ctrl+Shift+Z`
- Undo/Redo toolbar buttons with disabled state at boundaries
- SFX on undo/redo action

#### 1.4 File Import/Export

- **Drag & Drop**: `.nc/.gcode/.tap/.ngc` onto canvas with glowing drop zone
- **Open**: `<input type="file">` triggered by button or `Ctrl+O`
- **Save**: `Blob` + `URL.createObjectURL` + `<a download>` via `Ctrl+S`
- **Copy**: Clipboard API button in code panel header
- **localStorage auto-save**: Save on every edit, restore on load

#### 1.5 Keyboard Shortcuts

Register all shortcuts through a central keybinding registry (used by command palette later):

| Key | Action | Category |
|-----|--------|----------|
| `Ctrl+Z` | Undo | History |
| `Ctrl+Shift+Z` | Redo | History |
| `Ctrl+S` | Save .nc | File |
| `Ctrl+O` | Open file | File |
| `Ctrl+K` | Command palette | UI |
| `Delete` | Delete selected | Edit |
| `Arrows` | Nudge point (1 grid step) | Edit |
| `Shift+Arrows` | Nudge point (10x) | Edit |
| `Escape` | Deselect / exit mode | Edit |
| `F` | Fit view | View |
| `G` | Toggle snap | View |
| `H` | Toggle feed heat map | View |
| `Space` | Play/pause simulation | Sim |
| `Tab` | Next segment | Nav |
| `Shift+Tab` | Prev segment | Nav |
| `D` | Enter draw mode | Tool |
| `V` | Select mode | Tool |
| `?` | Show shortcuts help | UI |

#### 1.6 Pinch Zoom

Track two-finger distance + midpoint for simultaneous zoom + pan. Use `touch-action: none` on canvas.

---

### Phase 2: Segment Operations + Draw Mode

#### 2.1 Delete Segment
- Select → Delete key → remove from `S.doc.lines[]`
- Reconnect adjacent segments (next segment's start = deleted segment's start)
- Push undo snapshot before delete

#### 2.2 Draw Mode (tldraw-inspired state machine)
- Enter via `D` key or toolbar button
- **State machine**: `DrawMode.Idle` → `DrawMode.Placing` → `DrawMode.Confirming`
- **Click**: Add G1 from last endpoint to click position
- **Click + drag**: Add G2/G3 arc (drag direction = CW/CCW, drag distance = bulge)
- **Ghost preview**: Semi-transparent line/arc following cursor
- **Double-click / Enter / Escape**: Exit draw mode
- Cursor changes to crosshair, toolbar button highlighted
- SFX: drawVertex on each point placed, drawClose on mode exit

#### 2.3 Context Menu
- Right-click segment → popup: Delete / Duplicate / Convert to G1/G2/G3 / Insert After
- Right-click empty → Paste / Select All
- Long-press on mobile = right-click
- Keyboard shortcut hints next to each item

#### 2.4 Segment Type Conversion
- G1→G2: compute I/J from perpendicular offset at midpoint
- G2→G1: drop I/J values
- G2↔G3: flip I/J signs

---

### Phase 3: Visual Enhancements

#### 3.1 Feed Rate Heat Map
- Toggle via `H` key or toolbar "Heat" button
- Colorblind-friendly palette: **viridis** (purple → blue → green → yellow)
- Legend bar at canvas bottom showing min/max F values
- Hover tooltip shows exact feed rate per segment

#### 3.2 Tool Diameter / Kerf Visualization
- Sidebar input: "Tool ⌀" (default 6mm)
- Toggle "Kerf" button → semi-transparent band along toolpath
- `lineWidth = toolDia * zoom`, `globalAlpha = 0.15`
- `lineCap = 'round'` for ballnose effect

#### 3.3 Machining Stats Panel
Replace simple status bar with:
```
┌─────────────────────┐
│ 15 segments         │
│ G0: 3  G1: 8  Arcs: 4 │
│ Cut: 234.5mm        │
│ Rapid: 45.2mm       │
│ Time: ~0:42         │
│ Bounds: 40 × 30mm   │
└─────────────────────┘
```
Time estimate: sum(seg.length / seg.feed) + sum(rapid.length / 5000mm/min)

#### 3.4 Direction Arrows
- Small arrowheads every ~20px (screen space) along each segment
- Green entry arrow at program start, red exit at end
- Arrow spacing adapts to zoom (fewer when zoomed out)

#### 3.5 Origin Marker
- Crosshair + small circle at X0 Y0
- "WCS" label in small text
- Semi-transparent, always visible through toolpaths

---

### Phase 4: Multi-Select + Group Transform

#### 4.1 Selection Mechanics (Figma-inspired)
- Click = single select
- Shift+Click = toggle in/out of selection
- Ctrl+A = select all
- Bounding box shown around all selected segments

#### 4.2 Marquee Selection
- Click+drag on empty space = selection rectangle
- Segments intersecting rectangle get selected
- Shift+marquee = additive

#### 4.3 Group Transform
- Drag selected group to translate all
- Store original positions at drag-start, apply delta (avoid rotation drift per tldraw pattern)
- Context menu: Mirror X / Mirror Y

---

### Phase 5: Simulation Upgrades

#### 5.1 Playback Controls
- Play / Pause / Stop buttons (bottom bar or sidebar)
- Speed: 0.25x / 0.5x / 1x / 2x / 4x / 8x (dropdown or slider)
- Scrub bar: thin progress bar, drag to seek
- Current line highlighted in code panel during playback

#### 5.2 Material Removal (2D Bitmap)
- Toggle "Stock" button
- Render stock as filled rect on a dedicated offscreen canvas
- Use `globalCompositeOperation: 'destination-out'` with `lineWidth = toolDia * zoom` and `lineCap = 'round'`
- As traveling dot advances, erase material behind it
- Reset on Stop or replay

#### 5.3 Rapid Styling
- Rapids animate at 5x speed relative to cuts
- Dashed arrow trajectory instead of solid line
- Lower opacity during simulation playback

---

### Phase 6: Command Palette + Polish

#### 6.1 Command Palette (Excalidraw-inspired)
- `Ctrl+K` opens overlay with fuzzy search input
- All registered keyboard shortcuts appear as searchable commands
- Categories: File, Edit, View, Tool, Simulation
- Last-used command pinned at top
- Arrow keys to navigate, Enter to execute, Escape to close
- SFX: commandOpen, commandNavigate, commandExecute

#### 6.2 Toast Notifications
- Slide up from bottom-right, auto-dismiss 2.5s
- Stack up to 3 toasts
- Show for: file saved, segments deleted, undo action, mode changes

#### 6.3 Help Overlay
- `?` shows modal with all keyboard shortcuts in a grid
- Grouped by category (File, Edit, View, Tool, Sim)
- First visit: brief tooltip "Press ? for shortcuts"

#### 6.4 Sample Programs
- Dropdown with 4-5 built-in programs:
  - Bracket (current default)
  - Circle
  - Pocket (rectangular pocket with passes)
  - Contour with ramps
  - Complex part (lots of arcs)

#### 6.5 localStorage Persistence
- Auto-save `S.doc.lines` + `S.view` to localStorage on every edit (debounced 500ms)
- Restore on page load
- "New" button clears and loads default sample

## Acceptance Criteria

- [x] Architecture: event bus + modules, 3-layer canvas, 3-tier state
- [x] Undo/Redo works for all edit ops (drag, add, delete, code edit)
- [x] File open (drag-drop + picker), save (.nc download), clipboard copy
- [x] All keyboard shortcuts work
- [x] Draw mode: click=G1, click+drag=arc, ghost preview
- [x] Delete segment with reconnection
- [x] Context menu on right-click
- [x] Feed rate heat map (viridis palette)
- [x] Tool diameter kerf band
- [x] Stats panel with machining time estimate
- [x] Direction arrows on segments
- [x] Origin marker (WCS crosshair)
- [x] Multi-select (shift+click, marquee, Ctrl+A)
- [x] Simulation play/pause/speed/scrub
- [x] Material removal bitmap preview
- [x] Command palette (Ctrl+K, fuzzy search)
- [x] Toast notifications
- [x] Help overlay (? key)
- [x] Sample programs dropdown
- [x] localStorage auto-save/restore
- [x] Pinch zoom on touch
- [x] 60fps with 500+ segments
- [x] Single file, no dependencies, <3000 lines (1760 lines)
- [x] All existing features preserved (drag, audio, code panel)
