---
name: undo-history
description: Use when adding undo, redo or a history list to an app that edits structured data (items, folders, shapes, settings of objects). Covers what is undoable, snapshot versus patch design, when to record a step, human-readable step labels, the header buttons and history dropdown, keyboard shortcuts, restoring state safely, memory limits, and large datasets.
---

# Undo, redo and history

## 1. Scope

**Undoable (document changes):** create, delete, rename, move, reorder, restyle, edit geometry or properties, show/hide, archive/restore, import, bulk edits.

**Not undoable:** pan and zoom, selection, panel sizes, theme and other preferences, open dialogs, in-progress drawing (cancel it instead), transient previews.

**Excluded from history by design:** large reloadable data (terrain files, rasters), session-only items, synthetic test data. Say so in the UI help.

History lives in memory only and is lost on reload. Do not persist it, because it duplicates sensitive data (see the `local-data-storage-and-privacy` skill). Tell the user.

## 2. Model

A list of entries and a pointer:

```js
let hist = [];      // { id, snap, label, t }
let histIdx = -1;   // current entry
```

- `snap` is a serialised copy of the document state (JSON string). Plain strings are cheap to compare (`===`) and cannot be mutated by accident.
- `label` is a human description of the change that produced the entry (section 4).
- Entry 0 is the baseline ("Opened", or "Current state" after Clear).
- A new edit **truncates the redo tail**: `hist.splice(histIdx + 1)` then push.
- Undo = go to `histIdx - 1`; redo = go to `histIdx + 1`; jump = go to any id.

### Snapshot or patch?

| Dataset size | Approach |
|---|---|
| Up to a few thousand items | Whole-document JSON snapshots, capped by count and total characters. Simple and robust. |
| Larger | Store **patches or commands** (per-entity before/after, or an operation record with its inverse) instead of full snapshots, or snapshot only the changed entities plus a periodic full snapshot. |
| Very large | Offer a setting to disable history above a size threshold, and say so. |

Always measure: show the time to serialise a snapshot and the history size in diagnostics. Snapshotting the whole document on every edit is O(n) and is a common cause of lag with big data.

Limits: at most 50 entries and about 40 million characters in total; drop the oldest entries first, always keeping the current one.

## 3. When to record a step

Record **after a change is committed**, not while it is happening.

- Drag, resize or vertex move: one step on release.
- Typing in a field: one step after idle (about 500 ms) or on blur.
- Bulk operations: one step for the whole operation, labelled with the count.
- Imports: one step.
- Many code paths change data, so hook recording into the single function that persists the document (debounced save) rather than into each feature. That function calls `trackUndo()` first.
- `trackUndo()`: serialise; if identical to the current entry, return; otherwise derive the label from the previous and new state, truncate redo, push, trim to limits, refresh the buttons.
- While restoring, set a `busy` flag so recording is skipped.

Serialise only document state. Exclude flagged transient and excluded items (section 1) from the snapshot.

## 4. Readable labels

Derive the label by diffing the previous and new state. Flatten both into `Map(id → {node, parent, index, archived, metaString})`, then classify:

| Detected | Label |
|---|---|
| ids only in new | `Added placemark “Name”`; one top-level folder with children → `Added “Folder” (12 items)`; several → `Added 5 items` |
| ids only in old | `Deleted …` (same pattern) |
| archived flag turned on / off | `Archived “X”` / `Restored “X”` |
| name changed | `Renamed “Old” → “New”` |
| visibility changed | `Hid “X”` / `Showed “X”`; several → `Hid 8 items` |
| parent changed | `Moved “X” to “Folder”` or `to top level`; only index changed → `Reordered items` |
| metadata changed | only coordinates → `Moved “X” on map`; geometry points → `Edited shape “X”`; otherwise `Edited “X”`; several → `Edited 6 items` |

If there are two kinds of change, join with " · "; three or more → `Multiple changes`; nothing recognised → `Changed data`. Truncate names to about 28 characters. Use "top-level" nodes only when counting (ignore descendants of an added or deleted parent).

## 5. Restoring a step

`gotoHistory(id)`:

1. Refuse with a toast if the user is mid-drawing ("Finish or cancel drawing first").
2. **Flush** any pending debounced save first (it may add an entry).
3. Close the properties panel, stop any live previews and tracking, close dependent panels (charts).
4. Set `busy`. Replace the in-memory document from the snapshot, keeping excluded items (terrain, test data, session-only items) that are not part of snapshots.
5. Rebuild all derived state: map layers, lists, minimap, labels, indexes. If the current folder no longer exists, go to the top level. Clear selection (or keep it only if the id still exists).
6. Set `histIdx` to the entry, save the restored document to storage, clear `busy`, refresh buttons.

