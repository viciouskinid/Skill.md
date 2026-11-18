---
name: professional-desktop-application-ui
description: Design and build polished, professional desktop-style web applications with a left mode rail that is icon-only until hovered (then slides its labels out and pushes the side panel across), a contextual panel, a workspace and optional inspector. Use for new applications, UI redesigns, adding navigation to an existing app, dashboards, editors, chat or agent tools, management tools and data viewers, regardless of domain.
---

# Professional Desktop Application UI
Purpose
Create cohesive, high-quality application interfaces using a reusable
desktop-application architecture. The design reference is the overall
layout and interaction model of Hackodobe (an Adobe-style PDF editor),
but this skill is domain-neutral. Do not reproduce PDF-specific
features unless the user's application actually needs them.
The goal is to make applications feel like carefully designed
professional software: compact, efficient, contextual, visually
consistent, and focused on the user's primary work.

**Non-negotiable signature behaviour: the hover-expanding rail.** The
left rail is icon-only (about 62 px). Hovering it (or keyboard focus, or
pinning it) widens it to about 180 px, the labels slide/fade out of the
rail, and the contextual side panel is pushed to the right rather than
covered. Do not substitute always-visible labels, labels stacked under
the icons, a fixed-width sidebar, or an overlay that floats above the
panel. Read section 2B and use its reference CSS before building any
navigation. This applies to existing apps being redesigned as well as
new ones, and to chat, agent and admin tools as much as editors. Open
the reference page in a browser, or copy the CSS in 2B, instead of
working from a summary of this skill.
1. Start with the application's purpose
Before designing the interface:
Identify the application's primary user task and central working
object (if any).
Identify the major workflows or modes users need to switch between.
Identify global actions that should always be accessible.
Identify contextual tools that depend on the active mode.
Identify properties or metadata relevant to the current selection.
Identify useful status information, navigation controls, and
feedback.
Choose the simplest layout that supports these needs.
Do not force every application to use every panel. The architecture is a
flexible starting point, not a rigid template. For a focused utility,
omit unnecessary navigation or inspectors. For a complex application,
allow additional panels or secondary workspaces where justified.
2. Core application architecture
Use this as the default desktop layout:
``` text
┌──────────────────────────────────────────────────────────────────────┐
│ BRAND / APP NAME   Global menus and actions                 Account  │
├───────┬───────────────────┬──────────────────────────────┬───────────┤
│       │                   │                              │           │
│ MODE  │ CONTEXTUAL TOOLS  │       MAIN WORKSPACE         │ INSPECTOR │
│ RAIL  │ / TOOL PANEL      │                              │ / DETAILS │
│       │                   │                              │           │
│       │                   │                              │           │
│       │                   │                              │           │
├───────┴───────────────────┴──────────────────────────────┴───────────┤
│ STATUS / CURRENT MODE / SELECTION / CONTEXTUAL CONTROLS             │
└──────────────────────────────────────────────────────────────────────┘
```
A. Top toolbar
A compact horizontal bar at the top of the application.
Include as appropriate: - Application icon, brand mark, and application
name. - Global menus such as File, Edit, View, Tools, Window, and
Help. - Frequently used global actions. - Search, account, connection,
or workspace controls where relevant.
Guidelines: - Keep the toolbar compact and avoid filling it with every
available action. - Group related actions logically. - Use dropdown
menus for secondary or infrequent commands. - Show keyboard shortcuts in
menus when they exist. - Keep global actions distinct from mode-specific
tools. - Ensure menus have clear open, hover, focus, and selected
states.
B. Left navigation rail
A narrow vertical rail for switching between major application modes or
work areas.
Examples of generic modes: - Home or Overview - Create or Edit - Explore
or Browse - Organise or Manage - Analyse - Configure - History or
Activity
Reference behaviour (follow unless the user asks otherwise): - Collapsed
width about 62 px, icon-only. Hovering the rail, or keyboard focus
(`:focus-visible`), expands it to about 180 px and fades in the labels
and a small section label such as "Workspace". - Expansion pushes the layout, as in the reference: the rail's grid column widens (animate it) and the contextual panel and workspace move over; the rail does not float above them. - Provide a pin control that keeps the rail expanded
(persisted), so touch and keyboard users never depend on hover. -
Clicking the active mode a second time collapses the contextual panel;
clicking again reopens it. The collapsed panel's column must give up its
width (set the panel width token to 0), not just hide its contents. -
Navigation modes are workflows (for example Browse, Rooms, People,
Data); do not add a separate global "edit mode" toggle just to reveal
tools. - Global search belongs in the top bar only; do not duplicate it
in the rail or panels. - The right inspector is hidden until something
is selected.
Reference implementation (pure CSS, no JavaScript needed except the pin):

