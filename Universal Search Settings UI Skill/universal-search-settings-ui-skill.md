---
description: Design and implement polished, reusable search interfaces
  and comprehensive settings experiences for any application, including
  an extensive theme system with curated palettes, light/dark modes,
  custom themes, persistence, accessibility, and responsive behaviour.
name: universal-search-settings-ui
---

# Universal Search & Settings UI Skill

## Purpose

Use this skill when designing, implementing, reviewing, or improving the
**search interface** and **settings experience** of any application: web
apps, desktop applications, dashboards, admin tools, productivity
software, content libraries, developer tools, and mobile applications.

The skill is deliberately application-agnostic. Do not reproduce the
reference application's link tiles, folders, CSV editor, storage
inspector, image cache, or business logic. Reuse only the transferable
interaction and visual-design patterns.

The reference design uses: - A compact search trigger that opens a
floating, focused search surface. - A centred panel with a prominent
input, live result list, clear selection state, and concise keyboard
hints. - Search results with an icon, primary label, secondary
contextual text, and optional type/action tag. - Keyboard navigation
with Arrow Up/Down, Enter, Escape, and a global shortcut. - A settings
dialog with a persistent category navigation rail and a scrollable
content panel. - Clear section headings, explanatory helper text,
compact controls, preset colour swatches, a custom colour input, and
persistent preferences. - Responsive behaviour that changes the settings
navigation from a side rail to a horizontal category strip on narrow
screens.

Treat these as design principles, not rigid dimensions or implementation
constraints.

------------------------------------------------------------------------

# Part A --- Universal Search UI

## 1. Design intent

Search should feel immediate, discoverable, lightweight, and consistent
throughout the application. It should help users find the application's
meaningful objects and actions without forcing them to navigate through
menus.

Use a floating command/search panel when search is a global capability.
Use an inline search field when search is local to a specific list,
table, page, or collection. An application may use both, but their
scopes must be clearly distinguishable.

### Default visual structure

1.  A search trigger in the application's primary toolbar or navigation.
2.  A subtle dimmed backdrop when the global search panel is open.
3.  A centred or top-centred floating panel, sized to its content and
    viewport.
4.  A single prominent search input with search icon, placeholder, and
    clear/close control.
5.  A scrollable results region.
6.  Optional grouped results, recent searches, suggestions, empty
    states, and keyboard hints.

Suggested desktop dimensions: - Panel width: approximately 560--720 px,
constrained by viewport width. - Top offset: approximately 64--100 px,
or centred vertically if that suits the product. - Results max-height:
approximately 50--65 vh. - Input row height: approximately 48--56 px. -
Result row height: approximately 44--64 px. - Use a restrained radius
(10--16 px), subtle border, and layered shadow.

These are starting points, not mandatory values. Fit the interface to
the application.

## 2. Search trigger and opening behaviour

Provide at least one discoverable trigger: - Search icon/button in the
main toolbar. - A labelled search field in the relevant page. - A
command-palette shortcut such as Ctrl+K / Cmd+K. - Optionally `/` when
focus is not already in a text-entry control.

Recommended behaviour: - Opening global search focuses the input
automatically. - Preserve the user's current page and state when opening
and closing. - If already open, focus and select the existing query
rather than creating another panel. - Do not open global search over a
modal or another blocking interaction unless explicitly supported. -
Close on Escape, close button, or backdrop click. - Do not close when
the user clicks inside the panel. - Restore focus to the invoking
control after close. - On mobile, ensure the panel remains usable with
the virtual keyboard visible.

Avoid hijacking shortcuts while a user is typing in an input, textarea,
select, or contenteditable element. Respect browser and
assistive-technology conventions.

## 3. Search input

The search input should: - Be the visual focus of the panel. - Have a
clear placeholder describing the searchable content (for example,
"Search projects, people and files..."). - Support immediate typing and
editing. - Include a clear-query control when text is present. - Avoid a
visible border around the input itself if the containing input row
already provides sufficient structure. - Show loading, offline, or
search-error status where applicable. - Preserve a query only when doing
so is useful and expected; otherwise start with an empty query.

Use debouncing for expensive or remote search. Keep local search
responsive. Cancel or disregard stale asynchronous results so older
requests cannot overwrite newer ones.

## 4. Search result design

