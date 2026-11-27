---
name: settings-ui
description: Design, implement or review a settings overlay or page with header search across settings, category navigation, controls, a large theme system, custom themes, persistence, accessibility and mobile layout.
---

# Settings UI Skill

## Purpose

Use this skill when designing, implementing, reviewing, or improving the **settings experience** of any application: web apps, desktop applications, dashboards, admin tools, productivity software, content libraries, developer tools, and mobile applications.

The skill is application-agnostic. Do not reproduce the reference application's link tiles, folders, CSV editor, storage inspector, image cache, or business logic. Reuse only the transferable interaction and visual-design patterns.

The reference design uses:

- A settings overlay with a header containing the title, a **settings search**, and close.
- A persistent category navigation rail and a scrollable content panel.
- Clear section headings, helper text, compact controls, colour swatches, a custom colour input, and persistent preferences.
- A horizontal category strip in place of the rail on narrow screens.

Treat these as design principles, not rigid dimensions.

---

# Part A: Settings window

## 1. Design intent and layout

Settings should be one coherent, predictable control centre, not a collection of popups.

- Compact application: modal settings overlay (default).
- Larger product: full-page settings route.
- Mobile: full-screen settings view.

Layout:

- **Header:** title, settings search, close/back. Stays fixed.
- **Navigation rail:** category list. Stays visible on desktop.
- **Content panel:** independently scrollable setting sections.

The active category must be obvious.

## 2. Information architecture

Include only categories that have real settings. Category library:

- **General:** behaviour, startup and default views, language and locale, date/time/number/unit formats, default actions, confirmations.
- **Appearance:** themes, light/dark/system, accent, density, typography, font size, motion, icons.
- **Search:** default scope, content types, result density, recent history, shortcuts, suggestions and ranking.
- **Notifications:** in-app, email/push, quiet hours, sound and desktop alerts, per-category preferences.
- **Privacy and Security:** session and sign-in, privacy, data visibility, permissions, security options.
- **Data and Storage:** usage, cache, import/export, backup/restore, retention, offline data.
- **Accessibility:** reduced motion, high contrast, larger text, keyboard, screen reader, focus visibility.
- **Integrations:** connected services, API preferences, sync, connection status.
- **Advanced:** expert options, diagnostics, developer and experimental features, reset.
- **About:** version, build, licences, help and documentation.

Combine categories for small apps and split them for large ones. Avoid empty sections and very long navigation lists.

## 3. Panels and controls

Each panel has a section title, a short sentence on the effect of the settings, grouped related controls, optional dividers, inline validation or status, and consistent save/apply behaviour.

- **Toggle:** binary preference that applies immediately.
- **Checkbox:** independent option.
- **Radio group:** one of a small set.
- **Select:** one of a longer set.
- **Slider:** continuous or stepped value, current value visible.
- **Text/number input:** explicit value.
- **Colour picker:** custom colour.
- **Button:** export, clear, test, reset, reconnect.

Guidelines:

- Put labels before or beside controls, never only in placeholders.
- Explain non-obvious consequences; show units; show errors next to the control.
- Distinguish destructive actions and confirm irreversible or consequential ones.
- Do not add Save buttons to immediate settings, or imply immediate saving when changes are staged.
- For staged changes, show unsaved status with Apply/Save and Cancel/Discard.
- Offer reset-to-default where useful.
- Persist reliably and handle storage failures gracefully.

## 4. Navigation and responsive behaviour

Desktop: left rail about 180-240 px, readable labels with optional icons, soft theme-derived active background with stronger text, independently scrolling content, stable header.

Mobile: turn the rail into a horizontal category strip, compact list, or nested page. Do not shrink the rail until labels are unreadable. Use full width, touch-friendly controls, and avoid nested scrolling.

- Switching category must not lose unsaved edits.
- Preserve the active category on reopening if useful.
- Use current-page or current-tab semantics.
- Support deep links to categories when the app has routes.
- Explain why a category is unavailable instead of showing a broken control.
- Restore focus to the invoking control on close.

---

# Part B: Settings search

## 5. Scope and placement

Every settings window has its own search that finds **settings only**: never application content. It is part of the window, not a floating palette, and needs no recent searches, suggestions, or global shortcut.

