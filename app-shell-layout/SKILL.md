---
name: app-shell-layout
description: Use when designing or changing the overall layout of a desktop-style web app with a map or canvas: header, icon rail, resizable side panel, docked properties panel, bottom panel, status bar, toasts and dialogs. Says what content belongs in each region, how each region resizes and collapses, and how the layout adapts to tablets and phones.
---

# App shell layout

A tool-style app (map, canvas, editor) uses a fixed shell. Every feature has one correct home. Do not invent new regions; put features where this document says.

## 1. Regions

```
┌─────────────────────────────── Header (56 px) ───────────────────────────────┐
│ Brand            ·················  search · commands · inspector · locate │ undo redo ▾ │
├────┬──────────────────┬──────────────────────────────────────┬──────────────┤
│Rail│ Side panel       │ Main area (map / canvas)             │ Properties   │
│58px│ header           │  tool column  (top-left)             │ panel        │
│    │ tabs / breadcrumb│  draw banner  (top-centre)           │ (docked,     │
│    │ body (scrolls)   │  minimap      (bottom-left)          │  right)      │
│    │ footer (optional)│  zoom buttons (bottom-right)         │              │
│    │                  │  bottom panel (charts, resizable)    │              │
├────┴──────────────────┴──────────────────────────────────────┴──────────────┤
│ Status bar (28 px): coords · elevation · zoom/scale ··· progress · chips · build │
└──────────────────────────────────────────────────────────────────────────────┘
```

Flex layout: body is a column (header, workspace, status bar); the workspace is a row (rail, side panel, main area). Only the side panel body, the properties panel body and dialogs scroll; the page itself never scrolls (`overflow: hidden` on html and body).

## 2. What goes where

| Region | Contains | Never contains |
|---|---|---|
| **Header** | Brand, global search (icon that expands), command palette button, properties-panel toggle, locate-me, undo / redo / history dropdown | Tools that act on the canvas, section navigation |
| **Rail** | One icon per section (Places/Items, Draw, Links/Analysis, Terrain/Data, Layers, Settings last, pinned bottom). Tooltips and `aria-label`. Active marker. Clicking the active icon collapses the side panel. Hide icons for sections that have nothing to show | Text labels, actions |
| **Side panel** | The browsable content of the active section: lists, trees, archive, search results, layer pickers, settings | Properties of the selected item |
| **Properties panel (docked right)** | Edit form for the selected item: tabs Properties / Style / Geometry, live-saving fields, a "Saved" pill, destructive actions in its footer | Navigation |
| **Main area** | The map or canvas, plus its overlay controls | Lists |
| **Tool column** | Icon-only drawing and measuring tools, grouped with thin separators (create / measure / analyse / import), tooltips to the right | Settings |
| **Bottom panel** | Charts and profiles tied to the selection | General lists |
| **Status bar** | Passive readouts, progress, status chips, build stamp (see the `build-timestamp` skill), toggles for optional widgets | Primary actions |
| **Toasts** | Short confirmations, with an optional action such as Undo. Sit above the status bar, centred, 2.4 s (6.5 s with an action) | Errors needing a decision |
| **Dialogs** | Creating an item, import, export, confirmation of destructive actions, error details | Anything that should stay open while working |

Other surfaces: context menu (right-click and the `⋯` button, grouped: open, tools, organise, clipboard, export, destructive last), command palette (`Ctrl/Cmd+K`), drop overlay ("Drop to import"), full-screen import dialog, busy overlay with spinner and label for long jobs, plain loading screen until stored data is restored.

### Side panel anatomy (top to bottom)

1. Header row: section title, count badge (tooltip with breakdown), icon buttons for search, sort, archive, view options. Section-specific actions (for example Upload, Download in a data section) appear only for that section.
2. Search row (appears on demand).
3. Banner row: replaces the tab bar. Example: "← Back to places · 12 archived".
4. Breadcrumb: back button, path, `⋯` folder menu. Collapses long paths to `… / current` with a "show full path" toggle. May render vertically along the panel's left edge as an option.
5. Body: scrollable list. Rows are 32 px (44 px on touch), with icon, name, optional chip, count and a visibility toggle. Keyboard: arrows, Home/End, Enter, F2 rename, Alt+arrows reorder, `M` move to, `V` toggle visibility, context-menu key.
6. Footer: optional, hidden when empty.