A result should communicate, at a glance: - **What it is:** a strong
primary title. - **Where it belongs:** a secondary path, category,
owner, location, or contextual description. - **What kind of result it
is:** optional type label or badge. - **How it can be acted on:**
opening, previewing, navigating to its parent, or invoking a relevant
action.

Recommended row anatomy:

`[icon/avatar] [primary title + secondary context] [optional type/status] [optional action affordance]`

Guidelines: - Align icons and text consistently. - Truncate long titles
and secondary descriptions without allowing layout overflow. - Highlight
the active/keyboard-selected row using the current theme's selection
colour. - Keep hover and keyboard-selection states visually
consistent. - Use badges sparingly; they should add meaning, not
decoration. - Use distinct icons for genuinely different object types. -
If a result has a parent/container, optionally provide a secondary "open
location" action. - Make the whole result row clickable when there is
one obvious primary action. - Keep secondary actions separately
targetable and labelled. - Do not make the user rely on colour alone to
understand selection or status.

### Grouped results

For applications with multiple object types, group results by useful
categories such as: - Pages - Files - Projects - People - Records -
Commands / Actions - Recent items

Show group headings only when they improve scanning. Avoid excessive
empty categories. Allow a sensible global ranking across groups or a
configurable grouping mode.

### Ranking and matching

Use a relevance model appropriate to the data. A useful baseline: 1.
Exact title match. 2. Title begins with query. 3. Title contains query.
4. Important metadata contains query. 5. Description or contextual path
contains query. 6. Fuzzy/approximate matches where useful.

Tokenise multi-word queries and avoid returning results that match only
an irrelevant fragment. Rank by relevance, then by a stable secondary
criterion. Do not silently change ranking between keystrokes without a
reason.

Where appropriate, search: - Display names and aliases. - Descriptions
and metadata. - Parent/container names. - Identifiers users are likely
to know. - Tags and categories. - Relevant command names and synonyms.

Do not search private or inaccessible records merely because they exist
in the underlying data.

## 5. Keyboard and pointer interaction

Implement a predictable keyboard model:

  -----------------------------------------------------------------------
  Key                                 Behaviour
  ----------------------------------- -----------------------------------
  Arrow Down                          Move selection to next result

  Arrow Up                            Move selection to previous result

  Enter                               Activate selected result; if none
                                      is selected, activate the top
                                      result if that is the product's
                                      chosen convention

  Escape                              Close search, or clear a transient
                                      sub-state before closing if
                                      appropriate

  Ctrl+K / Cmd+K                      Open global search/command palette

  `/`                                 Optional shortcut when focus is not
                                      in a text-entry control

  Tab / Shift+Tab                     Move through interactive controls
                                      in logical order
  -----------------------------------------------------------------------

Additional requirements: - Keep the selected result in view while
navigating. - Support mouse hover without causing erratic selection
changes. - Ensure pointer and keyboard interaction do not fight each
other. - Announce result counts and loading/no-result changes to
assistive technology where appropriate. - Use semantic dialog, listbox,
option, and combobox patterns correctly; implement the required ARIA
relationships and keyboard behaviour rather than adding roles
decoratively. - Maintain a visible focus indicator.

## 6. Search states

Design all of these states, not just the populated result list:

-   **Idle:** prompt the user to type, or show useful recent/suggested
    items.
-   **Typing:** query visible; results update.
-   **Loading:** unobtrusive progress indicator or status.
-   **Results:** ranked or grouped result rows.
-   **Keyboard selection:** one clearly selected row.
-   **No results:** explain that nothing matched and suggest changing
    the query.
-   **Error:** explain that search failed and provide retry where
    possible.
-   **Offline:** explain any reduced capability if relevant.
-   **Recent searches:** optional, user-controllable, and privacy-aware.
-   **Suggestions:** optional shortcuts or popular destinations.
-   **No permission:** explain that the user cannot access matching
    content, without exposing protected details.

Never show a blank panel without an explanation.

## 7. Search scope and filters

For complex applications, support scope filters such as "All", "Current
project", "Current workspace", or object-type filters. Keep them compact
and easy to understand.

Rules: - Make the current scope visible. - Provide a simple way to
return to "All". - Avoid presenting a large filter form in a small
command palette. - Preserve filters only when that matches user
expectations. - Use contextual scope by default only when it is obvious
and reversible. - Do not silently narrow global search to the current
page.

## 8. Responsive and mobile behaviour

Desktop: - Floating panel with a moderate width and restrained
backdrop. - Keyboard hints may be shown in the footer.