- Place it in the settings header between the title and close button, about 240-340 px wide.
- On narrow screens (about 640 px and below), move it to a full-width second header row.
- Placeholder: "Search settings". Show a clear button when there is text.
- It stays visible while content scrolls and while switching categories.
- If a global search exists, its shortcut must not open over the settings overlay.

## 6. Behaviour

| State | Window behaviour |
|---|---|
| Empty query | Normal category panels |
| Typing | Results replace the content panel, grouped by category. The rail shows a match count per category and dims those with none |
| No matches | Name the query in a short message and suggest another word |
| Result chosen | Clear the query, open its category, expand any collapsed section, scroll the setting to centre, highlight it for about 2 s, focus its control |
| Category clicked while searching | Clear the query and open that category |
| Query cleared | Return to the previous category and scroll position |
| Window closed | Reset the query |

Choosing a result navigates to the setting; it never changes its value or runs its action. Searching never discards staged edits.

Each result shows the setting label (matched text marked), its helper text, and its category heading. Add the section name when labels repeat.

## 7. What to index

Index each setting's label, helper text, category, section, option names, and synonyms (for example "dark", "night" for colour mode).

- Use one source of truth. For static HTML, annotate the setting rows. If panels are rendered lazily, use a settings registry (`id`, `category`, `section`, `label`, `help`, `keywords`, `options`) that also drives rendering, so unopened categories are searchable.
- Include dynamic settings such as integrations, and settings under Advanced.
- Exclude settings the user is not permitted to see.
- List disabled settings with the reason they are unavailable.
- Never index or display stored values of passwords, tokens, keys, or personal data.
- Theme names may be indexed: choosing one opens the theme gallery with its theme search filled in, without applying the theme. The gallery's own search stays local to the gallery.

## 8. Matching, keyboard, and accessibility

Match case- and accent-insensitively. Every word in the query must match somewhere. Rank exact label, label prefix, label, keywords, options, category, then helper text. Order category groups by their best result. Local search needs no debounce.

| Key (inside settings) | Behaviour |
|---|---|
| `/` | Focus settings search when not in a text-entry control |
| Arrow Down / Up | Move through results, wrapping |
| Enter | Go to the selected result (first is preselected) |
| Escape | Clear a non-empty query; a second press closes settings |

- Use a combobox input controlling a listbox of grouped options, with `aria-expanded` and `aria-activedescendant`. Focus stays in the input while arrowing.
- Announce result counts and no-result messages through a status region.
- On open, focus the search on desktop. On touch devices, focus the active category so the keyboard does not pop up.
- The found-setting highlight uses an outline as well as colour.

## 9. Reference implementation (vanilla HTML)

Runnable starting point. Add `data-setting`, `.setting-label`, `.setting-help`, and optional `data-keywords` to each row. Restyle to the host application. It covers the behaviour above for static rows; add a registry for lazy rendering, and write disabled reasons into helper text.

