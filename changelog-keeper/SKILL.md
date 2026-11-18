---
name: changelog-keeper
description: Use whenever an AI creates or modifies an application's code (new feature, bug fix, refactor, UI tweak, dependency change). Requires a succinct change log entry for every change, preserved history, and a dedicated in-app changelog page, linked from a single row in Settings/About, without cluttering the existing UI.
---

# Changelog Keeper

Every change to an application ends with a changelog entry. No entry, no finished task.

## 1. When to write an entry
Write one for every delivered change that alters behaviour, UI, data format, or dependencies. Skip only for pure whitespace or comment edits. Several edits in one request = one entry with several lines.

A version is one release, not one message. A release is everything the user receives until they move on to something unrelated or say it has shipped. When work on the same app continues over several messages in one session, keep adding lines to the current top entry instead of creating a new version for each message.

## 2. Entry format
Newest first. Each entry:

```
{ v: "1.4.2", date: "2026-11-17", changes: [
    { t: "fixed", text: "CSV import keeps quoted commas", cause: "Row split used naive split(',')" },
    { t: "added", text: "Dark mode toggle in Settings" }
]}
```

- `v`: semver. Fix or tweak = patch, new feature = minor, breaking change or data-format change = major.
- `date`: ISO date the entry was delivered. Do not backdate, and do not split one day's work into several dated entries.
- `t`: one of `added`, `changed`, `fixed`, `removed`.
- `text`: one line, 12 words or fewer, plain language, says what the user will notice. No file names, no jargon.
- `cause`: required for `fixed` (root cause, one line). Optional otherwise (`why`).
- Never write "various improvements" or "misc fixes".

### When to bump the version
- Bump only when the current top entry has already been released. If the top entry is still unreleased, add lines to it and leave `v` alone. Choose the bump (patch, minor or major) from the biggest change in the entry, and raise it if a later line in the same entry needs a bigger one.
- If you are not sure whether the top entry has been released, ask once: "Is v1.4.2 already live, or should I add to it?" Do not guess.
- A fix to something added earlier in the same unreleased entry needs no new `fixed` line. Reword the original line instead. If the fix is a separate bug the user would have noticed, add a `fixed` line.
- The first changelog added to an app that never had one starts at the version the user names. If they do not name one, treat the existing app as 1.0.0 and make the first entry 1.1.0.

## 3. Where it lives (source of truth)
- **Single-file HTML app:** one block, kept between markers, edited in place:
  ```
  /* CHANGELOG:START */
  const APP_VERSION = "1.4.2";
  const CHANGELOG = [ ...entries... ];
  /* CHANGELOG:END */
  ```
- **Multi-file project:** `changelog.json` (same schema) loaded by the app, plus a generated `CHANGELOG.md` for humans and repos.
- `APP_VERSION` must equal `CHANGELOG[0].v`. Show it in the About section.

## 4. History rules
- Append new entries at the top. Never edit, reorder, merge or delete released entries (except fixing a typo the user asks about).
- The top entry may be edited, reworded or merged with entries made in the same session until the user confirms it has been released. Once released, it follows the rule above.
- If a change reverts an earlier release, add a new `changed`/`removed` entry referencing the old version.
- Do not trim history. If the block exceeds ~200 entries, move older ones to `changelog-archive.json` and keep a "View older" link.

## 5. In-app display
Goal: a dedicated changelog page that is easy to reach and never in the way.
- **Entry point:** one row in Settings → About, labelled `Changelog`, showing the version (`v1.4.2`) and a "What’s new" link. This is the only thing added to existing screens. If the app has no settings, reuse an existing version label or menu item. No new toolbar buttons, banners, popups or toasts.
- **The page:** its own view with its own route (`#changelog` in single-file apps, `/changelog` in routed apps) and a Back button. Never embed the entry list inside Settings or About; they only link to it. If the app already uses plain names as routes (for example folders or pages), use a reserved name such as `#_changelog` so it cannot clash. Make sure the new route does not let the app's normal actions (adding items by paste or drag, undo, search shortcuts) act on it.
- **Unread marker:** when the version differs from the last one the user opened, show a small dot on the Settings entry and the Changelog row only. Clear it when the page opens. Store last-seen version in `localStorage` (wrapped in try/catch, with an in-memory fallback so the dot still clears for the session).
- **Page contents:** entries grouped by version and date, newest expanded, older collapsed; coloured type tags; `cause` shown as a muted second line; filter chips (All / Added / Changed / Fixed / Removed); text search; "Copy as Markdown" and "Download CHANGELOG.md".
- Respect the app’s existing theme, fonts and spacing. Support light and dark if the app does.
- Render with `textContent`, never inject entry text as HTML.

## 6. Procedure for every task
1. Make the requested change.
2. Check whether the current top entry is released (section 2). If it is unreleased, add to it. If it is released, start a new entry with the right version bump. If unsure, ask.
3. Write or update the entry lines (section 2).
4. Update the changelog block and `APP_VERSION` together.
5. If the app has no changelog page yet, add the page and the single About row following section 5 as part of the same change (entry line: `added` "Changelog page in Settings > About").
6. Verify: the About row opens the page, Back works, the newest entry matches the work done, released entries are untouched, and `APP_VERSION` equals `CHANGELOG[0].v`.
7. In your reply, state the version and the entry lines in one short list, and say whether you added to an unreleased version or started a new one.

## 7. Do not
- Do not put the full entry list inside About or Settings, add a changelog panel that is always visible, a modal on startup, or a toast per update.
- Do not log internal refactors that have no user-visible effect, unless they changed data formats.
- Do not rebuild the changelog from scratch; always edit the existing block.
- Do not create a new version for every message in a session; one release is one version.
- Do not backdate entries or split one day's work across several dated entries.