``` css
:root { --rail-collapsed: 62px; --rail-expanded: 180px; }
.rail {                      /* a flex/grid SIBLING of the panel, never position:absolute */
  width: var(--rail-collapsed); flex: none; overflow: hidden;
  transition: width .22s cubic-bezier(.2,.8,.3,1);
}
.rail:hover,
.rail:has(:focus-visible),    /* NOT :focus-within: a clicked button keeps focus and would hold the rail open */
.rail.pinned { width: var(--rail-expanded); }

.rail-btn { display: flex; align-items: center; gap: 10px; width: 100%;
  padding: 8px 0 8px 11px; white-space: nowrap; overflow: hidden; }
.rail-btn .icon  { flex: none; width: 24px; text-align: center; }  /* icon stays put while the rail grows */
.rail-btn .label { opacity: 0; max-width: 0; overflow: hidden;
  transition: opacity .15s ease, max-width .25s ease; }
.rail:hover .label, .rail:has(:focus-visible) .label, .rail.pinned .label
  { opacity: 1; max-width: 120px; }
```

Notes: - The label sits to the right of its icon in a row. - Badges
(unread counts, warnings) that must be seen while collapsed are
positioned on the icon corner; low-priority badges appear only when
expanded. - Pin: a button at the bottom of the rail toggles `.pinned`,
sets `aria-pressed`, and persists in `localStorage` inside try/catch. -
Phones: there is no hover, so the rail lives in a slide-in drawer and is
shown permanently expanded (`width: var(--rail-expanded); transition:
none`) with labels visible. Give the closed drawer `visibility:hidden`
and `overflow:hidden` so its contents cannot overlap or trap focus. -
Use `title` and `aria-label` on every rail button because labels are
hidden when collapsed.

Verify the behaviour with a browser test, not by eye: at rest the rail
is 62 px wide, the panel's left edge is 62 px and labels have zero
width; while hovering the rail is 180 px, the panel's left edge is 180
px (pushed) and labels are visible; after the pointer leaves it returns
to 62 px; with the pin on it stays 180 px after reload.

Guidelines: - Use recognisable, consistent icons. - Show a clear
active-mode indicator. - Support a compact icon-only state and an
expanded state with labels. - Expansion may occur on hover, pinning, or
an explicit toggle; do not rely on hover alone where accessibility or
touch use matters. - Use tooltips when labels are hidden. - Keep
navigation separate from the tools available within a mode.
C. Contextual tool panel
A panel immediately to the right of the navigation rail. Its contents
change when the user selects a major mode.
Typical structure: - Panel title and concise description. - Clearly
labelled sections. - Tool buttons, action lists, cards, filters, and
grouped controls. - Optional search or filtering for large tool
collections. - Scrollable content when needed.
For list-based apps (chat, agents, tickets, files) the contextual panel
is the list of items for the active mode, with a primary "New ..."
button above it. Selecting an item shows it in the workspace, with the
most useful content (for example the latest report or message) first.
Modes with no list (for example Usage or Settings) collapse the panel to
zero width.