```html
<button id="open-settings">Settings</button>
<dialog id="settings" class="settings" aria-labelledby="settings-title">
  <header class="settings-header">
    <h2 id="settings-title">Settings</h2>
    <div class="settings-search">
      <input id="settings-query" type="text" role="combobox" autocomplete="off"
             aria-label="Search settings" aria-controls="settings-results"
             aria-expanded="false" aria-autocomplete="list" placeholder="Search settings">
      <button type="button" id="settings-clear" aria-label="Clear search" hidden>&times;</button>
    </div>
    <button type="button" id="settings-close" aria-label="Close settings">&times;</button>
  </header>
  <div class="settings-body">
    <nav class="settings-nav" aria-label="Settings categories">
      <button type="button" data-category="general" aria-current="page">General</button>
      <button type="button" data-category="appearance">Appearance</button>
    </nav>
    <div class="settings-content">
      <p id="settings-status" role="status"></p>
      <div id="settings-results" role="listbox" aria-label="Matching settings" hidden></div>
      <section data-panel="general">
        <h3>General</h3>
        <div class="setting" data-setting data-keywords="startup launch home">
          <label class="setting-label" for="start-view">Start view</label>
          <p class="setting-help">The page shown when the app opens.</p>
          <select id="start-view"><option>Dashboard</option><option>Recent items</option></select>
        </div>
      </section>
      <section data-panel="appearance" hidden>
        <h3>Appearance</h3>
        <div class="setting" data-setting data-keywords="dark night light theme">
          <label class="setting-label" for="mode">Colour mode</label>
          <p class="setting-help">Follow the device or choose a fixed mode.</p>
          <select id="mode"><option>System</option><option>Light</option><option>Dark</option></select>
        </div>
      </section>
    </div>
  </div>
</dialog>

<style>
.settings { width: min(820px, 94vw); height: min(600px, 88dvh); padding: 0;
  border: 1px solid var(--color-border, #dfe3eb); border-radius: 14px;
  background: var(--color-surface, #fff); color: var(--color-text, #202432); overflow: hidden; }
.settings[open] { display: flex; flex-direction: column; }
.settings::backdrop { background: rgb(0 0 0 / .45); }
.settings [hidden] { display: none !important; }
.settings-header { display: grid; grid-template-columns: 1fr minmax(0, 340px) auto;
  grid-template-areas: "title search close"; align-items: center; gap: 12px;
  padding: 12px 16px; border-bottom: 1px solid var(--color-border, #dfe3eb); }
.settings-header h2 { grid-area: title; margin: 0; font-size: 1.15rem; }
.settings-search { grid-area: search; display: flex; align-items: center;
  border: 1px solid var(--color-border, #dfe3eb); border-radius: 8px; }
.settings-search:focus-within { outline: 2px solid var(--color-focus, #536dfe); }
.settings-search input { flex: 1; min-width: 0; height: 36px; padding: 0 10px;
  border: 0; outline: 0; background: none; color: inherit; font: inherit; }
.settings-header button { min-width: 36px; height: 36px; border: 0; background: none;
  color: inherit; font-size: 1.25rem; cursor: pointer; }
#settings-close { grid-area: close; }
.settings-body { flex: 1; min-height: 0; display: grid; grid-template-columns: 200px 1fr; }
.settings-nav { display: flex; flex-direction: column; gap: 4px; padding: 12px;
  border-right: 1px solid var(--color-border, #dfe3eb); overflow: auto; }
.settings-nav button { display: flex; justify-content: space-between; gap: 8px; padding: 9px 12px;
  border: 0; border-radius: 8px; background: none; color: inherit; font: inherit;
  text-align: left; white-space: nowrap; cursor: pointer; }
.settings-nav button[aria-current] { background: var(--color-accent-soft, #edf0ff); font-weight: 600; }
.settings-nav button[data-count]::after { content: attr(data-count); opacity: .7; }
.settings-nav button[data-count="0"] { opacity: .45; }
.settings-content { overflow: auto; padding: 16px 20px; }
.settings-content h3 { margin-top: 0; }
#settings-status { margin: 0 0 8px; color: var(--color-text-muted, #697386); }
#settings-status:empty { margin: 0; }
.setting { padding: 12px; border-radius: 8px; }
.setting-label { display: block; font-weight: 600; }
.setting-help { margin: 4px 0 8px; color: var(--color-text-muted, #697386); }
.setting.is-found { background: var(--color-selected, #e7ebff);
  outline: 2px solid var(--color-focus, #536dfe); }
.result-group { margin: 12px 0 4px; font-size: .8rem; font-weight: 600;
  color: var(--color-text-muted, #697386); }
.result { padding: 10px 12px; border-radius: 8px; border-left: 3px solid transparent; cursor: pointer; }
.result[aria-selected="true"] { background: var(--color-selected, #e7ebff);
  border-left-color: var(--color-accent, #536dfe); }
.result small { display: block; color: var(--color-text-muted, #697386); }
.result mark { background: none; color: inherit; text-decoration: underline 2px var(--color-accent, #536dfe); }
@media (max-width: 640px) {
  .settings { width: 100vw; max-width: 100vw; height: 100dvh; max-height: 100dvh; border: 0; border-radius: 0; }
  .settings-header { grid-template-columns: 1fr auto; grid-template-areas: "title close" "search search"; }
  .settings-search input { height: 44px; font-size: 16px; }
  .settings-body { display: flex; flex-direction: column; }
  .settings-nav { flex: none; flex-direction: row; border-right: 0;
    border-bottom: 1px solid var(--color-border, #dfe3eb); }
  .settings-content { flex: 1; min-height: 0; }
}
</style>

<script>
(() => {
  const $ = (s) => document.querySelector(s);
  const dialog = $('#settings'), input = $('#settings-query'), clearBtn = $('#settings-clear');
  const list = $('#settings-results'), status = $('#settings-status');
  const content = dialog.querySelector('.settings-content');
  const tabs = [...dialog.querySelectorAll('[data-category]')];
  const panels = [...dialog.querySelectorAll('[data-panel]')];
  let category = tabs[0].dataset.category;
  let results = [], active = 0, searching = false, savedScroll = 0, flashTimer;

  const fold = (s) => s.normalize('NFD').replace(/\p{M}/gu, '').toLowerCase();
  const text = (el, sel) => el.querySelector(sel)?.textContent.trim() ?? '';

  // Read at search time, so rows added or relabelled later are included.
  const buildIndex = () => panels.flatMap((panel) => {
    const cat = panel.dataset.panel;
    const catLabel = tabs.find((t) => t.dataset.category === cat).textContent.trim();
    return [...panel.querySelectorAll('[data-setting]:not([hidden])')].map((row) => ({
      row, cat, catLabel,
      label: text(row, '.setting-label'),
      help: text(row, '.setting-help'),
      keywords: row.dataset.keywords ?? '',
      options: [...row.querySelectorAll('option')].map((o) => o.textContent).join(' '),
    }));
  });

  // Every word must match somewhere. Label beats keywords, options, category, help.
  function score(item, words) {
    const fields = [[item.label, 50], [item.keywords, 30], [item.options, 20],
                    [item.catLabel, 10], [item.help, 5]].map(([s, w]) => [fold(s), w]);
    let total = 0;
    for (const word of words) {
      const hit = fields.find(([s]) => s.includes(word));
      if (!hit) return 0;
      total += hit[1];
    }
    const label = fields[0][0], phrase = words.join(' ');
    return total + (label === phrase ? 100 : label.startsWith(phrase) ? 50 : 0);
  }

  function showCategory(id) {
    category = id;
    tabs.forEach((t) => t.dataset.category === id
      ? t.setAttribute('aria-current', 'page') : t.removeAttribute('aria-current'));
    panels.forEach((p) => { p.hidden = p.dataset.panel !== id; });
    content.scrollTop = 0;
  }

  function marked(label, word) {
    const span = document.createElement('span');
    const at = label.toLowerCase().indexOf(word);
    if (at < 0) { span.textContent = label; return span; }
    const mark = document.createElement('mark');
    mark.textContent = label.slice(at, at + word.length);
    span.append(label.slice(0, at), mark, label.slice(at + word.length));
    return span;
  }

  function search() {
    const words = fold(input.value).split(/\s+/).filter(Boolean);
    clearBtn.hidden = !input.value;
    if (!words.length) return endSearch();
    if (!searching) { savedScroll = content.scrollTop; searching = true; }

    const hits = buildIndex().map((item) => ({ ...item, score: score(item, words) }))
      .filter((item) => item.score > 0).sort((a, b) => b.score - a.score);
    // Map keeps insertion order, so the category with the best hit comes first.
    const groups = new Map();
    hits.forEach((h) => groups.set(h.cat, [...(groups.get(h.cat) ?? []), h]));
    results = [...groups.values()].flat();

    list.replaceChildren(...[...groups.values()].map((items) => {
      const group = document.createElement('div');
      group.setAttribute('role', 'group');
      group.setAttribute('aria-label', items[0].catLabel);
      const heading = document.createElement('div');
      heading.className = 'result-group';
      heading.setAttribute('aria-hidden', 'true');
      heading.textContent = items[0].catLabel;
      group.append(heading, ...items.map((item) => {
        const i = results.indexOf(item);
        const option = document.createElement('div');
        option.className = 'result';
        option.id = `settings-result-${i}`;
        option.dataset.index = i;
        option.setAttribute('role', 'option');
        const help = document.createElement('small');
        help.textContent = item.help;
        option.append(marked(item.label, words[0]), help);
        return option;
      }));
      return group;
    }));

    panels.forEach((p) => { p.hidden = true; });
    tabs.forEach((t) => {
      t.dataset.count = groups.get(t.dataset.category)?.length ?? 0;
      t.removeAttribute('aria-current');
    });
    list.hidden = !results.length;
    input.setAttribute('aria-expanded', String(results.length > 0));
    status.textContent = results.length
      ? `${results.length} matching setting${results.length === 1 ? '' : 's'}`
      : `No settings match "${input.value.trim()}". Try another word.`;
    content.scrollTop = 0;
    select(0);
  }

  function select(i) {
    if (!results.length) return input.removeAttribute('aria-activedescendant');
    active = (i + results.length) % results.length;
    list.querySelectorAll('[role="option"]').forEach((o) =>
      o.setAttribute('aria-selected', String(Number(o.dataset.index) === active)));
    input.setAttribute('aria-activedescendant', `settings-result-${active}`);
    $(`#settings-result-${active}`).scrollIntoView({ block: 'nearest' });
  }

  function endSearch(restore = true) {
    input.value = '';
    clearBtn.hidden = true;
    list.hidden = true;
    list.replaceChildren();
    status.textContent = '';
    results = [];
    input.setAttribute('aria-expanded', 'false');
    input.removeAttribute('aria-activedescendant');
    tabs.forEach((t) => delete t.dataset.count);
    if (!searching) return;
    searching = false;
    showCategory(category);
    if (restore) content.scrollTop = savedScroll;
  }

  function jumpTo(item) {
    if (!item) return;
    endSearch(false);
    showCategory(item.cat);
    dialog.querySelectorAll('.is-found').forEach((r) => r.classList.remove('is-found'));
    clearTimeout(flashTimer);
    item.row.classList.add('is-found');
    flashTimer = setTimeout(() => item.row.classList.remove('is-found'), 2000);
    item.row.closest('details')?.setAttribute('open', '');
    let target = item.row.querySelector('input, select, textarea, button');
    if (!target || target.disabled) { target = item.row; target.tabIndex = -1; }
    target.focus({ preventScroll: true });
    item.row.scrollIntoView({ block: 'center' });
  }

  function openSettings() {
    if (dialog.open) return;
    dialog.showModal();
    // Do not raise the on-screen keyboard just because settings opened.
    if (matchMedia('(pointer: coarse)').matches) tabs.find((t) => t.dataset.category === category).focus();
    else input.focus();
  }

  $('#open-settings').addEventListener('click', openSettings);
  $('#settings-close').addEventListener('click', () => dialog.close());
  clearBtn.addEventListener('click', () => { endSearch(); input.focus(); });
  tabs.forEach((t) => t.addEventListener('click', () => {
    endSearch(false);
    showCategory(t.dataset.category);
  }));
  input.addEventListener('input', search);
  input.addEventListener('keydown', (e) => {
    if (e.isComposing) return;
    if (e.key === 'ArrowDown' && results.length) { e.preventDefault(); select(active + 1); }
    else if (e.key === 'ArrowUp' && results.length) { e.preventDefault(); select(active - 1); }
    else if (e.key === 'Enter') { e.preventDefault(); jumpTo(results[active]); }
    else if (e.key === 'Escape' && input.value) { e.preventDefault(); e.stopPropagation(); endSearch(); }
  });
  list.addEventListener('pointermove', (e) => {
    const o = e.target.closest('[role="option"]');
    if (o && Number(o.dataset.index) !== active) select(Number(o.dataset.index));
  });
  list.addEventListener('mousedown', (e) => e.preventDefault()); // keep focus in the input
  list.addEventListener('click', (e) => {
    const o = e.target.closest('[role="option"]');
    if (o) jumpTo(results[Number(o.dataset.index)]);
  });
  dialog.addEventListener('keydown', (e) => {
    if (e.key === '/' && !e.target.matches('input, textarea, select, [contenteditable]')) {
      e.preventDefault();
      input.focus();
    }
  });
  dialog.addEventListener('click', (e) => { if (e.target === dialog) dialog.close(); });
  dialog.addEventListener('close', () => endSearch());
})();
</script>
```

---

# Part C: Theme system

## 10. Requirements

The theme selector is a first-class part of Appearance. Provide complete themes, not just accent colours. Each theme defines semantic tokens for page background, surfaces, primary/secondary/muted text, borders, accent and text on accent, hover/selected/focus/pressed states, success/warning/danger/info, and input, card, menu, dialog, tooltip and navigation surfaces, with optional shadow, radius, and density tokens.

Components must consume tokens, never hard-coded colours:

```css
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

