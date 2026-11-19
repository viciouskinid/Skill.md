---
name: atlasforge-build-datetime-version
description: Mandatory whenever an AI assistant creates or edits code for an app that shows "Build YYYY-MM-DD HH:MM:SS" (with an optional version) in a footer, about box or status area. Every response that changes code must set the build stamp of every changed file as its final edit, from a real clock reading taken at that moment, or as "unknown" plus an incrementing version when no clock is available, verify it, and state it in the reply. Covers reading the clock, the exact format, where the stamp lives (one per delivered file), patch-delivery rules, failure modes, automation, the link to the changelog date, and a checklist.
---

# Build datetime / version

The app shows when its code was last changed, so the user can trust that they are running the latest version. When the AI has a real clock, the timestamp is the only marker. When the AI has no clock, the timestamp is `unknown` and a version number is added so the change is still traceable.

## 1. The rule

> **Any response that creates or modifies code must end its editing with an updated build stamp (and version, if there is no clock) in every file it changed, and must say what they are.**

- "Modifies code" means any change to a delivered file: logic, markup, styles, configuration, comments or whitespace. If the bytes change, that file's stamp changes.
- It does not apply to responses that change nothing (explanations, reviews, questions), or to files that are not code (for example a Markdown skill or notes file).
- The stamp edit is the **last** edit of the response, after all other changes, after tests and after any fix a test led to. If a response makes several edits, set the stamp once per file, at the end.
- A follow-up prompt that edits the code again gets a **new** stamp. Never reuse the previous value.
- With a real clock, the new stamp must be later than the previous real stamp.
- **Several files in one response:** every file whose bytes changed gets the **same** new stamp, from one clock reading. A file that did not change keeps its old stamp. Do not stamp a file you did not touch, and do not leave a touched file behind.

## 2. Two cases

### Case A: a real clock is available

- Use the current time in the user's timezone (section 4 says how to get it).
- Format: `YYYY-MM-DD HH:MM:SS` (24-hour, zero-padded, a single space, no timezone suffix).
- **No version number is shown.** When the timestamp is trustworthy, a version adds nothing. Set `BUILD_VERSION = null`.

### Case B: no clock is available

The AI has no tool that returns the time, no timestamp in the conversation or system context, and the user has not told it the time (ask first, see section 4).

- Set `BUILD_STAMP = 'unknown'`.
- Set `BUILD_VERSION` to an integer: `1` if the previous build had a real stamp, otherwise the previous version plus 1. It increases on every code change made without a clock.
- The footer shows the reason: `Build unknown · v3 (AI had no clock access — version bumped.)`.
- As soon as a real clock is available again, return to Case A: write the real stamp and set `BUILD_VERSION = null`.

## 3. One source of truth per file

Each delivered code file that carries a stamp holds exactly **one** pair of values, as plain constants that the AI edits. The footer (or About box) only displays them.

```html
<script>
/* BUILD STAMP: on EVERY code change set BUILD_STAMP to the current local time (YYYY-MM-DD HH:MM:SS).
   If no clock is available set BUILD_STAMP = 'unknown' and increment BUILD_VERSION; with a clock set BUILD_VERSION = null. */
const BUILD_STAMP   = '2026-01-01 09:30:00'; // BUILD_STAMP
const BUILD_VERSION = null;                  // BUILD_VERSION
const BUILD_NOTE    = 'AI had no clock access — version bumped.';
</script>
<span id="build" class="status-build" title="Build date and time (local)"></span>
<script>
document.getElementById('build').textContent = BUILD_STAMP !== 'unknown'
  ? 'Build ' + BUILD_STAMP
  : 'Build unknown · v' + BUILD_VERSION + ' (' + BUILD_NOTE + ')';
</script>
```

Rules:

- **Never compute the stamp in the browser.** Code like `new Date()` at page load, or a value read back from localStorage, shows when the page was *opened*, not when the code was *built*. The browser clock always works, so such code also makes the "no clock" case impossible to represent. The AI edits the constants; the page only displays them.
- Use the names `BUILD_STAMP` and `BUILD_VERSION`, with the `// BUILD_STAMP` and `// BUILD_VERSION` markers on their lines so both can be found by searching. If an existing file uses other names (for example `const BUILD` or a hardcoded `BUILD: …` in the markup), convert it to this pattern in the same change, unless the user says to keep theirs.
- Keep the maintenance comment directly above the constants.
- Do not duplicate the values in comments, titles or meta tags. A footer that is plain text in the markup must be filled from the constants, not typed separately.