Guidelines: - Show only tools relevant to the active mode. - Group tools
by task, not arbitrary implementation categories. - Use compact rows
with an icon, label, and optional short description. - Distinguish the
selected tool from hover and disabled states. - Use a wider panel only
when the task requires more information. - Avoid excessive nesting and
long, unstructured lists.
D. Main workspace
The central area is the primary destination for the user's work. It
should receive the largest share of available space.
Depending on the application, it may contain: - A canvas or editor. - A
map, diagram, timeline, chart, or visualisation. - A document or media
viewer. - A data table or records browser. - A project overview or
dashboard. - A terminal, code editor, or structured work surface.
Guidelines: - Make the main task visually dominant. - Avoid unnecessary
borders, cards, and decoration inside the workspace. - Provide useful
empty, loading, error, and populated states. - Support appropriate
zooming, panning, scrolling, selection, and navigation when relevant. -
Keep workspace-specific controls close to the workspace without
confusing them with global menus. - Use overlays or floating controls
sparingly. - Preserve the workspace when side panels are opened, closed,
or resized.
E. Right inspector / properties panel
A contextual panel for details about the current selection, object,
record, or workspace.
Potential contents: - Properties and editable fields. - Appearance or
formatting controls. - Metadata and technical details. - Configuration
and advanced options. - Validation results or diagnostics. - Contextual
actions.
Guidelines: - Update the panel when the selection or context changes. -
When nothing is selected, show useful general information, a helpful
prompt, or hide the panel. - Group properties into logical sections. -
Use tabs only when they make substantial groups easier to navigate. -
Use suitable controls: text fields, dropdowns, segmented controls,
checkboxes, switches, colour selectors, sliders, and numeric inputs. -
Make read-only values visually distinct from editable values. - Explain
non-obvious settings with concise supporting text. - Allow the panel to
be hidden, and resize it only where useful.
F. Bottom status bar
A narrow persistent bar for concise, contextual information.
Possible items: - Current mode or active tool. - Selection count or
selected object. - Operation or connection status. - Validation state. -
Progress or background activity. - Context-specific navigation, zoom,
coordinates, scale, page, or record position.
Guidelines: - Prioritise information that helps users understand the
current state. - Keep labels compact and avoid repeating information
already prominent elsewhere. - Use clear status semantics for success,
warning, error, and in-progress states. - Place secondary workspace
controls here only when they are genuinely useful and contextually
appropriate.
3. Visual design language
Use a restrained, professional visual system.
Default appearance
Unless the user requests another style: - Use a dark neutral application
shell. - Use layered charcoal, slate, or near-black surfaces. - Use
subtle borders to separate panels. - Use one primary accent colour for
selection, focus, and primary actions. - A secondary accent may be used
sparingly for emphasis. - Reserve red, amber, and green for meaningful
error, warning, and success states. - Use soft shadows for floating
menus and dialogs, not every panel. - Prefer solid or subtly varied
surfaces over strong gradients.
The reference design uses a dark shell with a near-black background,
progressively lighter panels, muted grey text, fine borders, and
blue/purple accents. Treat these as inspiration rather than mandatory
colour values. Choose a palette that suits the application's purpose and
accessibility needs.
Suggested design tokens
Define colours, spacing, typography, radii, and shadows centrally using
CSS custom properties or the equivalent design-token system.
Example starting palette (adapt as needed):
``` css
:root {
  --app-bg: #0f1115;
  --panel-1: #171a20;
  --panel-2: #1c2029;
  --panel-3: #252a34;
  --panel-4: #2e3440;

  --border: #2a2f3a;
  --border-strong: #343a47;

  --text: #e7eaf0;
  --text-secondary: #c3cad8;
  --text-muted: #8a93a6;

  --accent: #5b8cff;
  --accent-secondary: #8b5cf6;
  --success: #22c55e;
  --warning: #f59e0b;
  --danger: #ef4444;

  --radius-sm: 6px;
  --radius-md: 8px;
  --radius-lg: 10px;

  --rail-width: 62px;
  --rail-expanded-width: 180px;
  --topbar-height: 48px;
  --statusbar-height: 34px;
}
```
Do not use these exact values blindly. Maintain contrast and visual
consistency.
Typography
Use a clean system sans-serif font stack unless the application has
a strong reason for another typeface.
Use compact but readable typography.
Establish a clear hierarchy between application title, panel title,
section label, control label, and supporting text.
Use muted text for secondary information, not for essential
instructions.
Use tabular numerals for measurements, counters, IDs, timestamps,
and other values that benefit from alignment.
Avoid oversized headings that consume workspace.
Spacing and density
Use a consistent spacing scale.
Prefer compact controls suitable for professional desktop workflows.
Provide enough spacing to distinguish groups and avoid visual
clutter.
Keep panel padding and control heights consistent.
Avoid both excessive whitespace and cramped, hard-to-target
controls.
Borders, surfaces, and elevation
Use subtle one-pixel borders to distinguish adjacent surfaces.
Use progressively lighter surfaces to show hierarchy.
Apply shadows to menus, popovers, dialogs, and floating elements.
Avoid placing every control inside its own card.
Use stronger borders or an accent outline for selected items.
4. Interaction and component patterns
Build a consistent interaction language across the entire application.
Buttons and actions
Provide default, hover, pressed, focus, active, and disabled states
as appropriate.
Use a primary style for the main action in a context.
Use neutral secondary buttons for ordinary actions.
Use destructive styling for potentially damaging actions.
Use icon-only buttons only when the icon is recognisable; provide a
tooltip and accessible name.
Do not make controls appear interactive if they do nothing.
Selection
Make selected navigation modes, tools, tabs, rows, objects, and
cards visually obvious.
Do not rely on colour alone; combine colour with an outline, marker,
icon, or other cue.
Keep hover and selected states visually distinct.
Menus and popovers
Anchor menus to the control that opened them.
Close menus appropriately when an action is selected, the user
clicks elsewhere, or Escape is pressed.
Keep menu widths and grouping consistent.
Avoid menus that extend beyond the viewport.
Tabs and segmented controls
Use tabs for distinct content sections.
Use segmented controls for a small set of mutually exclusive
options.
Clearly show the active option.
Avoid tabs when a simple heading and section layout would be
clearer.
Forms and property controls
Align labels and inputs consistently.
Use suitable input types and sensible defaults.
Validate values and explain errors near the relevant control.
Preserve entered values when switching context where appropriate.
Distinguish editable, read-only, disabled, and calculated values.
Use units and formatting consistently.
Dialogs and modals
Use dialogs for focused tasks that require attention or a contained
workflow.
Include a title, optional description, body, and clearly ordered
actions.
Keep dialogs within the viewport and their contents independently
scrollable.
Provide an obvious close or cancel action.
Require confirmation for destructive or difficult-to-reverse
operations when appropriate.
Do not use a modal for information that belongs naturally in a side
panel.
Notifications and feedback
Use concise toast notifications for non-blocking outcomes.
Use inline feedback for validation and field-level errors.
Use persistent status indicators for ongoing conditions.
Show progress for long-running operations.
Never report success before the operation has actually succeeded.
Tooltips and help
Use tooltips for icon-only controls and potentially unfamiliar
actions.
Add concise explanatory text for complex or consequential settings.
Keep help close to the task it explains.
5. Responsive and adaptive layout
Although the reference is desktop-oriented, the design must adapt to the
target device.
For desktop: - Keep the full multi-panel layout where space permits. -
Allow side panels to collapse or resize. - Give the central workspace
priority.
For smaller windows: - Collapse the navigation rail. - Hide or convert
the least important inspector into a drawer or overlay. - Reduce
contextual panel width. - Move secondary controls into menus. - Preserve
access to essential functions.
For touch devices: - Do not rely on hover to reveal essential actions. -
Increase hit targets. - Use drawers or stacked navigation where
appropriate. - Avoid desktop-only interaction assumptions.
Use responsive breakpoints based on content needs, not arbitrary device
labels. Prevent horizontal overflow and ensure dialogs remain usable.
6. Functional and technical quality
A polished appearance is not sufficient. The application must work
coherently.
Every visible button, menu item, tab, switch, slider, and
interactive card must have a defined behaviour.
Maintain a clear source of truth for application state.
Keep active mode, active tool, selection, inspector contents, and
status information synchronised.
Ensure panel visibility and resizing do not break the workspace.
Support keyboard navigation and visible focus indicators.
Provide accessible names and semantic controls.
Use appropriate keyboard shortcuts for frequent actions and document
them in Help where relevant.
Handle empty, loading, success, warning, and error states.
Avoid unnecessary re-rendering or expensive operations in
interactive workspaces.
Persist preferences only when appropriate and explain important
persistence behaviour.
Protect user data and avoid destructive actions without suitable
safeguards.
Keep the implementation modular and maintainable.
If building a web application, prefer CSS Grid or Flexbox for the
application shell. Use `min-width: 0` and `min-height: 0` on nested
flex/grid children where necessary so scrollable panels behave
correctly. Give each independently scrollable panel its own overflow
handling. Keep fixed application chrome separate from the scrolling
workspace.
7. Recommended implementation sequence
Define the application modes and primary workflows.
Establish the design tokens and global typography.
Build the application shell: top toolbar, navigation rail,
contextual panel, workspace, inspector, and status bar.
Implement responsive panel behaviour.
Build the central workspace and its primary interactions.
Add contextual tools and inspector content.
Add menus, dialogs, notifications, and keyboard shortcuts.
Connect all controls to real application state and functionality.
Test selection changes, panel toggling, resizing, scrolling, and
empty/error states.
Refine spacing, contrast, alignment, density, and interaction
feedback.
Do not spend time polishing decorative details before the main workflow
and layout work correctly.
8. Design review checklist
Before considering the interface complete, verify:
[ ] The primary task is immediately apparent.
[ ] The workspace has sufficient space and visual priority.
[ ] The rail is icon-only at rest, widens on hover/focus/pin, slides its
labels out, and pushes the side panel (verified by a browser test).
[ ] The rail has a persisted pin, and on phones it is a drawer that is
shown expanded.
[ ] Navigation is distinct from contextual tools.
[ ] Contextual tools update when the active mode changes.
[ ] Inspector content reflects the current selection or context.
[ ] The status bar communicates useful current state.
[ ] Panel widths, spacing, typography, and borders are consistent.
[ ] Selected, hover, focus, disabled, success, warning, and error
states are distinguishable.
[ ] Menus, popovers, and dialogs open and close correctly.
[ ] All visible controls perform their stated actions.
[ ] Keyboard and accessibility basics are covered.
[ ] The layout adapts to smaller viewports without losing essential
functionality.
[ ] The design is appropriate to the application's domain rather
than an unmodified copy of the reference.
9. Adaptation rule: preserve the system, not the subject matter
When applying this skill to a new application:
Reuse the layout architecture, interaction patterns, visual
hierarchy, and design discipline.
Replace all domain-specific labels, tools, controls, and workspace
elements with those required by the new application.
Do not assume a document canvas, page thumbnails, zoom controls, PDF
annotations, or any other feature from the reference application.
Choose the central workspace and contextual panels according to the
actual task.
Omit components that add no value.
Add specialised components when the domain requires them, while
keeping the overall design coherent.
The result should feel like a member of the same family of professional
desktop applications, not like a reskinned copy of Hackodobe.