## 11. Theme selector

Layout: heading and explanation, mode selector, theme search, category filters, gallery of preview cards, selected indicator, customisation, custom theme save/create, reset.

Each card has a name, a miniature-interface preview (cards, text, buttons, selection state, not just colour dots), optional light/dark label and tag, a clear selected indicator, and preview or immediate apply. Apply without reload and persist across sessions.

## 12. Modes

- **Light** and **Dark:** the theme's respective palette.
- **System:** follow the OS preference and update when it changes.
- **Auto (optional):** schedule or context, only if genuinely supported.

Changing mode keeps the selected theme. Where no dark palette exists, generate a carefully adjusted variant (not an inversion) and let users refine it.

## 13. Curated library

Provide at least 60 named themes, preferably 80 or more, with light and dark variants wherever practical. Starting catalogue:

- **Neutral and professional:** Classic Light, Classic Dark, Soft Grey, Graphite, Slate, Charcoal, Midnight, Monochrome, Paper, Warm Paper, Office Blue, Corporate Navy, Steel, Platinum, Minimal.
- **Blue and cyan:** Ocean Blue, Deep Ocean, Arctic, Ice, Sky, Azure, Cobalt, Sapphire, Denim, Pacific, Lagoon, Glacier, Blue Steel, Electric Blue, Midnight Blue.
- **Purple and pink:** Lavender, Soft Lilac, Violet, Royal Purple, Amethyst, Orchid, Plum, Grape, Magenta, Rose, Blush, Dusty Pink, Raspberry, Neon Violet, Cosmic.
- **Green and nature:** Forest, Deep Forest, Sage, Mint, Seafoam, Emerald, Jade, Moss, Olive, Fern, Pine, Aurora Green, Tropical, Eucalyptus, Celadon.
- **Warm and earth:** Sand, Desert, Terracotta, Clay, Copper, Bronze, Amber, Honey, Sunset, Sunrise, Coral, Peach, Apricot, Autumn, Coffee, Mocha, Cocoa, Burgundy, Wine, Crimson.
- **High-contrast, vivid, experimental:** High Contrast Light, High Contrast Dark, Neon, Cyberpunk, Synthwave, Matrix, Terminal, Retro Green, Retro Amber, Arcade, Solarized Light, Solarized Dark, Nord, Dracula-inspired, Gruvbox-inspired, Tokyo Night-inspired, Monokai-inspired, Pastel, Candy, Rainbow.