### Projects with more than one file

- Prefer one stamp **per file that can be delivered or cached separately** (for example the main page and a plugin script it loads). Each has its own `BUILD_STAMP` / `BUILD_VERSION` pair and its own markers. This is what shows the user that a stale cached copy of one file is still being served while the other is current.
- The page's footer shows its own stamp. Secondary files report theirs to the page (for example through a small `addBuild(name, stamp)` hook), and the page lists them all, for instance in a hover or About box.
- Files that are always bundled and delivered together may share one stamp in a single file instead. State which approach the project uses in its standing instructions.
- Within one response the stamps of all changed files are identical (section 1).

## 4. Getting the real time

The AI must read a real clock. It must not guess, reuse the previous stamp, or use dates remembered from training. In order:

1. **A tool**: run a shell command in the user's timezone, for example `TZ=<IANA zone> date '+%Y-%m-%d %H:%M:%S'`, or call a time tool provided by the environment.
2. **A timestamp in the conversation or system context**, converted to the user's local zone.
3. **Ask the user** for the current time, if the conversation allows it and the change is not urgent.
4. **Otherwise use Case B.** Do not invent a time and do not add a minute to the old stamp.

The user's timezone: if it is not recorded in the project's instructions, ask once and record it there (for example `Timezone: Australia/Hobart`). Use local time including daylight saving. If the sandbox clock is UTC, convert it; never present UTC as local time.

Sanity check: if the clock returns a year before the current one (or a date earlier than the previous real stamp), treat the clock as suspicious and use Case B.

**One reading, taken last.** Read the clock once, as the final step before writing the stamps, and reuse that reading for every file stamp and for the changelog date (section 10). Do not read it at the start of a long task and carry the value forward.

## 5. Procedure for every code-changing response

1. Make all the requested code changes, and run any tests or checks, including fixes they lead to.
2. List the files whose bytes changed.
3. Read the clock (section 4). Choose Case A or Case B.
4. Set `BUILD_STAMP` and `BUILD_VERSION` in each changed file accordingly. This is the last edit.
5. **Verify**: search each changed file for the two markers. There must be exactly one `BUILD_STAMP` assignment and one `BUILD_VERSION` assignment per file; the stamp is `unknown` or matches the format and is later than the previous real stamp; the version is `null` with a real stamp and an integer when the stamp is `unknown`; all files changed in this response show the same stamp.
6. **Report** in one line at the end of the reply:
   - Case A: `Build stamp set to 2026-11-04 14:32:07 (index.html, gist_sync.js).`
   - Case B: `No clock available: build stamp unknown, version set to v3 (index.html).`

The reported line is evidence. A user who does not see it knows the rule was skipped.

## 6. When the response delivers patches instead of the whole file

Patch lists are where the stamp is most often forgotten.

- **The stamp edit is always item 0**, written out in full with the old and new lines, once for each file being patched:

  ```
  0. Build stamp (index.html)
  Find:    const BUILD_STAMP   = '2026-11-03 11:07:02'; // BUILD_STAMP
           const BUILD_VERSION = null;                  // BUILD_VERSION
  Replace: const BUILD_STAMP   = '2026-11-04 14:32:07'; // BUILD_STAMP
           const BUILD_VERSION = null;                  // BUILD_VERSION
  ```
- If the old values are unknown (the AI has only part of the file), tell the user to replace the two whole lines marked `BUILD_STAMP` and `BUILD_VERSION`.
- A single-snippet answer still includes the two lines and says where they go.

## 7. Failure modes to avoid

