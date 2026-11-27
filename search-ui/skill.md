---
name: universal-search-ui
description: Design, implement or review a global search or command palette for any application, with ranked results, keyboard navigation, search states, filters, mobile layout and accessibility.
---

# Universal Search UI Skill

## Purpose

Use this skill when designing, implementing, reviewing, or improving the **search interface** of any application: web apps, desktop applications, dashboards, admin tools, productivity software, content libraries, developer tools, and mobile applications.

The skill is application-agnostic. Do not reproduce the reference application's link tiles, folders, CSV editor, storage inspector, image cache, or business logic. Reuse only the transferable interaction and visual-design patterns.

The reference design uses:

- A compact search trigger that opens a floating, focused search surface.
- A centred panel with a prominent input, live result list, clear selection state, and concise keyboard hints.
- Results with an icon, primary label, secondary context, and optional type/action tag.
- Keyboard navigation with Arrow Up/Down, Enter, Escape, and a global shortcut.

Treat these as design principles, not rigid dimensions or implementation constraints.

Searching *within a settings window* is a separate, local search. It belongs to the settings UI (see the `settings-ui` skill), not to this panel.

## 1. Design intent

Search should feel immediate, discoverable, lightweight, and consistent throughout the application. It should help users find meaningful objects and actions without navigating through menus.

Use a floating command/search panel when search is a global capability. Use an inline search field when search is local to a list, table, page, or collection. An application may use both, but their scopes must be clearly distinguishable.

### Default visual structure

1. A search trigger in the primary toolbar or navigation.
2. A subtle dimmed backdrop when the panel is open.
3. A centred or top-centred floating panel, sized to its content and viewport.
4. A single prominent input with search icon, placeholder, and clear/close control.
5. A scrollable results region.
6. Optional grouped results, recent searches, suggestions, empty states, and keyboard hints.

Suggested desktop dimensions (starting points, not mandatory):

- Panel width: about 560-720 px, constrained by viewport width.
- Top offset: about 64-100 px, or vertically centred.
- Results max-height: about 50-65 vh.
- Input row height: about 48-56 px.
- Result row height: about 44-64 px.
- Restrained radius (10-16 px), subtle border, layered shadow.

## 2. Trigger and opening behaviour

Provide at least one discoverable trigger:

- Search icon/button in the main toolbar.
- A labelled search field on the relevant page.
- A command-palette shortcut such as Ctrl+K / Cmd+K.
- Optionally `/` when focus is not in a text-entry control.

Recommended behaviour:

- Opening global search focuses the input.
- Preserve the user's current page and state when opening and closing.
- If already open, focus and select the existing query instead of creating another panel.
- Do not open over a modal or other blocking interaction unless explicitly supported.
- Close on Escape, close button, or backdrop click. Do not close on clicks inside the panel.
- Restore focus to the invoking control after close.
- On mobile, keep the panel usable with the virtual keyboard visible.

Avoid hijacking shortcuts while the user is typing in an input, textarea, select, or contenteditable element. Respect browser and assistive-technology conventions.

## 3. Search input

- Make it the visual focus of the panel.
- Use a placeholder that describes searchable content ("Search projects, people and files...").
- Support immediate typing and editing.
- Include a clear-query control when text is present.
- Avoid a border around the input if its row already provides structure, but keep focus visible.
- Show loading, offline, or error status where applicable.
- Preserve a query only when useful and expected; otherwise start empty.

Debounce expensive or remote search. Keep local search responsive. Cancel or ignore stale asynchronous results so older requests cannot overwrite newer ones.

## 4. Result design

A result should communicate at a glance:

- **What it is:** a strong primary title.
- **Where it belongs:** a path, category, owner, location, or description.
- **What kind of result it is:** optional type label or badge.
- **How it can be acted on:** open, preview, go to parent, or invoke an action.

Row anatomy: `[icon/avatar] [title + secondary context] [type/status] [action]`

Guidelines:

- Align icons and text consistently.
- Truncate long titles and descriptions without layout overflow.
- Highlight the keyboard-selected row with the theme's selection colour.
- Keep hover and keyboard-selection states visually consistent.
- Use badges sparingly; they should add meaning.
- Use distinct icons for genuinely different object types.
- If a result has a parent, optionally offer an "open location" action.
- Make the whole row clickable when there is one obvious primary action.
- Keep secondary actions separately targetable and labelled.
- Do not rely on colour alone for selection or status.

### Grouped results

For multiple object types, group by useful categories such as Pages, Files, Projects, People, Records, Commands/Actions, and Recent items. Show headings only when they improve scanning. Avoid empty categories. Allow a sensible global ranking across groups or a configurable grouping mode.

### Ranking and matching

Use a relevance model suited to the data. A useful baseline:

1. Exact title match.
2. Title begins with query.
3. Title contains query.
4. Important metadata contains query.
5. Description or path contains query.
6. Fuzzy matches where useful.

Tokenise multi-word queries and avoid results that match only an irrelevant fragment. Rank by relevance, then a stable secondary criterion. Do not change ranking between keystrokes without reason.

Where appropriate, search display names, aliases, descriptions, metadata, parent names, identifiers users are likely to know, tags, categories, command names, and synonyms.

Do not search private or inaccessible records merely because they exist in the data.

## 5. Keyboard and pointer interaction

| Key | Behaviour |
|---|---|
| Arrow Down | Select next result |
| Arrow Up | Select previous result |
| Enter | Activate selected result; if none, the top result if that is the product's convention |
| Escape | Close search, or first clear a transient sub-state |
| Ctrl+K / Cmd+K | Open global search/command palette |
| `/` | Optional shortcut outside text-entry controls |
| Tab / Shift+Tab | Move through controls in logical order |

Also:

- Keep the selected result in view.
- Support hover without erratic selection changes; pointer and keyboard must not fight.
- Announce result counts, loading, and no-result changes to assistive technology.
- Implement dialog, combobox, listbox, and option patterns with their required ARIA relationships and keyboard behaviour, not decorative roles.
- Maintain a visible focus indicator.

## 6. Search states

Design all of these, not just the populated list:

- **Idle:** prompt to type, or show recent/suggested items.
- **Typing:** query visible; results update.
- **Loading:** unobtrusive progress or status.
- **Results:** ranked or grouped rows.
- **Keyboard selection:** one clearly selected row.
- **No results:** say nothing matched and suggest changing the query.
- **Error:** explain the failure and offer retry.
- **Offline:** explain reduced capability.
- **Recent searches:** optional, user-controllable, privacy-aware.
- **Suggestions:** optional shortcuts or popular destinations.
- **No permission:** explain without exposing protected details.

Never show a blank panel without explanation.

## 7. Scope and filters

For complex applications, support compact scope filters such as All, Current project, Current workspace, or object type.

- Make the current scope visible and easy to reset to All.
- Avoid a large filter form in a small palette.
- Preserve filters only when users expect it.
- Use contextual scope by default only when obvious and reversible.
- Do not silently narrow global search to the current page.

## 8. Responsive and mobile behaviour

- **Desktop:** floating panel, moderate width, restrained backdrop; keyboard hints may sit in the footer.
- **Tablet:** reduce margins and width; keep input and results touchable.
- **Mobile:** near-full-width sheet or full screen; respect safe areas and the virtual keyboard; touch targets about 44 x 44 px or larger; hide unhelpful keyboard hints; keep close and clear easy to reach; never depend on hover.

## 9. Visual design and layering

- Reuse the host design system's tokens and components when it has them.
- Otherwise use semantic tokens (surface, text, muted text, border, accent, hover, selected, focus) instead of hard-coded colours, so themes apply automatically.
- Keep spacing (4, 8, 12, 16, 20, 24, 32 px), radius, typography, and icon weight consistent.
- Use accent colour with restraint and separate primary from secondary information.
- Transitions: open/close about 120-200 ms, hover/selection about 100-160 ms. No bounce or large scaling. Respect reduced motion.
- Give backdrop and panel a deliberate z-index above toolbars and menus, below transient notifications.
- Meet WCAG AA contrast in light and dark modes.

## 10. Implementation workflow

Before building:

1. Identify platform and framework.
2. Inspect the design system, navigation, and tokens.
3. Identify searchable objects and actions, and who may see them.
4. Identify responsive breakpoints and accessibility needs.

Suggested components:

```text
SearchTrigger
GlobalSearch
  SearchInput
  SearchFilters
  SearchSuggestions
  SearchResultGroups
    SearchResultRow
  SearchEmptyState
  SearchFooter
```

Implement only what the application needs. Separate search logic from presentation, reuse controls, ensure result actions do the correct application-specific thing, avoid unnecessary dependencies, and preserve existing functionality and data.

## 11. Testing checklist

- [ ] Trigger is discoverable.
- [ ] Global shortcut works and does not hijack typing.
- [ ] Input receives focus on open; focus returns on close.
- [ ] Results update correctly and ranking is sensible.
- [ ] Empty, loading, error, and offline states are handled.
- [ ] Arrow Up/Down selection works and Enter activates the correct result.
- [ ] Escape and backdrop click close the panel; inside clicks do not.
- [ ] Secondary actions work independently.
- [ ] Long text does not break layout.
- [ ] Mobile touch interaction works.
- [ ] Screen-reader semantics and announcements are correct.

## 12. Expected output

1. Inspect the application and its conventions.
2. Propose or implement search suited to its data and workflows.
3. Make it responsive, accessible, and keyboard-friendly.
4. Report what was implemented, assumptions, and limitations.