This is a target, not a limit. Add distinct, coherent themes and avoid near-duplicates. For themes inspired by third-party ones, use an original compatible palette and imply no affiliation or exact reproduction.

## 14. Discovery

- Categories: All, Light, Dark, Neutral, Professional, Blue, Purple and Pink, Green, Warm, High Contrast, Retro and Terminal, Colourful, Custom.
- Search by name; filter by family and mode; sort by name or curated order.
- Optional favourites and recently used.
- Clear selected state and compact theme count.
- Use a responsive filtered grid, never an unstructured wall of swatches.

## 15. Custom theme builder

- Fields: name, base mode, background, surface, primary and secondary text, border, accent, optional status colours.
- Optional: accent presets, custom accent, surface tint, radius, density, font scale, shadow intensity.
- Actions: live preview, save as new, duplicate, rename, delete with confirmation, revert to saved, JSON import/export where appropriate.
- Warn when contrast makes text or controls unreadable and recommend accessible alternatives.

## 16. Accent colour

Offer presets (Blue, Indigo, Violet, Purple, Magenta, Pink, Rose, Red, Coral, Orange, Amber, Gold, Yellow, Lime, Green, Emerald, Teal, Cyan, Sky, Navy, Slate, Graphite, Brown, Copper), a custom picker, and reset.