| Mistake | Why it happens | Fix |
|---|---|---|
| Stamp not touched | The change felt small | Any byte change counts |
| Placeholder or round time like `12:00:00` | No clock, or guessing | Read the clock; with none, use Case B |
| Copying the previous stamp | Laziness | Verify the stamp is new |
| Stamp computed with `new Date()` in the page | Looks automatic | It shows page-load time. Use edited constants |
| Clock read at the start of a long task | Convenient | Read it once, last (section 4) |
| Stamp set, then more edits or test fixes | Bad ordering | Stamp is the last edit |
| Version left in place after a real clock returns | Forgot to reset | Case A sets `BUILD_VERSION = null` |
| Version not incremented in Case B | Thinking "unknown" is enough | Each clockless change bumps it |
| One changed file stamped, another forgotten | Several files in one reply | List changed files; stamp all of them identically |
| Unchanged file re-stamped | Over-eager | Only changed files |
| Two stamps in the same file that differ | Duplicate locations | One pair of constants per file |
| Footer text typed by hand in the markup | Looks simpler | Fill it from the constants |
| UTC shown as local | Sandbox clock is UTC | Convert to the user's zone |
| Stamp in a different message from the code | Split work | Same response as the code change |
| Patches posted without the stamp | Patch delivery | Item 0 rule |
| No build line at the end of the reply | Forgot | Section 5 step 6 |

## 8. Make it hard to forget

1. **Say it in three places the assistant always sees:** this skill, the maintenance comment above the constants, and the project's standing instructions. Suggested standing instruction:

   > Whenever you create or modify any code file in this project, as your final edit set `BUILD_STAMP` to the current local time (`YYYY-MM-DD HH:MM:SS`, timezone `<zone>`) from a real clock and set `BUILD_VERSION = null`, in every file you changed, using one clock reading. If you cannot read a clock, ask me for the time; if that is not possible set `BUILD_STAMP = 'unknown'` and increase `BUILD_VERSION` by 1. Verify there is exactly one assignment of each per file, and end your reply with the build stamp line.

2. **Automate if there is a build step.** Prefer injecting the time at build time over editing by hand:

   ```sh
   # stamp.sh : run before saving or committing
   TZ=Australia/Hobart date '+%Y-%m-%d %H:%M:%S' | xargs -I{} \
     sed -i.bak "s/^const BUILD_STAMP *= '.*'; \/\/ BUILD_STAMP/const BUILD_STAMP   = '{}'; \/\/ BUILD_STAMP/; s/^const BUILD_VERSION *= .*; \/\/ BUILD_VERSION/const BUILD_VERSION = null;                  \/\/ BUILD_VERSION/" app.html && rm app.html.bak
   ```

   Wire it into a pre-commit hook, an npm `prebuild` script or a file watcher. With a bundler, inject the value at build time (a define plugin) and never edit it by hand.

3. **Check script.** A small script (pre-commit or CI) that fails when a file's modification time is more than a minute newer than its `BUILD_STAMP`, when more than one assignment of either constant exists in a file, or when `BUILD_VERSION` is set while the stamp is a real time.

4. **Single-file fallback.** Optionally show `document.lastModified` in the tooltip. It is not a replacement, but a mismatch with the stamp reveals a forgotten update.

5. **Visible freshness.** Show the stamp in the footer on every screen size, or at least in an About box.

## 9. Acceptance checklist

- [ ] Each stamped file has exactly one `BUILD_STAMP` and one `BUILD_VERSION` assignment, with markers and the maintenance comment.
- [ ] The footer text is generated from the constants only; no clock code runs in the page.
- [ ] With a clock: format `YYYY-MM-DD HH:MM:SS`, user's timezone, read last, later than before, `BUILD_VERSION = null`, footer reads `Build <stamp>`.
- [ ] Without a clock: stamp `unknown`, version incremented, footer shows the version and reason.
- [ ] Every file changed in the response has the same new stamp; unchanged files kept theirs.
- [ ] After a second code-changing prompt in the same chat, the stamp (or version) has changed again.
- [ ] Patch deliveries list the stamp edit as item 0 for each patched file.
- [ ] Every code-changing reply ends with the build line from section 5.
- [ ] If the project has a changelog, its entry date matches the date of the stamp (section 10).

## 10. Working with the changelog

If the project uses the changelog-keeper skill:

- A changelog entry's `date` is the date part of the same real clock reading used for the stamp (in the user's timezone). Never take the date from memory, from the previous entry, or from a guess.
- When there is no clock (Case B), ask the user for the date. If they cannot say, leave the date for them to fill in and say so. Do not invent one.
- The version in the changelog (`APP_VERSION`) and the build stamp are separate things: the changelog version marks a release, the build stamp marks the last edit. Updating one does not replace updating the other.
- A reply that changes code ends with both lines: the changelog line (version and entries) and the build line.