Tablet: - Reduce margins and panel width. - Keep the input and results
comfortably touchable.

Mobile: - Use a near-full-width sheet or full-screen search view. -
Respect safe areas and the virtual keyboard. - Use touch targets of
roughly 44 × 44 px or larger. - Hide desktop-only keyboard hints when
they are not useful. - Keep close and clear controls easy to reach. -
Ensure result rows do not depend on hover.

------------------------------------------------------------------------

# Part B --- Universal Settings UI

## 9. Design intent

Settings should be a coherent, predictable control centre rather than a
collection of unrelated popups.

Use a dialog or dedicated settings page depending on the application's
complexity: - A compact application may use a modal settings window. - A
larger product may use a full-page settings route. - A mobile
application may use a full-screen settings view.

The reference pattern is a two-region layout: - **Navigation rail:**
fixed-width category list. - **Content panel:** independently scrollable
setting sections.

The header should identify the settings area and provide a close/back
action. The active category should be obvious. Keep the navigation
visible on desktop while the content panel scrolls.

## 10. Settings information architecture

Choose categories based on the application. Do not blindly include every
category below. Include only categories that have real settings.

A generic category library:

### General

-   Application behaviour
-   Startup and default views
-   Language and locale
-   Date, time, number, and measurement formats
-   Default actions
-   Confirmation behaviour

### Appearance

-   Themes
-   Light/dark/system mode
-   Accent colour
-   Density and spacing
-   Typography and font size
-   Motion and animation
-   Icon and visual preferences

### Search

-   Default search scope
-   Searchable content types
-   Result density
-   Recent search history
-   Keyboard shortcuts
-   Suggestions and ranking preferences

### Notifications

-   In-app notifications
-   Email/push preferences where applicable
-   Quiet hours
-   Sound and desktop alerts
-   Per-category notification preferences

### Privacy and Security

-   Session and sign-in preferences
-   Privacy controls
-   Data visibility
-   Permission management
-   Security options relevant to the product

### Data and Storage

-   Storage usage
-   Cache management
-   Import/export
-   Backup and restore
-   Retention controls
-   Local/offline data settings

### Accessibility

-   Reduced motion
-   High contrast
-   Larger text
-   Keyboard navigation
-   Screen-reader preferences
-   Focus visibility

### Integrations

-   Connected services
-   API or integration preferences
-   Sync behaviour
-   Connection status

### Advanced

-   Expert options
-   Diagnostics
-   Developer options
-   Experimental features
-   Reset controls

### About

-   Application version
-   Build information
-   Licences and acknowledgements
-   Help, support, and documentation

For a small application, combine categories. For a large application,
split them into clear subcategories. Avoid empty sections and overly
long navigation lists.

## 11. Settings layout and controls

Each category panel should use a consistent structure: - Section
title. - Short explanatory sentence describing the effect of the
settings. - Related controls grouped together. - Optional divider
between distinct groups. - Inline validation or status feedback. -
Save/apply behaviour that is consistent throughout the application.

Use appropriate controls: - Toggle: a binary preference that takes
effect immediately. - Checkbox: an independently selectable option. -
Radio group: one choice from a small set. - Select/dropdown: one choice
from a longer set. - Slider: a continuous or stepped value with visible
current value. - Text/number input: explicit values. - Colour picker:
custom colour. - Button: action such as export, clear, test, reset, or
reconnect.

Guidelines: - Put labels before or beside controls, not hidden in
placeholder text. - Explain non-obvious consequences in helper text. -
Show units for numeric values. - Validate input and explain errors next
to the affected control. - Distinguish destructive actions from ordinary
actions. - Ask for confirmation before irreversible or consequential
actions. - Avoid a Save button for every setting if settings apply
immediately; equally, do not imply immediate saving when changes are
only staged. - If changes are staged, show unsaved status and provide
Apply/Save and Cancel/Discard. - Provide a clear reset-to-default option
where useful. - Persist settings reliably and handle storage failures
gracefully.

## 12. Navigation and responsive settings

Desktop: - Use a left navigation rail around 180--240 px wide. - Keep
category labels readable and optionally pair them with icons. -
Highlight the active category with a soft theme-derived background and
stronger text. - Allow the content panel to scroll independently. - Keep
the settings title and close action stable.

Mobile: - Convert the left rail into a horizontally scrollable category
strip, compact list, or nested navigation page. - Do not shrink the
desktop rail until labels become unreadable. - Ensure the content area
uses the full available width. - Keep controls touch-friendly. - Avoid
nested scrolling where possible.