Accent updates primary buttons, links, focus indicators, selected navigation and tabs, selection highlights, relevant icons and borders, and progress indicators. Generate hover, soft, border, and contrast variants; do not paint every component with it. A custom accent overrides the theme accent without destroying the theme.

## 17. Other appearance controls

Where suitable: density (Compact/Comfortable/Spacious), corners (Sharp/Subtle/Rounded/Pill), typography (System/Sans/Serif/Mono), text size (Small/Default/Large/Extra large), motion (System/Reduced/Off), shadows (None/Subtle/Standard/Strong), navigation style (Standard/Compact/Expanded), icon style (Outline/Filled/Minimal). These must change real tokens and components, not only previews.

## 18. Persistence and architecture

- A registry of built-in and user themes.
- Selected theme ID, mode, optional accent and appearance overrides.
- One `applyTheme()` mapping configuration to tokens.
- A persistence layer and a schema migration strategy.
- Web: CSS custom properties; localStorage or a settings backend; apply before first paint; listen to `prefers-color-scheme` in System mode; handle storage exceptions; validate imported JSON.

```json
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

Never hard-code the palette into components, or let theme changes overwrite application data or unrelated preferences.

---

# Part D: Quality and workflow

## 19. Accessibility

- WCAG AA contrast for normal text wherever possible, with stronger options available.
- Never rely on colour alone for selected, error, success, or warning.
- Visible focus rings in every theme.
- Respect reduced motion.
- Correct dialog focus management and Escape behaviour.
- Semantic labels and descriptions; full keyboard operation.
- No tiny swatches or touch targets; legible disabled controls.
- Test overlays, tooltips, menus, and charts in light and dark; keep status colours distinguishable in every theme.

## 20. Visual design, motion, and layering

- Reuse the host design system when there is one.
- Match the application's spacing (4, 8, 12, 16, 20, 24, 32 px), typography, icons, borders, and button hierarchy.
- Radius: controls 6-8 px, cards 10-14 px, dialogs 12-18 px; pills only for chips and pill controls.
- Subtle hover/selected states (100-160 ms), theme change 120-220 ms without flashes, no bounce.
- Define z-index order: content, sticky bars, menus, search backdrop, settings dialog, tooltips and notifications.
- Close transient menus on outside click, Escape, or navigation. Never close the dialog while the user interacts with its content.

## 21. Implementation workflow

Before building: identify platform and framework; inspect the design system; identify genuinely configurable settings; decide immediate vs staged saving; determine persistence scope (account or device); identify breakpoints and accessibility needs; check for an existing theme system to extend.

Suggested components:

```text
SettingsTrigger
Settings
  SettingsHeader
    SettingsSearch
  SettingsNavigation
  SettingsSearchResults
  SettingsContent
    GeneralSettings
    AppearanceSettings
      ThemeModeSelector, ThemeSearch, ThemeFilters,
      ThemeGallery, AccentColourPicker, CustomThemeEditor
    (other categories as needed)
