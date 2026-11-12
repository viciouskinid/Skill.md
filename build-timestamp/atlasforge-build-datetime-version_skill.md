---
name: atlasforge-build-datetime-version
description: Mandatory whenever an AI assistant creates or edits code for an app that shows "Build YYYY-MM-DD HH:MM:SS" (with an optional version) in a footer, about box or status area. Every response that changes code must set the build stamp as its final edit, from a real clock when one is available, or as "unknown" plus an incrementing version when it is not, verify it, and state it in the reply. Covers reading the clock, the exact format, where the stamp lives, patch-delivery rules, failure modes, automation and a checklist.
---

# Build datetime / version

The app shows when its code was last changed, so the user can trust that they are running the latest version. When the AI has a real clock, the timestamp is the only marker. When the AI has no clock, the timestamp is `unknown` and a version number is added so the change is still traceable.

## 1. The rule

> **Any response that creates or modifies code must end its editing with an updated build stamp (and version, if there is no clock), and must say what they are.**

- "Modifies code" means any change to the delivered file(s): logic, markup, styles, configuration, comments or whitespace. If the bytes change, the stamp changes.
- It does not apply to responses that change nothing (explanations, reviews, questions).
- The stamp edit is the **last** edit of the response, after all other changes. If a response makes several edits, set it once, at the end.
- A follow-up prompt that edits the code again gets a **new** stamp. Never reuse the previous value.
- With a real clock, the new stamp must be later than the previous real stamp.

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

## 3. One source of truth

Exactly **one** place holds the values, as plain constants that the AI edits. The footer only displays them.

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
- Keep the `// BUILD_STAMP` and `// BUILD_VERSION` markers on their lines so both can be found by searching.
- Keep the maintenance comment directly above the constants.
- Do not duplicate the values in comments, titles or meta tags.
- In multi-file projects the constants live in one file only (for example `build.js` or the main entry file).

## 4. Getting the real time

The AI must read a real clock. It must not guess, reuse the previous stamp, or use dates remembered from training. In order:

1. **A tool**: run a shell command in the user's timezone, for example `TZ=<IANA zone> date '+%Y-%m-%d %H:%M:%S'`, or call a time tool provided by the environment.
2. **A timestamp in the conversation or system context**, converted to the user's local zone.
3. **Ask the user** for the current time, if the conversation allows it and the change is not urgent.
4. **Otherwise use Case B.** Do not invent a time and do not add a minute to the old stamp.

The user's timezone: ask once and record it in the project's instructions (for example `Timezone: Australia/Hobart`). Use local time including daylight saving. If the sandbox clock is UTC, convert it; never present UTC as local time.

Sanity check: if the clock returns a year before the current one (or a date earlier than the previous real stamp), treat the clock as suspicious and use Case B.

## 5. Procedure for every code-changing response

1. Make all the requested code changes.
2. Try to read the clock (section 4). Choose Case A or Case B.
3. Set `BUILD_STAMP` and `BUILD_VERSION` accordingly. This is the last edit.
4. **Verify**: search the file for the two markers. There must be exactly one `BUILD_STAMP` assignment and one `BUILD_VERSION` assignment; the stamp is `unknown` or matches the format and is later than the previous real stamp; the version is `null` with a real stamp and an integer when the stamp is `unknown`.
5. **Report** in one line at the end of the reply:
   - Case A: `Build stamp set to 2026-11-04 14:32:07.`
   - Case B: `No clock available: build stamp unknown, version set to v3.`

The reported line is evidence. A user who does not see it knows the rule was skipped.

## 6. When the response delivers patches instead of the whole file

Patch lists are where the stamp is most often forgotten.

- **The stamp edit is always item 0**, written out in full with the old and new lines:

  ```
  0. Build stamp
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
| Placeholder time like `12:00:00` | No clock | Use Case B instead |
| Copying the previous stamp | Laziness | Verify the stamp is new |
| Stamp computed with `new Date()` in the page | Looks automatic | It shows page-load time. Use edited constants |
| Version left in place after a real clock returns | Forgot to reset | Case A sets `BUILD_VERSION = null` |
| Version not incremented in Case B | Thinking "unknown" is enough | Each clockless change bumps it |
| Updating the stamp early, then making more edits | Bad ordering | Stamp is the last edit |
| Two stamps that differ | Duplicate locations | Single constant |
| UTC shown as local | Sandbox clock is UTC | Convert to the user's zone |
| Stamp in a different message from the code | Split work | Same response as the code change |
| Patches posted without the stamp | Patch delivery | Item 0 rule |

## 8. Make it hard to forget

1. **Say it in three places the assistant always sees:** this skill, the maintenance comment above the constants, and the project's standing instructions. Suggested standing instruction:

   > Whenever you create or modify any code file in this project, as your final edit set `BUILD_STAMP` to the current local time (`YYYY-MM-DD HH:MM:SS`, timezone `<zone>`) from a real clock and set `BUILD_VERSION = null`. If you cannot read a clock, ask me for the time; if that is not possible set `BUILD_STAMP = 'unknown'` and increase `BUILD_VERSION` by 1. Verify there is exactly one assignment of each, and end your reply with the build stamp line.

2. **Automate if there is a build step.** Prefer injecting the time at build time over editing by hand:

   ```sh
   # stamp.sh : run before saving or committing
   TZ=Australia/Hobart date '+%Y-%m-%d %H:%M:%S' | xargs -I{} \
     sed -i.bak "s/^const BUILD_STAMP *= '.*'; \/\/ BUILD_STAMP/const BUILD_STAMP   = '{}'; \/\/ BUILD_STAMP/; s/^const BUILD_VERSION *= .*; \/\/ BUILD_VERSION/const BUILD_VERSION = null;                  \/\/ BUILD_VERSION/" app.html && rm app.html.bak
   ```

   Wire it into a pre-commit hook, an npm `prebuild` script or a file watcher. With a bundler, inject the value at build time (a define plugin) and never edit it by hand.

3. **Check script.** A small script (pre-commit or CI) that fails when the file's modification time is more than a minute newer than `BUILD_STAMP`, when more than one assignment of either constant exists, or when `BUILD_VERSION` is set while the stamp is a real time.

4. **Single-file fallback.** Optionally show `document.lastModified` in the tooltip. It is not a replacement, but a mismatch with the stamp reveals a forgotten update.

5. **Visible freshness.** Show the stamp in the footer on every screen size, or at least in an About box.

## 9. Acceptance checklist

- [ ] Exactly one `BUILD_STAMP` and one `BUILD_VERSION` assignment, with markers and the maintenance comment.
- [ ] The footer text is generated from the constants only; no clock code runs in the page.
- [ ] With a clock: format `YYYY-MM-DD HH:MM:SS`, user's timezone, later than before, `BUILD_VERSION = null`, footer reads `Build <stamp>`.
- [ ] Without a clock: stamp `unknown`, version incremented, footer shows the version and reason.
- [ ] After a second code-changing prompt in the same chat, the stamp (or version) has changed again.
- [ ] Patch deliveries list the stamp edit as item 0.
- [ ] Every code-changing reply ends with the build line from section 5.