## 3. Resizing and collapsing

Use CSS variables for sizes (`--side-w`, `--inspector-w`, `--bottom-h`). Remember each in localStorage (`prefSet`). Clamp to the viewport on window resize. Use Pointer Events (not mouse events) so touch works, with `setPointerCapture`. Disable width/height transitions while dragging; keep a 0.28 s transition for open and close.

| Panel | Default | Min | Max | Handle |
|---|---|---|---|---|
| Side panel | 320 px | 220 | 600 (or 50% of viewport) | 8 px wide strip on its right edge, `ew-resize`, highlighted on hover/drag |
| Properties panel | 380 px | 320 | 560 | same, on its left edge |
| Bottom panel | 320 px | 150 | map height | 12 px strip on its top edge, `ns-resize`, with a small grip bar |

Handle requirements for all three:

- `role="separator"`, `aria-orientation`, `aria-label`, `tabindex="0"`, `aria-valuenow/min/max`.
- Keyboard: arrows move 16 px, Shift+arrows 48 px, Home = min, End = max (bottom panel: Enter/End toggles full height).
- Double-click resets to default (side panels) or toggles full height (bottom panel).
- Bottom panel: dragging the top edge within about 28 px of the top snaps to full height (show an accent outline while snapping); the previous height is remembered and restored when un-expanding.
- Docked panels shrink the main area instead of covering it: add a class to the main area that sets its `right` offset to the panel width, and shift right-aligned overlay controls (zoom buttons) by the same amount. The bottom panel narrows the same way when the properties panel is open.
- After any resize, the map or canvas must re-measure itself (`ResizeObserver` on the main element), and any chart must redraw.
- Collapsing the side panel (click the active rail icon) sets its width to 0 with `overflow: hidden`; the main area expands.

## 4. Responsive behaviour

| Width | Behaviour |
|---|---|
| ≤ 900 px | Properties panel becomes a full-width overlay with a close button and closes when the user taps outside. Minimap hidden. Side panel keeps its width. |
| ≤ 700 px | Rail becomes a bottom tab bar (56 px). Side panel becomes a bottom sheet (46 vh, full width, no resize handle) above the tab bar. Header search expands to a full-width overlay. Tool tooltips off. Status bar keeps only the coordinates and progress. Tool column wraps. |
| `(pointer: coarse)` | Every interactive control at least 44 x 44 px. |

## 5. Theming and accessibility

- All colours are CSS variables on `:root` (surface, surface-soft, text levels, accent, border, danger, focus ring). Provide light, dark and system modes; set `data-theme` from a tiny inline script in `<head>` before first paint; add `<meta name="color-scheme">` and update `theme-color`.
- Never rely on colour alone: status chips carry an icon and text (for example ✓ Clear, ✕ Blocked).
- Visible `:focus-visible` outlines (2 px accent) on everything interactive; menus are `role="menu"` with arrow-key navigation, type-ahead and focus return on close.
- Respect `prefers-reduced-motion` (collapse transitions to near zero).
- Minimum text size about 10.5 px; use tabular numerals for readouts.
- Dialogs trap focus, close on Escape, and return focus.

## 6. First-run and empty states

When there is no data, the side panel shows a short welcome with a three-step checklist (for example load data, add an item, run an analysis), each with a button and a done state, plus a tip about drag-and-drop and the command palette. Show a plain loading screen until stored data has been restored so this welcome never flashes for returning users.

## 7. Acceptance checklist

- [ ] Each feature sits in the region named in the table in section 2.
- [ ] All three resize handles work with mouse, touch and keyboard, persist, clamp, and re-measure the map.
- [ ] Opening the properties panel never covers the map controls.
- [ ] At 900 px and 700 px the layout adapts as described.
- [ ] Everything is operable by keyboard; focus is always visible.
- [ ] Light, dark and system themes work with no flash on load.
- [ ] The page never scrolls; only the panel bodies and dialogs do.
