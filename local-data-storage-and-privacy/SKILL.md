---
name: local-data-storage-and-privacy
description: Use when an app stores data in the browser (localStorage, IndexedDB) or handles files the user uploads, imports or drops in. Defines what goes where, and the hard rule that uploaded data is sensitive and must never be sent over the network. Covers import, export, backup, clearing data, diagnostics and a no-leak checklist.
---

# Local storage and data privacy

## 1. The hard rules

Treat everything the user imports, uploads, drops, pastes or draws as **sensitive and confidential**.

1. **Uploaded and created data never leaves the device.** No upload, POST, beacon, WebSocket message, email link, share link, cloud sync or third-party API call may carry it, in whole or in part, in any form (including names, coordinates, counts of private items, file names or thumbnails).
2. **No analytics, telemetry, error reporting or session replay** that could include user content. Do not add them, and do not add SDKs that phone home.
3. **No sending user data to AI or other web services** (summarise, classify, geocode, translate, reverse-geocode), unless the user explicitly asks for a named feature and is told exactly what will be sent. Default answer is no.
4. **Files are read locally** with the File API (`file.text()`, `arrayBuffer()`) and processed in the browser. Never use a `<form>` that submits files.
5. **User data never goes into URLs** (path, query or hash), clipboard automatically, or logs. `console.log` must not print content.
6. If a requested feature would break a rule above, stop and ask the user instead of building it.

When unsure whether something is "user data", assume it is.

## 2. What the network may be used for

Allowed, because no user content is sent, but each one must be disclosed in the UI or docs:

| Use | What it reveals | Guidance |
|---|---|---|
| Map tiles | Area being viewed | Disclose. Keep to a short list of known hosts. |
| Place-name search | Text typed into the search box | Optional and clearly labelled. Offer a switch to disable. Coordinates are parsed locally and never sent. |
| App scripts, styles, fonts | The user's IP and that the app was opened | Prefer bundling or self-hosting. If a CDN is unavoidable, pin an exact version and add `integrity` (SRI) and `crossorigin`. |

Everything else is blocked by default. The app should work fully offline except tiles and search.

### Enforce it

Add a Content-Security-Policy that makes leaks impossible, adjusted to the hosts actually used:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'none';
  script-src 'self' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: blob: https://tile.example.org;
  font-src 'self';
  connect-src 'self' https://geocoder.example.org;
  worker-src 'self' blob:;
  form-action 'none'; base-uri 'none'; frame-ancestors 'none'">
<meta name="referrer" content="origin">
```

Notes: `connect-src` lists only hosts the app may call with `fetch`; tile images load through `img-src`. Prefer script hashes over `'unsafe-inline'` where possible. Limit the referrer to the origin so the full page URL is never sent; a few tile providers require a referrer, so test. Do not register service workers that sync or phone home.

## 3. Where data lives

| Kind of data | Store | Why |
|---|---|---|
| Small preferences: theme, panel widths, last basemap, feature flags, tool settings | `localStorage`, keys prefixed with an app namespace (`myapp:theme`) | Synchronous, tiny, read at startup before first paint. |
| User content: items, folders, geometry, notes, archive | IndexedDB | Large, structured, asynchronous, survives reloads. |
| Large binary files the user loaded (elevation tiles, rasters, attachments) | IndexedDB, separate object store | Keeps the main dataset light. |
| Transient data that must not persist (previews, photos with location, test data) | Memory only | Gone on reload. Revoke `URL.createObjectURL` URLs when finished. |

Rules:

- **Never put user content in localStorage**: it is synchronous (blocks the UI), about 5 MB, string-only, and easy to dump.
- **Never use cookies.**
- Decide persistence per data class and tell the user. Ask when it is not obvious. Session-only data must not appear in saved lists, exports or undo history unless the user opts in.
- Do not store derived data that can be recomputed (caches, thumbnails) unless it is large and slow to rebuild; mark it as cache so "clear" can drop it.

### localStorage helpers

```js
const NS = 'myapp:';
const prefGet = (k, d) => { try { const v = localStorage.getItem(NS + k); return v === null ? d : JSON.parse(v); } catch { return d; } };
const prefSet = (k, v) => { try { localStorage.setItem(NS + k, JSON.stringify(v)); } catch {} };
```

Always wrap in try/catch (private mode and quota can throw). Apply theme from localStorage in a tiny inline script in `<head>` so there is no flash.

### IndexedDB layout

```
database  "myapp"  (version N)
  store "data"      keyPath "id"   → one record per top-level collection, or one per entity for big sets
  store "blobs"     keyPath "key"  → large binary payloads
  store "settings"  keyPath "key"  → settings too structured for localStorage