Navigation behaviour: - Selecting a category updates the visible panel
without losing unrelated edits. - Preserve the active category when
reopening settings if useful. - Use a clear active state and accessible
current-page/current-tab semantics. - Support direct links to a settings
category when the application has routes. - If a category is unavailable
due to permissions or platform limitations, explain why rather than
presenting a broken control.

------------------------------------------------------------------------

# Part C --- Extensive Theme System

## 13. Theme system requirements

The theme selector is a first-class part of Appearance settings. It must
provide complete themes, not merely a list of accent colours.

A theme should define a coherent set of semantic design tokens,
including: - Application/page background. - Main surface and elevated
surface. - Primary and secondary text. - Muted text. - Borders and
dividers. - Primary/accent colour. - Text colour on accent surfaces. -
Hover, selected, focus, and pressed states. - Success, warning, danger,
and information colours. - Input, card, menu, dialog, tooltip, and
navigation surfaces. - Optional shadow, radius, and density tokens.

Use semantic tokens rather than hard-coding colours throughout
components. Example token names:

``` css
:root {
  --color-bg: #f6f7fb;
  --color-surface: #ffffff;
  --color-surface-elevated: #ffffff;
  --color-text: #202432;
  --color-text-muted: #697386;
  --color-border: #dfe3eb;
  --color-accent: #536dfe;
  --color-on-accent: #ffffff;
  --color-accent-soft: #edf0ff;
  --color-hover: #f1f3f8;
  --color-selected: #e7ebff;
  --color-focus: #536dfe;
  --color-success: #21865a;
  --color-warning: #a96b08;
  --color-danger: #c13d4b;
  --color-info: #2879b9;
}
```

The exact tokens can vary, but components must consume semantic tokens
consistently.

## 14. Theme selector experience

Build a dedicated theme selection area inside Appearance settings.

Recommended layout: 1. Appearance heading and short explanation. 2. Mode
selector: Light / Dark / System / optionally Auto. 3. Theme search
field. 4. Category chips or filters. 5. Theme gallery using preview
cards. 6. Selected-theme indicator. 7. Customisation controls. 8. Custom
theme save/create controls. 9. Reset-to-default option.

Each theme card should include: - Theme name. - Small visual preview of
background, surface, text, and accent. - Optional light/dark label. -
Optional short descriptive tag. - A clear selected indicator. - A
preview action or immediate apply behaviour.

The preview should resemble a miniature application interface, not just
four coloured circles. Include sample cards, text, buttons, and
selection states.

Do not require a reload to preview or apply a theme. Theme changes
should update the application immediately and persist across sessions.

## 15. Theme modes

Support these modes: - **Light:** use the theme's light palette. -
**Dark:** use its dark palette. - **System:** follow the operating
system preference and update when that preference changes. - **Auto
(optional):** follow a user-defined schedule or context if the
application genuinely supports it.

Changing the mode must not discard the selected theme. Where a theme has
no separately designed dark palette, generate a carefully adjusted dark
variant and allow the user to refine it. Do not simply invert all
colours.

## 16. Large curated theme library

Provide a substantial theme catalogue. Aim for at least 60 named
options, preferably 80 or more, with light and dark variants wherever
practical.

Use the following theme families as a starting catalogue. Names are
inspiration and should be implemented as coherent palettes rather than
literal brand copies.

### Neutral, professional, and productivity

1.  Classic Light
2.  Classic Dark
3.  Soft Grey
4.  Graphite
5.  Slate
6.  Charcoal
7.  Midnight
8.  Monochrome
9.  Paper
10. Warm Paper
11. Office Blue
12. Corporate Navy
13. Steel
14. Platinum
15. Minimal

### Blue and cyan

16. Ocean Blue
17. Deep Ocean
18. Arctic
19. Ice
20. Sky
21. Azure
22. Cobalt
23. Sapphire
24. Denim
25. Pacific
26. Lagoon
27. Glacier
28. Blue Steel
29. Electric Blue
30. Midnight Blue

### Purple, violet, and pink

31. Lavender
32. Soft Lilac
33. Violet
34. Royal Purple
35. Amethyst
36. Orchid
37. Plum
38. Grape
39. Magenta
40. Rose
41. Blush
42. Dusty Pink
43. Raspberry
44. Neon Violet
45. Cosmic