```

Do not create empty components. Separate settings state from presentation and the theme registry from the gallery. Use reusable controls, one settings persistence service, safe defaults, and validated imports. Hide settings the user cannot see. Avoid unnecessary dependencies and preserve existing functionality and data.

## 22. Testing checklist

Settings:

- [ ] Opens and closes reliably; focus returns.
- [ ] Category navigation and active state work; content scrolls independently.
- [ ] Settings persist at the intended scope.
- [ ] Unsaved changes are handled; reset and destructive actions are confirmed.
- [ ] Mobile navigation is usable; controls have labels and helper text.

Settings search:

- [ ] Sits in the header on desktop and wraps to its own row on mobile.
- [ ] Finds settings in unopened categories, by label, synonym, and option name.
- [ ] Rail counts match the results.
- [ ] Choosing a result opens the category, highlights and focuses the setting, and changes no value.
- [ ] Arrows, Enter, and `/` work; Escape clears, then closes.
- [ ] Clearing restores category and scroll; reopening starts empty.
- [ ] No-result state is announced; hidden and secret data never appear.

Themes:

- [ ] At least 60 distinct themes with coherent tokens.
- [ ] Light, dark, and system work; previews are accurate.
- [ ] Applies without reload and persists.
- [ ] Custom accent works; custom themes can be created, saved, duplicated, renamed, deleted.
- [ ] Import/export is validated.
- [ ] All states legible, contrast checked, reduced motion respected, application data unaffected.

## 23. Expected output

1. Inspect the application and its conventions.
2. Implement settings suited to its real configuration needs, with header search.
3. Include a searchable, filterable gallery of at least 60 themes, preferably 100.
4. Include light, dark, and system modes, a custom accent picker, and a working custom theme builder.
5. Use semantic tokens throughout.
6. Make it responsive, accessible, keyboard-friendly, and persistent.
7. Preserve existing functionality and data.
8. Report what was implemented, assumptions, and limitations.