```

Rules:

- Open the database once and cache the promise. Wrap `get`, `put`, `getAll`, `delete`, `clear` in small promise helpers that resolve on `tx.oncomplete`.
- Use `onupgradeneeded` for versioned migrations. Every record gets a `schemaVersion`. Write a migration function per version bump and never change the shape of stored data without one.
- Store plain JSON-compatible objects: no functions, no DOM or map-library objects. Rebuild runtime objects (map layers, DOM) from the stored data on load.
- Use stable unique ids for every entity.
- For very large datasets, store entities individually (or in chunks of a few thousand) so one change rewrites a chunk, not everything.
- **Debounce writes** (about 500 ms) and flush on `visibilitychange` (hidden) and `pagehide`. Provide a "save now" internal function for actions that need certainty (delete, import).
- Handle `QuotaExceededError`: show a clear message and offer export. Show usage with `navigator.storage.estimate()` in settings.
- Call `navigator.storage.persist()` after the user has stored meaningful data, so the browser is less likely to evict it. Tell the user it may still be lost (clearing site data, private windows, storage pressure), so they should export backups.
- Multiple tabs: detect a second tab (BroadcastChannel or `navigator.locks`) and warn, or last-writer-wins will silently lose edits.

### Startup

Restore preferences, then data, then render. Show a plain loading screen until restoration finishes, so the user never sees an empty "no data" state that then fills in a few seconds later.

## 4. Import

- Support drag and drop, a file picker, and, where useful, folder selection (`webkitdirectory`) and dropped folders (`webkitGetAsEntry`, which must be called synchronously during the drop event). Allow adding several files and folders before confirming.
- Show a report before or after: files found, accepted, skipped and why, duplicates.
- Validate and sanitise every imported value before it reaches the DOM: escape text, whitelist colours and icon class names, clamp numbers. Imported files are untrusted input.
- Parse large files in chunks with a progress bar.
- Imports go in their own folder by default; "import directly" is an explicit choice for restoring the app's own exports.
- Read-only inspection (for example reading EXIF GPS from photos) happens in memory and does not copy the file anywhere.

## 4a. Export and backup

- Exports are user-initiated downloads built from a `Blob` and `URL.createObjectURL`; revoke the URL after a few seconds.
- Offer a full backup (everything, including settings that matter) and scoped exports (selected items, one folder). Say what each format keeps and loses.
- Warn that an exported file contains all the data it covers and is no longer protected by the app.
- Text cells starting with `= + - @` must be prefixed when exporting CSV to stop spreadsheet formula injection.
- Offer a periodic "last backed up" reminder in settings.

## 5. Clearing data

Provide a settings section "Clear data" with separate, confirmed actions:

- Clear view (hide everything, keep data).
- Clear cached data (tiles in memory, derived caches).
- Clear stored files (large binaries).
- Clear all user content.
- **Erase everything**: delete the IndexedDB database(s) with `indexedDB.deleteDatabase`, remove all namespaced localStorage keys, clear any Cache API entries, revoke object URLs, reset in-memory state and undo history, and then reload.

Each states what will be lost and that it cannot be undone (unless the undo history covers it, in which case offer Undo in a toast).

## 6. Diagnostics and logging

- Debug or "copy diagnostics" output contains counts, timings, settings and browser info only. No names, descriptions, coordinates, file names or file contents unless the user explicitly ticks an opt-in.
- Error toasts may show technical detail but must not echo user content.
- Remove debugging helpers (stress tests, diagnostics buttons) once the issue is solved, or leave them clearly labelled as temporary.

## 7. When adding or changing features

Before writing code, ask: does this send anything off the device? Does it store something new? Where, in which store, and how is it cleared? Does it appear in exports, undo and diagnostics? Update the clear-data section and the disclosure text in the same change.

## 8. Acceptance checklist

- [ ] With dev tools open on the Network tab, load a file, edit, export and clear: there are no requests except tiles, optional search and static assets.
- [ ] Search the code for `fetch(`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `<form`, `<img src` built from data, `new Image().src`, `navigator.share`: each use is justified and carries no user data.
- [ ] CSP is present and lists only the needed hosts.
- [ ] localStorage holds only small preferences, all namespaced.
- [ ] Data survives reload; the "empty state" never flashes before restore.
- [ ] Schema version and a migration path exist.
- [ ] Erase everything leaves no IndexedDB database, no namespaced keys and no caches.
- [ ] Session-only data is absent from saved data, exports and history, and is gone after reload.
- [ ] Diagnostics output was read through and contains no content.