### Green and nature

46. Forest
47. Deep Forest
48. Sage
49. Mint
50. Seafoam
51. Emerald
52. Jade
53. Moss
54. Olive
55. Fern
56. Pine
57. Aurora Green
58. Tropical
59. Eucalyptus
60. Celadon

### Warm, earth, and sunset

61. Sand
62. Desert
63. Terracotta
64. Clay
65. Copper
66. Bronze
67. Amber
68. Honey
69. Sunset
70. Sunrise
71. Coral
72. Peach
73. Apricot
74. Autumn
75. Coffee
76. Mocha
77. Cocoa
78. Burgundy
79. Wine
80. Crimson

### High-contrast, vivid, and experimental

81. High Contrast Light
82. High Contrast Dark
83. Neon
84. Cyberpunk
85. Synthwave
86. Matrix
87. Terminal
88. Retro Green
89. Retro Amber
90. Arcade
91. Solarized Light
92. Solarized Dark
93. Nord
94. Dracula-inspired
95. Gruvbox-inspired
96. Tokyo Night-inspired
97. Monokai-inspired
98. Pastel
99. Candy
100. Rainbow

These are a minimum catalogue target, not a limit. Add more themes where
they are genuinely distinct and visually coherent. Avoid near-duplicate
palettes that differ only by a few RGB values.

For named themes inspired by established third-party themes, use an
original compatible palette and avoid implying official affiliation or
exact reproduction.

## 17. Theme categories and discovery

Make a large theme library easy to browse: - All themes - Light - Dark -
Neutral - Professional - Blue - Purple and Pink - Green - Warm - High
Contrast - Retro and Terminal - Colourful - Custom

Include: - Search by theme name. - Filter by family and mode. - Sort by
name or curated order. - Optional favourites. - Recently used themes. -
Clear selected-theme state. - A compact theme count.

Do not display all 100 themes as an unstructured wall of swatches. Use a
responsive grid, category filters, and search. Keep the selected theme
easy to identify.

## 18. Custom theme builder

Allow users to create and save their own themes.

At minimum, expose: - Theme name. - Base mode (light/dark). - Background
colour. - Surface colour. - Primary text colour. - Secondary text
colour. - Border colour. - Accent colour. - Optional success, warning,
danger, and information colours.

Useful additional options: - Preset accent palettes. - Custom accent
picker. - Surface tint strength. - Corner radius. - Interface density. -
Font scale. - Shadow intensity.

Provide: - Live preview. - Save as new theme. - Duplicate theme. -
Rename custom theme. - Delete custom theme with confirmation. - Reset
custom theme to its original saved state. - Import/export theme
definitions as JSON where appropriate.

Never allow a custom theme to make essential text or controls unreadable
without warning. Check contrast and recommend accessible alternatives.

## 19. Accent colour and colour presets

In addition to full themes, allow the user to adjust the accent colour
independently.

Provide a large palette of preset accent colours, for example: - Blue,
Indigo, Violet, Purple, Magenta, Pink, Rose, Red, Coral, Orange, Amber,
Gold, Yellow, Lime, Green, Emerald, Teal, Cyan, Sky, Navy, Slate,
Graphite, Brown, Copper.

Also provide a custom colour picker and reset button.

Accent colour should update: - Primary buttons. - Links. - Focus
indicators. - Selected navigation items. - Active tabs. - Selection
highlights. - Relevant icons and borders. - Progress indicators where
appropriate.

Generate appropriate hover, soft-background, border, and contrast
variants from the accent colour. Do not use the accent colour as the
background of every component.

A custom accent should override the theme accent without destroying the
underlying theme. Provide a clear action to restore the theme's original
accent.

## 20. Additional appearance controls

Where suitable, include: - Interface density: Compact / Comfortable /
Spacious. - Corner style: Sharp / Subtle / Rounded / Pill. - Typography:
System / Sans / Serif / Monospace, where supported. - Text size: Small /
Default / Large / Extra large. - Motion: System / Reduced / Off. -
Shadows: None / Subtle / Standard / Strong. - Navigation style: Standard
/ Compact / Expanded. - Icon style: Outline / Filled / Minimal, if the
app has multiple icon sets.

These controls should change real interface tokens and components, not
just their preview.

## 21. Theme persistence and architecture

Use a single central theme definition and application mechanism.