Use a busy overlay if the dataset is large.

## 6. User interface

- **Header group** at the right: Undo, Redo and a small caret that opens the history dropdown. Disabled states are real (`disabled`), not just dimmed.
- Tooltips show the action: `Undo: Renamed “A” → “B” (Ctrl+Z)`, `Redo: … (Ctrl+Shift+Z)`.
- **History dropdown**: title "History" with a Clear button (keeps the current state as the new baseline). Newest entry first. Each row: dot, label, time (or "Current"). The current row is highlighted; rows after the pointer (redo-able) are greyed with a dashed dot. Click a row to jump there. Arrow keys move, Escape closes and returns focus to the caret. Footer: "Click a step to jump to it · greyed steps are redo-able · this session only".
- **Keyboard:** `Ctrl/Cmd+Z` undo; `Ctrl/Cmd+Shift+Z` or `Ctrl/Cmd+Y` redo. **Ignore** the shortcut when focus is in an input, textarea, select or contenteditable so native text undo works.
- **Toasts with Undo:** destructive actions (delete, clear, bulk delete, archive) show a toast with an Undo button that calls undo once.
- Toast on undo and redo: `Undid: <label>`, `Redid: <label>`. "Nothing to undo" when at the start.

## 7. Compact reference implementation

```js
const LIMIT = 50, MAX_CHARS = 40e6;
let hist = [], idx = -1, last = null, busy = false, seq = 0;

function snapshot() { return JSON.stringify(serialiseDocument()); }   // excludes transient items

function reset(label = 'Opened') {
  const s = snapshot(); hist = [{ id: ++seq, snap: s, label, t: Date.now() }];
  idx = 0; last = JSON.parse(s); refreshButtons();
}
function track() {                       // call from the single "document saved" function
  if (busy) return;
  const s = snapshot();
  if (idx < 0) return reset();
  if (s === hist[idx].snap) return;
  const obj = JSON.parse(s);
  hist.splice(idx + 1);
  hist.push({ id: ++seq, snap: s, label: describeChange(last, obj), t: Date.now() });
  idx = hist.length - 1; last = obj;
  let total = hist.reduce((a, h) => a + h.snap.length, 0);
  while (hist.length > LIMIT || (hist.length > 2 && total > MAX_CHARS)) { total -= hist.shift().snap.length; idx--; }
  refreshButtons();
}
async function goto(id) {
  if (isDrawing()) return toast('Finish or cancel drawing first');
  await flushPendingSave();
  const i = hist.findIndex(h => h.id === id); if (i < 0 || i === idx) return false;
  busy = true;
  try { closeEditors(); const o = JSON.parse(hist[i].snap); replaceDocument(o); rebuildDerivedState(); idx = i; last = o; await saveNow(); }
  finally { busy = false; }
  refreshButtons(); return true;
}
const undo = () => idx > 0 && goto(hist[idx - 1].id);
const redo = () => idx < hist.length - 1 && goto(hist[idx + 1].id);
```

## 8. Pitfalls

- Recording inside drag or input events creates hundreds of entries.
- Forgetting the busy flag while restoring creates a phantom entry that wipes the redo tail.
- Not flushing the debounced save before undo loses the last edit.
- Snapshots that include runtime objects (map layers, DOM nodes) fail to serialise or leak memory. Store plain data only.
- Unstable ids make diffs report everything as deleted and re-added.
- Keeping stale references to old node objects after a restore. Re-look everything up by id.
- Excluded data (test data, session-only items) accidentally included, bloating memory.

## 9. Acceptance checklist

- [ ] Every edit type in section 1 undoes and redoes correctly, including deleting a folder with children.
- [ ] A drag is one step; typing a name is one step.
- [ ] Labels read naturally for single and bulk changes.
- [ ] Making a new edit after undo drops the redo steps.
- [ ] The dropdown jumps to any step, greys future steps, and works by keyboard.
- [ ] Undo is blocked mid-drawing; text-field undo still works natively.
- [ ] Destructive actions offer Undo in the toast.
- [ ] Memory stays within limits after 200 edits on a large dataset; snapshot time is reported in diagnostics.
- [ ] Reload clears history and nothing about it is written to storage.