Recommended architecture: - A theme registry containing built-in themes
and user-created themes. - A selected theme ID. - A selected mode
(light, dark, system). - Optional accent override. - Optional appearance
overrides such as density and radius. - A single `applyTheme()` or
equivalent function that maps the chosen configuration to semantic
tokens. - A persistence layer appropriate to the platform. - A migration
strategy when theme schema changes.

For a web application: - CSS custom properties are a practical token
delivery mechanism. - Store preferences in localStorage or the
application's settings backend as appropriate. - Apply saved preferences
before first paint to avoid a flash of the default theme. - Listen to
`prefers-color-scheme` when System mode is active. - Keep user data
scoped to the application and handle storage exceptions. - Validate
imported theme JSON before applying it.

Example configuration shape:

``` json
{
  "themeId": "ocean-blue",
  "mode": "system",
  "accentOverride": null,
  "density": "comfortable",
  "radius": "rounded",
  "fontScale": "default",
  "reducedMotion": "system"
}
```

Do not hard-code the selected palette independently into individual
components. Do not let theme changes overwrite application data or
unrelated preferences.

## 22. Accessibility and quality requirements

-   Meet WCAG AA contrast for normal text wherever possible.
-   Provide stronger contrast options for users who need them.
-   Do not communicate selected, error, success, or warning states by
    colour alone.
-   Maintain visible focus rings in every theme.
-   Respect reduced-motion preferences.
-   Ensure dialogs have correct focus management and Escape behaviour.
-   Use semantic labels and descriptions for every control.
-   Support keyboard-only operation.
-   Avoid tiny swatches or touch targets.
-   Check disabled controls for sufficient legibility.
-   Test overlays, tooltips, dropdowns, menus, and charts in both light
    and dark modes.
-   Check status colours against every theme; semantic colours should
    remain distinguishable.

------------------------------------------------------------------------

# Part D --- Visual Design Language

## 23. Shared visual principles

The search and settings interfaces should look like parts of the same
application.

Use: - Consistent spacing scale. - Consistent border radius. -
Consistent typography hierarchy. - Consistent icon size and stroke
weight. - Theme-derived borders, surfaces, shadows, and focus colours. -
Restrained use of accent colour. - Clear separation between primary and
secondary information. - Subtle hover and selected states. - Consistent
button hierarchy.

Suggested spacing scale: 4, 8, 12, 16, 20, 24, 32 px.

Suggested radius scale: - Small controls: 6--8 px. - Cards and panels:
10--14 px. - Large sheets/dialogs: 12--18 px. - Pills: fully rounded
only for chips and intentionally pill-shaped controls.

Do not impose these values if the host application already has a design
system. Reuse its tokens and component conventions.

## 24. Motion and transitions

Use short, purposeful transitions: - Search panel open/close:
approximately 120--200 ms. - Hover and selection: approximately 100--160
ms. - Theme change: approximately 120--220 ms where it does not cause
distracting flashes.

Avoid excessive movement, bounce, or large scaling effects. Respect
reduced-motion settings and provide a no-motion path.

## 25. Layering and overlays

Define a deliberate z-index hierarchy for: - Main content. - Sticky
navigation and toolbars. - Dropdown menus. - Search panel/backdrop. -
Settings dialog. - Tooltips and transient notifications.

Prevent overlays from appearing behind application headers or other
dialogs. Close transient menus on outside click, Escape, navigation, or
viewport changes where appropriate. Do not close a dialog unexpectedly
when the user is interacting with its content.

------------------------------------------------------------------------

# Part E --- Implementation Workflow

## 26. Before implementation

First understand the host application: 1. Identify its platform and
framework. 2. Inspect the existing design system, navigation,
typography, spacing, and colour tokens. 3. Identify the objects and
actions that should be searchable. 4. Identify which settings are
genuinely configurable. 5. Determine whether settings apply immediately
or require explicit saving. 6. Determine persistence and account/device
scope. 7. Identify responsive breakpoints and accessibility
requirements. 8. Check whether a theme system already exists and whether
it can be extended.

Do not rewrite unrelated application functionality. Keep the
implementation modular and minimise regressions.

## 27. Suggested component structure

Adapt naming to the host framework:

``` text
Application
├── AppToolbar
│   ├── SearchTrigger
│   └── SettingsTrigger
├── GlobalSearch
│   ├── SearchInput
│   ├── SearchFilters
│   ├── SearchSuggestions
│   ├── SearchResultGroups
│   ├── SearchResultRow
│   ├── SearchEmptyState
│   └── SearchFooter
└── Settings
    ├── SettingsHeader
    ├── SettingsNavigation
    └── SettingsContent
        ├── GeneralSettings
        ├── AppearanceSettings
        │   ├── ThemeModeSelector
        │   ├── ThemeSearch
        │   ├── ThemeFilters
        │   ├── ThemeGallery
        │   ├── AccentColourPicker
        │   └── CustomThemeEditor
        ├── SearchSettings
        ├── NotificationSettings
        ├── PrivacySettings
        ├── DataSettings
        ├── AccessibilitySettings
        ├── IntegrationSettings
        └── AboutSettings
```

Do not create empty components merely to match this diagram. Implement
only the categories and controls relevant to the application.

## 28. Implementation rules

-   Separate search logic from search presentation.
-   Separate settings state from settings presentation.
-   Keep the theme registry separate from the theme gallery.
-   Use reusable controls and consistent component APIs.
-   Avoid duplicated theme definitions.
-   Avoid inline hard-coded colours in components.
-   Persist preferences through one well-defined settings service.
-   Debounce remote search and prevent stale responses.
-   Ensure keyboard and pointer interactions work together.
-   Ensure search result actions perform the correct
    application-specific action.
-   Avoid exposing settings the current user cannot change.
-   Use safe defaults and validate imported data.
-   Preserve existing application functionality and data.
-   Do not add unnecessary dependencies for simple interactions.

## 29. Testing checklist

### Search

-   [ ] Search trigger is discoverable.
-   [ ] Global shortcut works and does not hijack typing.
-   [ ] Input receives focus on open.
-   [ ] Results update correctly.
-   [ ] Relevance ranking is sensible.
-   [ ] Empty, loading, error, and offline states are handled.
-   [ ] Arrow Up/Down selection works.
-   [ ] Enter activates the correct result.
-   [ ] Escape closes the panel.
-   [ ] Backdrop click closes the panel.
-   [ ] Clicking inside does not close it.
-   [ ] Secondary actions work independently.
-   [ ] Long titles and descriptions do not break layout.
-   [ ] Mobile touch interaction works.
-   [ ] Screen-reader semantics and announcements are correct.

### Settings

-   [ ] Settings can be opened and closed reliably.
-   [ ] Category navigation works.
-   [ ] Active category is clearly indicated.
-   [ ] Content scrolls independently where appropriate.
-   [ ] Settings persist at the intended scope.
-   [ ] Unsaved changes are handled correctly.
-   [ ] Reset and destructive actions are confirmed.
-   [ ] Mobile navigation is usable.
-   [ ] All controls have labels and helper text where needed.

### Themes

-   [ ] At least 60--100 distinct curated themes are available.
-   [ ] Themes have coherent semantic token sets.
-   [ ] Light, dark, and system modes work.
-   [ ] Theme preview is accurate.
-   [ ] Theme applies without reload.
-   [ ] Selection persists across reloads/sessions.
-   [ ] Custom accent colour works.
-   [ ] Custom themes can be created, saved, duplicated, renamed, and
    deleted.
-   [ ] Theme import/export is validated if supported.
-   [ ] Focus, hover, selected, disabled, and status states remain
    legible.
-   [ ] Contrast is checked in every theme.
-   [ ] Reduced motion is respected.
-   [ ] Existing application data is unaffected.

------------------------------------------------------------------------

# Part F --- Expected Output When Using This Skill

When asked to create or improve an application using this skill:

1.  Inspect the existing application and identify its design
    conventions.
2.  Propose or implement a search experience suited to its data and
    workflows.
3.  Propose or implement a settings experience suited to its actual
    configuration needs.
4.  Include a substantial searchable/filterable theme gallery with at
    least 60 distinct theme options, preferably 100.
5.  Include light, dark, and system modes.
6.  Include a custom accent picker and a functional custom theme
    builder.
7.  Ensure the theme system uses semantic tokens throughout the
    application.
8.  Make the UI responsive, accessible, keyboard-friendly, and
    persistent.
9.  Preserve the application's existing functionality and data.
10. Report what was implemented, any assumptions, and any remaining
    limitations.

**Core principle:** Reuse the reference application's clarity, compact
floating search panel, keyboard-first result navigation, category-based
settings layout, and simple colour customisation. Generalise these
patterns into a reusable design system that can be applied to any
application, regardless of its purpose, data model, or framework.
