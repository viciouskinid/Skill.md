---
name: ai-debug-overlay
description: Adds a hotkey-toggled, draggable debug overlay to any HTML application. It captures console errors, uncaught exceptions, failed resource loads and every fetch/XHR request and response, and has checkboxes for optional sources the AI can ask the human to switch on - cookies, localStorage, sessionStorage, IndexedDB, service workers/caches, WebSocket/EventSource, click/input trail, page events, DOM mutations, performance, environment, accessibility checks, DOM snapshot and registered app state - plus log persistence across page loads and a before/after baseline comparison (for example a login page versus the signed-in page). The human browses it all in the overlay and copies a structured report back to the AI. Use this skill whenever you create or substantially modify an HTML page, single-file HTML app, web app, game, tool, dashboard, prototype or frontend project (vanilla JS, React, Vue, Svelte, Vite, Next, etc.), and whenever the user mentions debugging, console errors, network errors, failed requests, cookies, login or session problems, sign-in not working, saved data not persisting, slowness, 'it's broken / blank screen / nothing happens', or wants a way to give you feedback about runtime behavior. Include it by default in new HTML apps, even if the user doesn't ask, because it is the cheapest way to close the feedback loop when you cannot see the browser console yourself.
---

# AI Debug Overlay

You usually cannot see the browser console or network tab of the app you build. The person testing it can, but copying errors by hand is slow and lossy: they forget the request body, miss the failed call that preceded the crash, or paste only the last line of a stack trace.

This skill fixes that by embedding a small, dependency-free script in the app. One hotkey opens an overlay that summarizes what went wrong and copies a complete, AI-readable report to the clipboard. The human pastes it into the chat, and you get the errors, the stack traces, the exact requests/responses that led up to them, and (when the AI has asked the human to switch them on) cookie, storage, WebSocket, performance, DOM and app-state details.

## What it captures

Always on:

- `console.error`, `console.warn`, plus `log/info/debug` as lower-priority context
- Uncaught exceptions (`window.onerror`) and unhandled promise rejections
- Failed resource loads (`<img>`, `<script>`, `<link>` 404s) and CSP violations
- Every `fetch` and `XMLHttpRequest`: method, URL, status, duration, request headers/body, response headers/body (text-like content only, truncated)
- Environment basics: page URL, user agent, viewport, session uptime

Everything else is a **source** with a checkbox (overlay > *Capture settings*), so the AI can ask for exactly the information it needs and nothing more. Cookies are on by default; all others are off until someone ticks them.

| Source id | What you get |
|---|---|
| `cookies` (on) | Live list of JS-visible cookies (name, value, size; plus domain, path, expiry, Secure, SameSite where the Cookie Store API exists). Flags duplicate names. Change log of every add/change/remove with the stack of whoever wrote it and the raw `document.cookie = "..."` assignment. Writes that changed nothing are logged as `no-op` (rejected `Secure`, wrong path, and so on). |
| `localStorage`, `sessionStorage` | Live key/value list with sizes; change log (`setItem`/`removeItem`/`clear`) with call stacks; `QuotaExceededError` and friends logged as errors. |
| `indexedDB` | Every database, object store, record count, key path, index, and a sample of records per store; write log for `put/add/delete/clear`. |
| `cache` | Service worker registrations and states, Cache API names and cached URLs. Explains "stale after deploy" bugs. |
| `appState` | Values the app registered with `__aiFeedback.watch(name, () => state)` (Redux, Pinia, any object). Values are masked by name. |
| `ws` | WebSocket and EventSource: connect, open, each message in/out, close code, unclean closes flagged as errors. Only connections created after the source is on (use the capture link to get them from page load). |
| `postMessage` | Messages received from other windows/iframes (origin, redacted data). |
| `input` | Click trail, committed form input (values masked; **passwords never recorded**), form submits with method, action and field names, special keys and shortcuts (never typed characters). |
| `nav` | `pushState`/`replaceState`/`popstate`/`hashchange`, tab visibility, online/offline, pagehide. |
| `mutations` | DOM changes under selectors given to `__aiFeedback.observe(selector)` (or the Watch box on the Activity tab). |
| `perf` | FCP, LCP, CLS, first input delay, navigation timings, long tasks, slow resources (> `slowResourceMs`), JS heap trend. |
| `env` | Device, viewport, locale/timezone, connection type, storage quota, permission states (geolocation, notifications, camera, microphone, clipboard). Answers "works for me, not for them". |
| `a11y` | Basic DOM sanity checks: images without alt, unlabeled controls, nameless buttons/links, duplicate ids, missing `lang`/`title`/viewport meta, `h1` count. Not a full audit. |
| `dom` | Current HTML with scripts, secrets, typed values and `data:` URIs stripped, truncated to `maxDomChars`. The closest thing to "what the page looks like" (there is no screenshot). |
| `persist` | Keeps the log, page-load list and baseline across redirects and reloads in this tab (stored in `sessionStorage`, redacted). Needed to follow a login redirect. |

Two more tools sit on top of the sources:

- **Compare tab / baseline**: *Mark baseline* on one page (for example the login page), do something (sign in), then open the Compare tab or copy the report. It lists what changed: URL and title, cookie/localStorage/sessionStorage names (values masked, real values compared by hash), elements that appeared or disappeared (links, buttons, fields, headings, forms), simple signals (password fields, logout/login controls, auth-looking cookies and storage keys, before and after), and the requests made since the baseline. Marking switches `persist` on automatically.
- **Page loads**: with `persist` on (or after any reload), the report lists every page load in the tab with navigation type, redirect count, HTTP status of the document, referrer and title, and entries are labelled `p1`, `p2`… Cookie/storage changes that happened *between* two pages (a server `Set-Cookie` on a redirect, for example) are logged as `via: across page load`.

Sensitive data is redacted automatically: auth/cookie headers; values under keys like `password`, `token`, `secret`, `api_key` in JSON bodies, **urlencoded form bodies** (`password=…`) and URL query strings; JSON-ish pairs inside other text (socket.io frames, log lines). Cookie/storage/state values are also masked when the **name** looks sensitive (`session`, `sid`, `auth`, `jwt`, `csrf`, `token`, `secret`, …) or the value looks like a JWT. Persisted data and baselines hold masked values only (plus a hash to detect changes). The **Redact secrets** checkbox turns masking off for the current page load only (it is never remembered). The human should still glance at a report before pasting it anywhere public.

Limits worth knowing:

- Page scripts cannot see **HttpOnly** cookies (most real session cookies), and `Set-Cookie` response headers are hidden from `fetch`/XHR. Server-set JS-visible cookies are noticed within ~1.5s (`via: external/poll`). For HttpOnly sessions, infer sign-in from later API calls (200 vs 401) and from the page itself (Compare tab).
- Only `fetch`/XHR are recorded as requests. A classic form post or link click is a page navigation, so you see the **submit event** (action, method, field names) and the **page load** record (redirects, status), but not the intermediate redirect hops or the navigation request's headers.
- Pages opened from `file://` generally cannot store cookies. Serve over `http://localhost` if cookies matter.
- Sandboxed iframes (including some artifact viewers) block cookies and storage entirely; the Cookies tab says so.
- The overlay only runs on pages that include it. A login on another site (OAuth/SSO) is not recorded; the log is normally still there when you return to the app in the same tab.
- Direct property writes like `localStorage.foo = 1` bypass `setItem`, so they appear via the timer diff (no call stack).
- History-type sources only record from the moment they are on. To record from page load, set the flag in `CONFIG.capture`, or open the page with `?aidebug=ws,perf,persist,open` (the **Copy capture link** button builds this for you; `all` ticks everything and `open` shows the overlay on load).
- WebSocket/EventSource and the DOM/storage patches wrap browser APIs; code that grabbed the original `fetch`, `WebSocket` etc. before the script ran is not seen, which is why the script must be first.

## Hotkey and overlay

- **Alt+Shift+D** toggles the overlay (uses the physical `D` key, so it works across keyboard layouts and on macOS where Option changes the produced character). `Esc` closes it.
- **Draggable and resizable**: drag the title bar to move it anywhere (it stays on screen); drag the bottom-right corner to resize; double-click the title bar to snap it back to the bottom-right corner. The position is remembered across reloads.
- Top: summary chips (errors, warnings, failed requests, slowest request, plus counts for enabled sources)
- Tabs: **Problems** (default), **Network** (with a *Page loads* section), **Console**, **Cookies**, **Storage**, **Realtime**, **Activity**, **Perf**, **Page** (environment, service workers/caches, accessibility checks, DOM snapshot, app state), **Compare**. Click a row to expand it; JSON is pretty-printed and URL-encoded values show their decoded form. A tab for a source that is off says so and has a *Turn on* button.
- **Capture settings** (collapsed under the tabs): one checkbox per source with a one-line description, *Tick all*, *Untick all*, and **Copy capture link** (a URL that switches the ticked sources on from page load).
- A **filter box** searches the current tab. **↻ Refresh** re-reads cookie attributes, IndexedDB, caches, DOM, etc.
- Buttons: **Copy report**, **Copy errors only**, **Download .md**, **Download .json**, **Clear**, **Close**
- If the browser blocks clipboard access (sandboxed iframes, some artifact viewers), the overlay reveals a pre-selected text box so the human can press Ctrl/Cmd+C manually.
- Typing in the overlay does not trigger the app's own keyboard shortcuts.
- The overlay stores its checkbox choices and position under one localStorage key (`__aiDebugPrefs`) and, with `persist`, a log under one sessionStorage key (`__aiDebugLog`). It never reports on either.

## Asking the human for more information

Every report starts with a **Capture on / Capture off** list and, if anything is off, a line telling you how to get more. When the report does not contain what you need, do not guess: tell the human exactly what to tick and what to do, for example:

> Press **Alt+Shift+D**, open *Capture settings*, tick **Keep log across page loads**, **Cookies**, **localStorage**, **Click / input trail** and **Page events**, go to the *Compare* tab and press **Mark baseline** on the login page, then sign in, then press **Copy report** and paste it here.

Or give them a link that starts with the right sources already on: `http://localhost:3000/login?aidebug=cookies,localStorage,input,nav,persist,open`.

Which sources to request for common symptoms:

| Symptom | Ask for |
|---|---|
| Can't sign in / logged out unexpectedly / "is it actually signed in?" | `persist` + Compare baseline on the login page, `cookies`, `localStorage`, `sessionStorage`, `input`, `nav`, `dom`, `appState` (if the app exposes an auth flag) |
| Data not saving / vanishing | `localStorage`, `sessionStorage`, `indexedDB` (the change log shows who wrote or removed each key) |
| Live updates / chat / notifications not arriving | `ws`, `nav` (offline events), `env` |
| Slow, janky, memory growing | `perf` |
| Old version shows after a deploy | `cache` (service worker and cache contents) |
| Works for me, not on their device | `env`, `a11y`, `dom`, `perf` |
| Button does nothing / UI not reacting | `input`, `mutations` on the relevant container, `dom`, `appState` |
| Embedded widget / iframe misbehaving | `postMessage`, `nav` |
| Layout broken | `dom`, `env`, `a11y` |

When you **build** an app with a central state store (Redux, Pinia, Zustand, a plain object), register it once so state can be requested later, and expose an explicit auth signal if there is a login:

```js
window.__aiFeedback?.watch('store', () => store.getState());
window.__aiFeedback?.watch('auth', () => ({ signedIn: !!currentUser, userId: currentUser?.id }));
```

## How to install it

The script must run **before any other script** on the page. Otherwise early errors and the app's first requests are missed, and those are often the most important ones.

1. Copy the script from "The script" below verbatim into a `<script>` tag as the **first child of `<head>`**.
   - Single-file HTML: paste it inline.
   - Multi-file project: save it as `ai-debug-overlay.js` (e.g. `public/` in Vite/Next/CRA) and add `<script src="/ai-debug-overlay.js"></script>` as the first script in `index.html` (for Next.js App Router, load it with `<Script strategy="beforeInteractive">` in the root layout).
   - Do not add `defer`, `async`, or `type="module"` to it.
2. Tell the user, in one short line, that the hotkey exists: "Press **Alt+Shift+D** to open the debug overlay (drag it by the title bar); use *Copy report* and paste it back to me if anything breaks." Add that extra sources (cookies, storage, WebSocket, performance, DOM snapshot, …) are checkboxes under *Capture settings* and that I may ask them to tick some.
3. If the app is clearly meant for production (a deployed site, not a prototype or local tool), wrap the install in a dev-only condition (e.g. `localhost`, `?debug=1`, or `import.meta.env.DEV`) and mention that you did so. The overlay exposes request bodies and, when enabled, cookie, storage, DOM and state contents, so it should not ship to end users by accident.
4. If the page already has a CSP, allow the inline script (nonce/hash) or load it as a same-origin file.

If you are modifying an app that already contains the overlay (`window.__aiFeedback` appears in the source), leave it alone, unless the user asks for the newer features: the header comment of this version says `ai-debug-overlay v3`, and older copies lack the source checkboxes, Compare tab, persistence and dragging. To upgrade, replace the whole old script block with the current one (do not merge by hand).

### Injecting it into a site you do not control

The script is self-contained, so it can also be injected without editing the site (useful for debugging a login on someone else's page):

- **Userscript manager (Tampermonkey/Violentmonkey), best option**: wrap the script body (without the `<script>` tags) in a userscript with `// @match` for the site, `// @run-at document-start` and `// @grant none`. It runs first on every load, so nothing is missed and `persist` can follow a login redirect. Use `?aidebug=…` links or the checkboxes to choose sources.
- **DevTools console or a Chrome *Snippet***: paste the script body and run it. It starts late (earlier errors and requests are missed, and code that already grabbed `fetch`/`WebSocket` is not seen) and must be pasted again after each full page load. Firefox asks you to type "allow pasting" first.
- **Bookmarklet**: impractical, the script is too large.
- Sites that enforce **Trusted Types** (some Google properties, for example) can block the overlay's `innerHTML` use, and strict CSPs can block inline injection by some methods. Only the top frame is covered; iframes need their own injection.
- Only do this on sites and accounts you are entitled to test, and treat reports from real logins as sensitive even though values are masked.

## Using a report when the human pastes one

Triage in this order, because earlier problems frequently cause later ones:

1. Read the **Capture on / off** lines near the top so you know which sources were switched on ("none captured" for cookies means nothing if cookies were off). If you need a source that is off, ask for it (see *Asking the human for more information*).
2. Read the **Summary** for the shape of the failure (e.g. one 500 followed by many TypeErrors points at the API, not the UI). Check **Page loads** to see whether the problem spans a redirect or reload.
3. Find the **earliest** entry in **Problems** (entries are numbered and timestamped as seconds since page load, prefixed `p1`, `p2`… when the log spans several page loads). Later errors are often downstream of it.
4. For failed requests, compare the request body and URL against what the server or API expects, and read the response body; it usually states the cause.
5. For exceptions, use the stack trace to locate the file/line, then read the code before changing it.
6. Check the **Network log** for requests that "succeeded" but returned unexpected shapes, and the **Console** section for context logged just before the failure.
7. For "I'm logged out", "my data disappeared" or "settings don't save", read **Cookies**, **localStorage/sessionStorage**, **IndexedDB** and **Storage changes**: look for a missing or expired cookie, duplicate cookie names, `no-op` cookie writes (rejected attributes), a key that was removed or overwritten (the stack says by whom), values that are not valid JSON, `QuotaExceededError`, or changes logged `across page load`.
8. For "did sign-in actually work?", read **Changes since baseline**: the URL and title change, the password field and login controls disappearing, a logout control appearing, an auth-looking cookie or storage key being added, and the requests since the baseline (`POST /login → 200`, then `GET /me → 200` versus `401`). No single signal is proof; look for several agreeing. If the report shows the sign-in succeeded but the app still behaves as signed out, look at what the app checks (cookie name, token key, `/me` response shape).
9. For slowness, read **Performance** (long tasks, LCP element, slow resources); for realtime issues, the **WebSocket** log (close codes, last message before the drop); for "stale" behavior, **Service workers & caches**.
10. State your diagnosis, make the fix, and ask the human to reproduce and send a fresh report if the problem might persist.

If you have your own browser tool (Playwright, Claude in Chrome, etc.), you do not need the human: load the page, then run in the page context:

```js
__aiFeedback.setCapture('localStorage', true);   // ids: cookies, localStorage, sessionStorage, indexedDB, cache, appState, ws, postMessage, input, nav, mutations, perf, env, a11y, dom, persist, redact
__aiFeedback.mark();                              // baseline (turns persist on, survives navigations)
// ...sign in / exercise the app...
await __aiFeedback.refresh();                     // re-read IndexedDB, caches, cookie attributes, env
__aiFeedback.report();                            // markdown, same as Copy report
__aiFeedback.compare();                           // structured before/after
__aiFeedback.json();                              // everything, structured
```

`__aiFeedback.storage()` returns the current cookie/storage state, `sources()` the sources that are on, `watch(name, getter)` registers app state, and `observe(selector)` watches DOM changes. Sources enabled after load only record from that moment, so for history from page load pass the flags in the URL (`?aidebug=…`) or set `CONFIG.capture`.

## The script

```html
<script>
/* ai-debug-overlay v3: captures console/runtime errors + network traffic, plus optional sources (cookies, storage, IndexedDB, caches, WebSocket, input trail, perf, env, a11y, DOM, app state), log persistence and baseline compare; Alt+Shift+D toggles a draggable overlay. */
(() => {
  if (window.__aiFeedback) return;

  const CONFIG = {
    hotkey: { code: 'KeyD', alt: true, shift: true, ctrl: false, meta: false },
    maxEntries: 500,        // ring buffer; non-problem entries are evicted first
    maxBodyChars: 4000,     // per request/response body in the full detail view
    slowMs: 2000,           // requests slower than this are flagged in the summary
    redactHeaders: ['authorization', 'cookie', 'set-cookie', 'x-api-key', 'proxy-authorization'],
    redactKeys: /pass(word)?|secret|token|api[_-]?key|authorization|session/i,
    // Browser-state capture. These are the defaults; the human can flip them live under
    // "Capture settings" in the overlay (the choice is remembered in localStorage.__aiDebugPrefs).
    capture: { // which sources start on; everything except cookies is off until someone asks for it
      cookies: true, localStorage: false, sessionStorage: false, indexedDB: false, cache: false, appState: false,
      ws: false, postMessage: false, input: false, nav: false, mutations: false,
      perf: false, env: false, a11y: false, dom: false, persist: false,
    },
    redactStorageKeys: /pass(word)?|secret|token|api[_-]?key|authoriz|auth(?!or)|sess(ion)?|(^|[_-])sid$|jwt|csrf|xsrf|credential|bearer|otp/i,
    maxStorageValueChars: 600, // per cookie/storage value in the copied or downloaded report
    idbSampleRecords: 5,       // records sampled per IndexedDB object store
    pollMs: 1500,              // how often cookies / Web Storage are diffed for changes made outside the patched APIs
    caps: { realtime: 200, activity: 200, storage: 200 }, // max kept entries per noisy type, so they cannot evict errors and requests
    slowResourceMs: 1000,      // resources slower than this appear under Perf
    maxDomChars: 20000,        // DOM snapshot size in the view and report
    maxPersistEntries: 300,    // entries kept across page loads when 'persist' is on
  };

  /* What can be switched on. [group, [[id, label, description], ...]] */
  const SOURCES = [
    ['State', [
      ['cookies', 'Cookies', 'JS-visible cookies, attributes, change log'],
      ['localStorage', 'localStorage', 'keys and values, change log'],
      ['sessionStorage', 'sessionStorage', 'keys and values, change log'],
      ['indexedDB', 'IndexedDB', 'databases, stores, sample records, write log'],
      ['cache', 'Service workers & Cache API', 'registrations, cache names and cached URLs'],
      ['appState', 'App state', 'values the app registered with __aiFeedback.watch()'],
    ]],
    ['Traffic', [
      ['ws', 'WebSocket / EventSource', 'connect, open, send, message, close, error'],
      ['postMessage', 'postMessage (incoming)', 'messages received from other windows/iframes'],
    ]],
    ['Activity', [
      ['input', 'Click / input trail', 'clicks, committed form input (masked), submits, special keys'],
      ['nav', 'Page events', 'URL changes, tab visibility, online/offline'],
      ['mutations', 'DOM mutations', 'changes under selectors passed to __aiFeedback.observe()'],
    ]],
    ['Performance', [
      ['perf', 'Performance', 'web vitals, long tasks, slow resources, JS heap'],
    ]],
    ['Session', [
      ['persist', 'Keep log across page loads', 'survives redirects and reloads in this tab (stored in sessionStorage); needed to follow a login redirect'],
    ]],
    ['Page', [
      ['env', 'Environment & permissions', 'device, locale, network, storage quota, permission states'],
      ['a11y', 'Accessibility & DOM checks', 'missing alt/labels/names, duplicate ids, lang/title'],
      ['dom', 'DOM snapshot', 'current HTML with scripts, secrets and data: URIs stripped'],
    ]],
  ];
  const SOURCE_IDS = SOURCES.flatMap((g) => g[1].map((x) => x[0]));
  const SOURCE_LABEL = Object.fromEntries(SOURCES.flatMap((g) => g[1].map((x) => [x[0], x[1]])));

  const FENCE = '`'.repeat(3);
  const t0 = performance.now();
  const entries = [];
  let seq = 0;
  let cookieMeta = [], idbSnap = null, lastCookies = null, swSnap = null, domSnap = null, a11ySnap = null, appSnap = null;
  const envAsync = { quota: null, perms: {} };
  let pageNo = 1, BASE = null, saveT;
  const pages = []; // one record per page load in this tab (kept across loads when 'persist' is on)
  const lastArea = { localStorage: null, sessionStorage: null };

  /* ---------- prefs (capture toggles + panel position) ---------- */
  const PREF_KEY = '__aiDebugPrefs', LOG_KEY = '__aiDebugLog';
  const isOwnKey = (k) => k === PREF_KEY || k === LOG_KEY; // the overlay's own storage keys are never reported
  const prefs = { ...CONFIG.capture, redact: true, pos: null };
  try {
    const saved = JSON.parse(localStorage.getItem(PREF_KEY) || '{}');
    for (const k of SOURCE_IDS) if (typeof saved[k] === 'boolean') prefs[k] = saved[k];
    if (saved.pos && isFinite(saved.pos.x) && isFinite(saved.pos.y)) prefs.pos = saved.pos;
  } catch {}
  let autoOpen = false;
  try { // ?aidebug=ws,perf,open  (or aidebug=all) switches sources on from page load; "open" also shows the overlay
    const tokens = String(new URLSearchParams(location.search).get('aidebug') || '').split(',').map((x) => x.trim()).filter(Boolean);
    (tokens.includes('all') ? SOURCE_IDS : tokens).forEach((k) => { if (SOURCE_IDS.includes(k)) prefs[k] = true; });
    autoOpen = tokens.includes('open');
  } catch {}
  const savePrefs = () => {
    try { const { redact, ...rest } = prefs; localStorage.setItem(PREF_KEY, JSON.stringify(rest)); } catch {}
  };

  /* ---------- helpers ---------- */
  const trunc = (s, n = CONFIG.maxBodyChars) => {
    s = String(s);
    return s.length > n ? s.slice(0, n) + `… [truncated ${s.length - n} chars]` : s;
  };
  const safeStringify = (v) => {
    const seen = new WeakSet();
    try {
      return JSON.stringify(v, (k, val) => {
        if (typeof val === 'bigint') return val.toString();
        if (typeof val === 'function') return `[Function ${val.name || 'anonymous'}]`;
        if (val instanceof Error) return { name: val.name, message: val.message, stack: val.stack };
        if (typeof Blob !== 'undefined' && val instanceof Blob) return `[Blob ${val.type || 'unknown'} ${val.size}B]`;
        if (val instanceof ArrayBuffer || ArrayBuffer.isView(val)) return `[binary ${val.byteLength}B]`;
        if (val instanceof Map) return { '[Map]': [...val] };
        if (val instanceof Set) return { '[Set]': [...val] };
        if (val && typeof val === 'object') {
          if (seen.has(val)) return '[Circular]';
          seen.add(val);
        }
        return val;
      });
    } catch { return String(v); }
  };
  const fmtArg = (a) =>
    typeof a === 'string' ? a :
    a instanceof Error ? (a.stack || `${a.name}: ${a.message}`) :
    safeStringify(a);
  const esc = (s) => String(s).replace(/[&<>"]/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;' }[c]));
  const since = () => ((performance.now() - t0) / 1000).toFixed(1);
  const at = (e) => (pageNo > 1 ? `p${e.pg || 1} ` : '') + `+${e.t}s`; // timestamp label; p2 = second page load
  const callerStack = () => (new Error().stack || '').split('\n').slice(3, 9).join('\n'); // call from inside a wrapper

  const redactUrl = (u) => {
    try {
      const x = new URL(u, location.href);
      for (const k of [...x.searchParams.keys()]) if (CONFIG.redactKeys.test(k)) x.searchParams.set(k, 'REDACTED');
      return x.href;
    } catch { return String(u); }
  };
  const redactHeaders = (h) => {
    const out = {};
    try {
      for (const [k, v] of new Headers(h || {}).entries()) out[k] = CONFIG.redactHeaders.includes(k.toLowerCase()) ? '[REDACTED]' : v;
    } catch {}
    return out;
  };
  const redactBody = (text, re = CONFIG.redactKeys) => {
    try {
      const j = JSON.parse(text);
      const walk = (o) => {
        if (o && typeof o === 'object') for (const k in o) {
          if (re.test(k) && (o[k] === null || typeof o[k] !== 'object')) o[k] = '[REDACTED]';
          else walk(o[k]);
        }
      };
      walk(j);
      return JSON.stringify(j);
    } catch { return text; }
  };
  const redactForm = (text, re = CONFIG.redactKeys) => { // key=value&key2=value2 bodies (login forms, urlencoded POSTs)
    try {
      const q = new URLSearchParams(text);
      for (const k of [...new Set(q.keys())]) if (re.test(k)) q.set(k, 'REDACTED');
      return q.toString();
    } catch { return text; }
  };
  const looksForm = (x) => /^[^\s=&]+=[^\s&]*(&[^\s=&]+=[^\s&]*)*$/.test(x);
  const redactAny = (text, re) => { // JSON or urlencoded; anything else is returned as is
    const t = String(text), x = t.trim(), r = re || CONFIG.redactKeys;
    if (/^[\[{]/.test(x)) return redactBody(t, re);
    if (looksForm(x)) return redactForm(x, re);
    if (t.length > 200000) return t;
    // last resort for text that merely contains JSON-ish pairs (socket.io frames, log lines): "token":"abc" -> "token":"[REDACTED]"
    return t.replace(new RegExp('("?[\\w-]*(?:' + r.source + ')[\\w-]*"?\\s*[:=]\\s*)"[^"]*"', 'gi'), (m, pre) => pre + '"[REDACTED]"');
  };
  const JWT_RE = /^eyJ[\w-]+\.[\w-]+\.[\w-]*$/;
  const mask = (key, val) => { // redaction for cookie / storage values
    if (val == null || !prefs.redact) return val;
    const s = String(val);
    if (CONFIG.redactStorageKeys.test(String(key))) return '[REDACTED]';
    if (JWT_RE.test(s.trim())) return '[REDACTED JWT]';
    return /^\s*[\[{]/.test(s) ? redactBody(s, CONFIG.redactStorageKeys) : s;
  };
  const describeBody = (b) => {
    if (b == null) return undefined;
    if (typeof b === 'string') return trunc(redactAny(b));
    if (b instanceof URLSearchParams) return trunc(redactForm(b.toString()));
    if (typeof FormData !== 'undefined' && b instanceof FormData) return `[FormData: ${[...b.keys()].join(', ')}]`;
    if (typeof Blob !== 'undefined' && b instanceof Blob) return `[Blob ${b.type || 'unknown'} ${b.size}B]`;
    if (b instanceof ArrayBuffer || ArrayBuffer.isView(b)) return `[binary ${b.byteLength}B]`;
    return '[unreadable body]';
  };
  const TEXTY = /json|text|xml|javascript|html|x-www-form/i;

  const isFailed = (e) => e.type === 'network' && !e.pending && (e.error || !e.status || e.status >= 400);
  const isProblem = (e) => e.level === 'error' || isFailed(e);

  function push(e) {
    const last = entries[entries.length - 1];
    if (last && e.type !== 'network' && last.type === e.type && last.message === e.message) {
      last.count = (last.count || 1) + 1; // collapse consecutive duplicates (render loops etc.)
      if (prefs.persist) schedSave();
      render();
      return last;
    }
    e.id = ++seq;
    e.pg = pageNo;
    e.ts = new Date().toISOString();
    e.t = since();
    entries.push(e);
    const cap = CONFIG.caps[e.type];
    if (cap) {
      let n = 0, first = -1;
      for (let i = 0; i < entries.length; i++) if (entries[i].type === e.type) { if (first < 0) first = i; n++; }
      if (n > cap) entries.splice(first, 1);
    }
    if (entries.length > CONFIG.maxEntries) {
      const i = entries.findIndex((x) => !isProblem(x));
      entries.splice(i === -1 ? 0 : i, 1);
    }
    if (prefs.persist) schedSave();
    render();
    return e;
  }
  const pushRT = (o) => push({ type: 'realtime', ...o, message: [o.chan, o.cid, o.dir, o.data, o.url].join('\u0001') });
  const pushAct = (kind, msg, extra) => push({ type: 'activity', kind, msg, ...extra, message: kind + '\u0001' + msg });
  const pushStore = (o) => push({ type: 'storage', ...o, message: o.message || [o.area, o.op, o.key, o.via, o.oldValue, o.newValue, o.raw].join('\u0001') });

  /* ---------- console ---------- */
  ['error', 'warn', 'info', 'log', 'debug'].forEach((level) => {
    const orig = console[level];
    console[level] = function (...args) {
      try {
        push({
          type: 'console', level,
          message: trunc(args.map(fmtArg).join(' '), 2000),
          stack: level === 'error' || level === 'warn' ? (new Error().stack || '').split('\n').slice(2, 8).join('\n') : undefined,
        });
      } catch {}
      return orig.apply(this, args);
    };
  });

  /* ---------- runtime errors ---------- */
  addEventListener('error', (e) => {
    const el = e.target;
    if (el && el !== window && (el.src || el.href)) {
      push({ type: 'resource', level: 'error', message: `Failed to load <${el.tagName.toLowerCase()}> ${el.src || el.href}` });
    } else {
      push({ type: 'exception', level: 'error', message: e.message, source: `${e.filename}:${e.lineno}:${e.colno}`, stack: e.error && e.error.stack });
    }
  }, true);
  addEventListener('unhandledrejection', (e) =>
    push({ type: 'unhandledrejection', level: 'error', message: fmtArg(e.reason), stack: e.reason && e.reason.stack }));
  addEventListener('securitypolicyviolation', (e) =>
    push({ type: 'csp', level: 'error', message: `CSP blocked ${e.violatedDirective}: ${e.blockedURI}` }));

  /* ---------- fetch ---------- */
  async function readBody(res, entry) {
    try {
      const ct = res.headers.get('content-type') || '';
      if (/event-stream/i.test(ct)) entry.responseBody = '[streaming response not captured]';
      else if (!TEXTY.test(ct)) entry.responseBody = `[${ct || 'unknown type'} not captured]`;
      else entry.responseBody = trunc(redactAny(await res.text()));
    } catch { entry.responseBody = '[could not read body]'; }
    render();
  }
  const origFetch = window.fetch;
  if (origFetch) {
    window.fetch = async function (input, init) {
      const start = performance.now();
      const isReq = typeof Request !== 'undefined' && input instanceof Request;
      const entry = push({
        type: 'network', via: 'fetch', pending: true,
        method: String((init && init.method) || (isReq && input.method) || 'GET').toUpperCase(),
        url: redactUrl(isReq ? input.url : String(input)),
        requestHeaders: redactHeaders((init && init.headers) || (isReq ? input.headers : {})),
        requestBody: describeBody(init && init.body),
      });
      try {
        const res = await origFetch.apply(this, arguments);
        Object.assign(entry, {
          pending: false, status: res.status, statusText: res.statusText,
          durationMs: Math.round(performance.now() - start), responseHeaders: redactHeaders(res.headers),
        });
        readBody(res.clone(), entry);
        return res;
      } catch (err) {
        Object.assign(entry, { pending: false, error: String(err), durationMs: Math.round(performance.now() - start) });
        render();
        throw err;
      }
    };
  }

  /* ---------- XMLHttpRequest ---------- */
  const XP = XMLHttpRequest.prototype, xOpen = XP.open, xSend = XP.send, xSet = XP.setRequestHeader;
  XP.open = function (method, url) {
    this.__ai = { type: 'network', via: 'xhr', method: String(method).toUpperCase(), url: redactUrl(url), requestHeaders: {} };
    return xOpen.apply(this, arguments);
  };
  XP.setRequestHeader = function (k, v) {
    if (this.__ai) this.__ai.requestHeaders[k] = CONFIG.redactHeaders.includes(String(k).toLowerCase()) ? '[REDACTED]' : v;
    return xSet.apply(this, arguments);
  };
  XP.send = function (body) {
    if (this.__ai) {
      const xhr = this, start = performance.now();
      const entry = push({ ...this.__ai, pending: true, requestBody: describeBody(body) });
      const done = (err) => {
        Object.assign(entry, { pending: false, status: xhr.status, statusText: xhr.statusText, durationMs: Math.round(performance.now() - start) });
        if (err) entry.error = err;
        try {
          const ct = xhr.getResponseHeader('content-type') || '';
          entry.responseBody = (xhr.responseType === '' || xhr.responseType === 'text')
            ? (TEXTY.test(ct) ? trunc(redactAny(xhr.responseText)) : `[${ct || 'unknown type'} not captured]`)
            : `[responseType ${xhr.responseType}]`;
        } catch {}
        render();
      };
      this.addEventListener('load', () => done());
      this.addEventListener('error', () => done('Network error'));
      this.addEventListener('timeout', () => done('Timeout'));
      this.addEventListener('abort', () => done('Aborted'));
    }
    return xSend.apply(this, arguments);
  };

  /* ---------- cookies ---------- */
  const safeDecode = (s) => { try { return decodeURIComponent(s); } catch { return s; } };
  const cookieError = () => { try { void document.cookie; return ''; } catch (e) { return String((e && e.message) || e); } };
  const readCookies = () => {
    const out = [];
    try {
      for (const part of document.cookie.split(/;\s*/)) {
        if (!part) continue;
        const i = part.indexOf('=');
        out.push({ name: i < 0 ? '' : part.slice(0, i), value: i < 0 ? part : part.slice(i + 1) });
      }
    } catch {}
    return out;
  };
  const cookieKeyed = () => { // Map of name (or name#2 for duplicates) -> raw value
    const seen = new Map(), m = new Map();
    for (const c of readCookies()) {
      const n = (seen.get(c.name) || 0) + 1;
      seen.set(c.name, n);
      m.set(n > 1 ? `${c.name}#${n}` : c.name, c.value);
    }
    return m;
  };
  const cookiesNow = () => {
    const pool = cookieMeta.slice(), names = new Map(), list = readCookies();
    list.forEach((c) => names.set(c.name, (names.get(c.name) || 0) + 1));
    return list.map((c) => {
      const i = pool.findIndex((m) => m.name === c.name);
      const m = i >= 0 ? pool.splice(i, 1)[0] : null;
      return {
        name: c.name, value: mask(c.name, c.value), size: c.name.length + c.value.length, dup: names.get(c.name) > 1,
        ...(m ? { domain: m.domain || '', path: m.path || '/', expires: m.expires, secure: !!m.secure, sameSite: m.sameSite } : {}),
      };
    });
  };
  function diffMaps(prev, now) {
    const out = [];
    if (!prev) return out;
    for (const [k, v] of now) {
      if (!prev.has(k)) out.push({ op: 'add', key: k, newValue: mask(k, v) });
      else if (prev.get(k) !== v) out.push({ op: 'change', key: k, oldValue: mask(k, prev.get(k)), newValue: mask(k, v) });
    }
    for (const [k, v] of prev) if (!now.has(k)) out.push({ op: 'remove', key: k, oldValue: mask(k, v) });
    return out;
  }
  function diffCookies() { const now = cookieKeyed(), out = diffMaps(lastCookies, now); lastCookies = now; return out; }
  const maskAssign = (raw) => {
    const parts = String(raw).split(';'), i = parts[0].indexOf('=');
    const n = i < 0 ? '' : parts[0].slice(0, i).trim(), v = i < 0 ? parts[0] : parts[0].slice(i + 1);
    parts[0] = `${n}=${mask(n, v)}`;
    return parts.join(';').trim();
  };
  function onCookieWrite(raw, stack) {
    const ch = diffCookies(), via = 'document.cookie', assign = maskAssign(raw);
    if (ch.length) ch.forEach((c) => pushStore({ area: 'cookie', via, raw: assign, stack, ...c }));
    else pushStore({
      area: 'cookie', op: 'no-op', key: assign.split('=')[0].trim(), via, raw: assign, stack,
      note: 'Write changed nothing visible: identical value, or the browser rejected it (e.g. Secure on http, wrong domain/path, already-expired date, HttpOnly cookie)',
    });
  }
  try {
    const d = Object.getOwnPropertyDescriptor(Document.prototype, 'cookie');
    if (d && d.configurable && d.set) {
      Object.defineProperty(Document.prototype, 'cookie', {
        configurable: true, enumerable: d.enumerable, get: d.get,
        set(v) {
          const stack = prefs.cookies ? callerStack() : undefined;
          d.set.call(this, v);
          if (prefs.cookies) { try { onCookieWrite(v, stack); } catch {} }
        },
      });
    }
  } catch {}

  /* ---------- localStorage / sessionStorage ---------- */
  const area = (n) => { try { return window[n]; } catch { return null; } };
  const readArea = (n) => {
    const m = new Map(), s = area(n);
    if (s) try { for (let i = 0; i < s.length; i++) { const k = s.key(i); if (!isOwnKey(k)) m.set(k, s.getItem(k)); } } catch {}
    return m;
  };
  const areaNow = (n) => [...readArea(n)].map(([key, value]) => ({ key, size: String(value).length, value: mask(key, value) }));
  function diffArea(n) { const now = readArea(n), out = diffMaps(lastArea[n], now); lastArea[n] = now; return out; }
  try {
    const SP = Storage.prototype;
    ['setItem', 'removeItem', 'clear'].forEach((m) => {
      const orig = SP[m];
      SP[m] = function (...a) {
        const n = this === area('localStorage') ? 'localStorage' : this === area('sessionStorage') ? 'sessionStorage' : null;
        const live = !!(n && prefs[n] && !isOwnKey(a[0]));
        const stack = live ? callerStack() : undefined;
        let r;
        try { r = orig.apply(this, a); }
        catch (err) {
          if (live) try { pushStore({ area: n, op: 'error', level: 'error', key: String(a[0]), via: `${n}.${m}`, message: `${n}.${m}(${a[0]}) threw ${err.name}: ${err.message}`, stack }); } catch {}
          throw err;
        }
        if (live) try {
          const ch = diffArea(n), via = `${n}.${m}`;
          if (m === 'clear') pushStore({ area: n, op: 'clear', key: '*', via, oldValue: `${ch.length} key(s) removed`, stack });
          else if (ch.length) ch.forEach((c) => pushStore({ area: n, via, stack, ...c }));
          else if (m === 'setItem') pushStore({ area: n, op: 'no-op', key: String(a[0]), via, note: 'Same value was already stored', stack });
        } catch {}
        return r;
      };
    });
  } catch {}

  /* ---------- IndexedDB ---------- */
  const idbName = (os) => { try { return `${os.transaction.db.name}/${os.name}`; } catch { return String(os && os.name); } };
  try {
    const OP = IDBObjectStore.prototype;
    ['put', 'add', 'delete', 'clear'].forEach((m) => {
      const orig = OP[m];
      if (!orig) return;
      OP[m] = function (...a) {
        const live = prefs.indexedDB;
        const stack = live ? callerStack() : undefined;
        let r;
        try { r = orig.apply(this, a); }
        catch (err) {
          if (live) try { pushStore({ area: 'indexedDB', op: 'error', level: 'error', key: idbName(this), via: `IDBObjectStore.${m}`, message: `${idbName(this)}.${m}() threw ${err.name}: ${err.message}`, stack }); } catch {}
          throw err;
        }
        if (live) try {
          const kp = this.keyPath;
          const rec = m === 'clear' ? undefined
            : m === 'delete' ? safeStringify(a[0] instanceof IDBKeyRange ? '[range]' : a[0])
            : safeStringify(a[1] !== undefined ? a[1] : (typeof kp === 'string' && a[0] && typeof a[0] === 'object' ? a[0][kp] : undefined));
          pushStore({
            area: 'indexedDB', op: m, key: idbName(this) + (rec !== undefined ? ` [${rec}]` : ''), via: `IDBObjectStore.${m}`, stack,
            newValue: m === 'put' || m === 'add' ? trunc(mask(this.name, safeStringify(a[0])), 500) : undefined,
          });
        } catch {}
        return r;
      };
    });
  } catch {}
  const idbReq = (r) => new Promise((res, rej) => { r.onsuccess = () => res(r.result); r.onerror = () => rej(r.error); });
  const sampleStore = (os, limit) => new Promise((res) => {
    const out = []; let r;
    try { r = os.openCursor(); } catch (e) { return res(out); }
    r.onsuccess = () => {
      const c = r.result;
      if (c && out.length < limit) { out.push({ key: safeStringify(c.key), valueText: mask(os.name, safeStringify(c.value)) }); c.continue(); }
      else res(out);
    };
    r.onerror = () => res(out);
  });
  async function snapIdb() {
    const out = { takenAt: new Date().toISOString(), t: since(), dbs: [] };
    if (!window.indexedDB) { out.note = 'IndexedDB is not available in this context.'; return out; }
    if (!indexedDB.databases) { out.note = 'This browser cannot list IndexedDB databases (indexedDB.databases is missing); writes are still logged.'; return out; }
    let list = [];
    try { list = await indexedDB.databases(); } catch (e) { out.note = String(e); return out; }
    for (const info of list) {
      const d = { name: info.name, version: info.version, stores: [] };
      out.dbs.push(d);
      let db;
      try {
        db = await new Promise((res, rej) => {
          const r = indexedDB.open(info.name);
          r.onsuccess = () => res(r.result);
          r.onerror = () => rej(r.error || new Error('open failed'));
          r.onblocked = () => rej(new Error('open blocked'));
          r.onupgradeneeded = () => { try { r.transaction.abort(); } catch {} }; // never create or upgrade anything
        });
        for (const n of [...db.objectStoreNames]) {
          const os = db.transaction(n, 'readonly').objectStore(n);
          const count = await idbReq(os.count());
          const sample = await sampleStore(os, CONFIG.idbSampleRecords);
          d.stores.push({ name: n, keyPath: os.keyPath, autoIncrement: os.autoIncrement, indexes: [...os.indexNames], count, sample });
        }
      } catch (e) { d.error = String((e && e.message) || e); }
      finally { try { db && db.close(); } catch {} }
    }
    return out;
  }

  /* ---------- realtime: WebSocket / EventSource ---------- */
  let rtSeq = 0;
  const rtData = (d) => (typeof d === 'string' ? trunc(redactAny(d), 500) : String(describeBody(d)));
  try {
    if (window.WebSocket) {
      const O = window.WebSocket;
      const W = class extends O {
        constructor(...a) {
          super(...a);
          const cid = ++rtSeq, url = redactUrl(String(a[0]));
          this.__aiId = cid;
          const log = (o) => { if (prefs.ws) try { pushRT({ chan: 'ws', cid, url, ...o }); } catch {} };
          log({ dir: 'connect' });
          this.addEventListener('open', () => log({ dir: 'open' }));
          this.addEventListener('message', (e) => log({ dir: 'in', data: rtData(e.data) }));
          this.addEventListener('error', () => log({ dir: 'error', level: 'error', data: 'WebSocket error (the browser gives no detail; check Network and Console)' }));
          this.addEventListener('close', (e) => log({ dir: 'close', data: `code ${e.code}${e.reason ? ' ' + e.reason : ''}${e.wasClean ? '' : ' (unclean)'}`, ...(e.wasClean ? {} : { level: 'error' }) }));
        }
        send(d) {
          if (prefs.ws) try { pushRT({ chan: 'ws', cid: this.__aiId, dir: 'out', data: rtData(d) }); } catch {}
          return super.send(d);
        }
      };
      try { Object.defineProperty(W, 'name', { value: 'WebSocket' }); } catch {}
      window.WebSocket = W;
    }
    if (window.EventSource) {
      const O = window.EventSource;
      const E = class extends O {
        constructor(...a) {
          super(...a);
          const cid = ++rtSeq, url = redactUrl(String(a[0]));
          const log = (o) => { if (prefs.ws) try { pushRT({ chan: 'sse', cid, url, ...o }); } catch {} };
          log({ dir: 'connect' });
          this.addEventListener('open', () => log({ dir: 'open' }));
          this.addEventListener('message', (e) => log({ dir: 'in', data: rtData(e.data) }));
          this.addEventListener('error', () => log(this.readyState === 2 ? { dir: 'error', level: 'error', data: 'connection closed' } : { dir: 'error', data: 'connection lost, the browser will retry' }));
        }
      };
      try { Object.defineProperty(E, 'name', { value: 'EventSource' }); } catch {}
      window.EventSource = E;
    }
  } catch {}

  /* ---------- activity: clicks/inputs, page events, postMessage, DOM mutations ---------- */
  const txtOf = (el) => String(el.innerText != null ? el.innerText : el.textContent || ''); // innerText skips hidden text; textContent is the fallback
  const selOf = (el) => {
    try {
      if (!el || !el.tagName) return el === document ? 'document' : String(el);
      let s = el.tagName.toLowerCase();
      if (el.id) s += '#' + el.id;
      else if (typeof el.className === 'string' && el.className.trim()) s += '.' + el.className.trim().split(/\s+/).slice(0, 2).join('.');
      const nm = el.getAttribute && (el.getAttribute('name') || el.getAttribute('aria-label') || el.getAttribute('data-testid'));
      return nm ? `${s}[${nm}]` : s;
    } catch { return '?'; }
  };
  const inOverlay = (e) => !!(host && (e.target === host || (e.composedPath && e.composedPath().includes(host))));
  addEventListener('click', (e) => {
    if (!prefs.input || inOverlay(e)) return;
    const t = e.target;
    const txt = /^(INPUT|TEXTAREA|SELECT)$/.test(t.tagName) || (t.children && t.children.length > 3) ? '' : txtOf(t).replace(/\s+/g, ' ').trim().slice(0, 40);
    pushAct('click', `${selOf(t)}${txt ? ` "${txt}"` : ''}`);
  }, true);
  addEventListener('change', (e) => { // committed values only (not every keystroke); passwords are never recorded
    if (!prefs.input || inOverlay(e)) return;
    const t = e.target;
    let v;
    if (t.type === 'password') v = '[hidden]';
    else if (t.type === 'checkbox' || t.type === 'radio') v = String(t.checked);
    else if (t.type === 'file') v = `${t.files ? t.files.length : 0} file(s)`;
    else v = trunc(String(mask(t.name || t.id || t.type, t.value)), 80);
    pushAct('input', `${selOf(t)} = ${v}`);
  }, true);
  addEventListener('submit', (e) => { // where the form goes and which fields it sends (names only, never values)
    if (!prefs.input || inOverlay(e)) return;
    const f = e.target;
    try {
      const names = [...(f.elements || [])].map((x) => x.name).filter(Boolean).join(', ');
      pushAct('submit', `${selOf(f)} ${String(f.method || 'get').toUpperCase()} ${shortUrl(redactUrl(f.action || location.href))} fields: ${names || '(none)'}`);
    } catch { pushAct('submit', selOf(f)); }
  }, true);
  addEventListener('keydown', (e) => { // special keys and shortcuts only, never typed characters
    if (!prefs.input || inOverlay(e) || e.code === CONFIG.hotkey.code && e.altKey) return;
    if (/^(Enter|Escape|Tab|Arrow\w+|F\d+|Backspace|Delete|Home|End|Page\w+)$/.test(e.key) || ((e.ctrlKey || e.metaKey) && e.key.length === 1)) {
      pushAct('key', `${e.ctrlKey ? 'Ctrl+' : ''}${e.metaKey ? 'Meta+' : ''}${e.altKey ? 'Alt+' : ''}${e.shiftKey ? 'Shift+' : ''}${e.key} on ${selOf(e.target)}`);
    }
  }, true);
  const navLog = (m) => { if (prefs.nav) try { pushAct('page', m); } catch {} };
  document.addEventListener('visibilitychange', () => navLog(`visibility → ${document.visibilityState}`));
  addEventListener('online', () => navLog('browser went online'));
  addEventListener('offline', () => navLog('browser went offline'));
  addEventListener('popstate', () => navLog(`popstate → ${redactUrl(location.href)}`));
  addEventListener('hashchange', () => navLog(`hashchange → ${redactUrl(location.href)}`));
  addEventListener('pagehide', () => navLog('pagehide (leaving or caching the page)'));
  try {
    ['pushState', 'replaceState'].forEach((m) => {
      const o = History.prototype[m];
      History.prototype[m] = function (...a) { const r = o.apply(this, a); navLog(`history.${m} → ${redactUrl(location.href)}`); return r; };
    });
  } catch {}
  addEventListener('message', (e) => {
    if (!prefs.postMessage) return;
    let d;
    try { d = typeof e.data === 'string' ? e.data : safeStringify(e.data); } catch { d = '[unserializable]'; }
    pushAct('message', `from ${e.origin || '(none)'}${e.source === window ? ' (self)' : ''}: ${trunc(redactAny(String(d)), 300)}`);
  }, true);
  const watchers = [];
  function observe(selector) { // watch elements matching selector (as they exist now); returns how many
    let els = [];
    try { els = [...document.querySelectorAll(selector)]; } catch { return 0; }
    els.forEach((el) => {
      new MutationObserver((list) => {
        if (!prefs.mutations) return;
        list.slice(0, 20).forEach((m) => {
          const what = m.type === 'attributes'
            ? `attr ${m.attributeName}: ${trunc(mask(m.attributeName, m.oldValue), 40)} → ${trunc(mask(m.attributeName, m.target.getAttribute(m.attributeName)), 40)}`
            : m.type === 'childList' ? `children +${m.addedNodes.length} −${m.removedNodes.length}` : 'text changed';
          pushAct('mutation', `${selOf(m.target)} ${what}`);
        });
      }).observe(el, { subtree: true, childList: true, attributes: true, attributeOldValue: true, characterData: true });
    });
    watchers.push({ selector, elements: els.length });
    return els.length;
  }

  /* ---------- performance ---------- */
  const PERF = { fcp: null, lcp: null, cls: 0, fid: null, longTasks: [], slow: [], mem: [] };
  let perfStarted = false;
  function startPerf() {
    if (perfStarted || !window.PerformanceObserver) return;
    perfStarted = true;
    const obs = (type, cb) => {
      try {
        new PerformanceObserver((l) => { try { l.getEntries().forEach(cb); } catch {} render(); }).observe({ type, buffered: true });
      } catch {}
    };
    obs('paint', (e) => { if (e.name === 'first-contentful-paint') PERF.fcp = Math.round(e.startTime); });
    obs('largest-contentful-paint', (e) => { PERF.lcp = { ms: Math.round(e.startTime), el: e.element ? selOf(e.element) : '', url: e.url || '' }; });
    obs('layout-shift', (e) => { if (!e.hadRecentInput) PERF.cls += e.value; });
    obs('first-input', (e) => { PERF.fid = Math.round(e.processingStart - e.startTime); });
    obs('longtask', (e) => { PERF.longTasks.push({ at: Math.round(e.startTime), ms: Math.round(e.duration) }); if (PERF.longTasks.length > 100) PERF.longTasks.shift(); });
    obs('resource', (e) => {
      if (e.duration > CONFIG.slowResourceMs) {
        PERF.slow.push({ url: redactUrl(e.name), ms: Math.round(e.duration), type: e.initiatorType, kb: Math.round((e.transferSize || 0) / 1024) });
        if (PERF.slow.length > 50) PERF.slow.shift();
      }
    });
  }
  const perfNow = () => {
    let nav = null;
    try {
      const n = performance.getEntriesByType('navigation')[0];
      if (n) nav = { type: n.type, ttfbMs: Math.round(n.responseStart), domContentLoadedMs: Math.round(n.domContentLoadedEventEnd), loadMs: Math.round(n.loadEventEnd) };
    } catch {}
    const lt = PERF.longTasks, mem = PERF.mem;
    return {
      fcpMs: PERF.fcp, lcp: PERF.lcp, cls: +PERF.cls.toFixed(3), firstInputDelayMs: PERF.fid, nav,
      longTasks: { count: lt.length, totalMs: lt.reduce((a, b) => a + b.ms, 0), worstMs: lt.reduce((a, b) => Math.max(a, b.ms), 0) },
      slowResources: PERF.slow.slice().sort((a, b) => b.ms - a.ms).slice(0, 10),
      heap: mem.length ? { nowMB: mem[mem.length - 1], peakMB: Math.max(...mem), samples: mem.slice() } : null,
    };
  };

  /* ---------- page: environment, service workers/caches, a11y checks, DOM snapshot, app state ---------- */
  function envNow() {
    const n = navigator, c = n.connection || {};
    const mm = (q) => { try { return matchMedia(q).matches; } catch { return undefined; } };
    let tz; try { tz = Intl.DateTimeFormat().resolvedOptions().timeZone; } catch {}
    return {
      page: { url: redactUrl(location.href), referrer: document.referrer ? redactUrl(document.referrer) : '', readyState: document.readyState, visibility: document.visibilityState, charset: document.characterSet, secureContext: window.isSecureContext, crossOriginIsolated: window.crossOriginIsolated },
      device: { pixelRatio: window.devicePixelRatio, screen: window.screen ? `${screen.width}x${screen.height}` : undefined, viewport: `${innerWidth}x${innerHeight}`, touchPoints: n.maxTouchPoints, cpuCores: n.hardwareConcurrency, memoryGB: n.deviceMemory, platform: n.userAgentData ? n.userAgentData.platform : n.platform },
      locale: { language: n.language, languages: (n.languages || []).join(', '), timezone: tz, utcOffsetMin: -new Date().getTimezoneOffset() },
      network: { online: n.onLine, effectiveType: c.effectiveType, downlinkMbps: c.downlink, rttMs: c.rtt, saveData: c.saveData },
      preferences: { darkMode: mm('(prefers-color-scheme: dark)'), reducedMotion: mm('(prefers-reduced-motion: reduce)'), cookiesEnabled: n.cookieEnabled },
      storageQuota: envAsync.quota || undefined,
      permissions: Object.keys(envAsync.perms).length ? envAsync.perms : undefined,
    };
  }
  async function snapCaches() {
    const out = { t: since(), regs: [], caches: [], controlled: !!(navigator.serviceWorker && navigator.serviceWorker.controller) };
    if (navigator.serviceWorker && navigator.serviceWorker.getRegistrations) {
      try {
        for (const r of await navigator.serviceWorker.getRegistrations()) {
          const w = r.active || r.waiting || r.installing;
          out.regs.push({ scope: r.scope, script: w ? w.scriptURL : '', state: w ? w.state : '', waiting: !!r.waiting, installing: !!r.installing });
        }
      } catch (e) { out.note = String(e); }
    } else out.note = 'Service workers are not available in this context.';
    if (window.caches) {
      try {
        for (const k of await caches.keys()) {
          const reqs = await (await caches.open(k)).keys();
          out.caches.push({ name: k, count: reqs.length, urls: reqs.slice(0, 10).map((r) => redactUrl(r.url)) });
        }
      } catch (e) { out.note = (out.note ? out.note + ' ' : '') + String(e); }
    }
    return out;
  }
  function a11yNow() {
    const out = [];
    const all = (q) => { try { return [...document.querySelectorAll(q)].filter((el) => !(host && host.contains(el))); } catch { return []; } };
    const add = (rule, els, hint) => { if (els.length) out.push({ rule, hint, count: els.length, examples: els.slice(0, 5).map(selOf) }); };
    const named = (el) => el.getAttribute('aria-label') || el.getAttribute('aria-labelledby') || el.getAttribute('title') || (el.textContent || '').trim() || el.querySelector('img[alt]:not([alt=""])');
    const labelled = (el) => {
      if (el.getAttribute('aria-label') || el.getAttribute('aria-labelledby') || el.getAttribute('title') || el.closest('label')) return true;
      try { return !!(el.id && document.querySelector(`label[for="${window.CSS && CSS.escape ? CSS.escape(el.id) : el.id}"]`)); } catch { return false; }
    };
    add('img without alt', all('img:not([alt])'), 'Add alt text, or alt="" for decorative images.');
    add('form control without label', all('input:not([type=hidden]):not([type=submit]):not([type=button]):not([type=reset]), select, textarea').filter((el) => !labelled(el)), 'Wrap in <label>, use label[for], or add aria-label.');
    add('button without accessible name', all('button, [role=button]').filter((el) => !named(el) && !el.value), 'Add text, aria-label or title.');
    add('link without accessible name', all('a[href]').filter((el) => !named(el)), 'Add text, aria-label or title.');
    add('iframe without title', all('iframe:not([title])'), 'Add a title attribute.');
    const ids = new Map();
    all('[id]').forEach((el) => ids.set(el.id, (ids.get(el.id) || []).concat(el)));
    add('duplicate id', [...ids.values()].filter((l) => l.length > 1).map((l) => l[1]), 'ids must be unique; getElementById and label[for] pick the first match.');
    const doc = document.documentElement;
    if (!doc.getAttribute('lang')) out.push({ rule: '<html> has no lang attribute', count: 1, examples: [] });
    if (!document.title) out.push({ rule: 'document has no <title>', count: 1, examples: [] });
    if (!document.querySelector('meta[name=viewport]')) out.push({ rule: 'no viewport meta tag (mobile layout will be wrong)', count: 1, examples: [] });
    const h1 = all('h1').length;
    if (h1 !== 1) out.push({ rule: `${h1} <h1> elements (expected 1)`, count: h1, examples: [] });
    return out;
  }
  function domNow() {
    try {
      const c = document.documentElement.cloneNode(true);
      c.querySelectorAll('#__ai-debug-overlay').forEach((n) => n.remove());
      c.querySelectorAll('script').forEach((x) => { const n = (x.textContent || '').length; x.textContent = n ? `[script ${n} chars removed]` : ''; });
      c.querySelectorAll('style').forEach((x) => { x.textContent = trunc(x.textContent || '', 200); });
      c.querySelectorAll('input,textarea').forEach((el) => {
        const n = el.getAttribute('name') || el.id || el.type;
        if (el.type === 'password') el.setAttribute('value', '[hidden]');
        else if (el.hasAttribute('value')) el.setAttribute('value', String(mask(n, el.getAttribute('value'))));
      });
      const nodes = c.querySelectorAll('*').length;
      const html = c.outerHTML.replace(/data:[^"')\s]{80,}/g, 'data:[…]').replace(/\n\s*\n+/g, '\n');
      return { nodes, chars: html.length, html: trunc(html, CONFIG.maxDomChars) };
    } catch (e) { return { nodes: 0, chars: 0, html: `[could not snapshot: ${e}]` }; }
  }
  const watches = new Map();
  function appStateNow() {
    const out = {};
    watches.forEach((fn, name) => {
      try { const v = fn(); out[name] = trunc(mask(name, typeof v === 'string' ? v : safeStringify(v)), CONFIG.maxBodyChars); }
      catch (e) { out[name] = `[getter threw ${e}]`; }
    });
    return out;
  }

  /* ---------- persistence across page loads + baseline comparison ---------- */
  const fnv = (v) => { let h = 2166136261; const t = String(v); for (let i = 0; i < t.length; i++) { h ^= t.charCodeAt(i); h = Math.imul(h, 16777619); } return (h >>> 0).toString(36); };
  function stateNow() { // name -> [hash of the real value, masked value], for the sources that are on. Real values are never stored.
    const m = (map) => { const o = {}; for (const [k, v] of map) o[k] = [fnv(v), trunc(String(mask(k, v)), 200)]; return o; };
    const st = {};
    if (prefs.cookies) st.cookies = m(cookieKeyed());
    if (prefs.localStorage) st.localStorage = m(readArea('localStorage'));
    if (prefs.sessionStorage) st.sessionStorage = m(readArea('sessionStorage'));
    return st;
  }
  function diffStateObjs(a, b) {
    const has = (o, k) => Object.prototype.hasOwnProperty.call(o, k), out = [];
    for (const k of Object.keys(b)) {
      if (!has(a, k)) out.push({ op: 'add', key: k, newValue: b[k][1] });
      else if (a[k][0] !== b[k][0]) out.push({ op: 'change', key: k, oldValue: a[k][1], newValue: b[k][1] });
    }
    for (const k of Object.keys(a)) if (!has(b, k)) out.push({ op: 'remove', key: k, oldValue: a[k][1] });
    return out;
  }
  const navInfo = () => {
    let n = null;
    try { n = performance.getEntriesByType('navigation')[0]; } catch {}
    return {
      ts: new Date().toISOString(), url: redactUrl(location.href), referrer: document.referrer ? redactUrl(document.referrer) : '',
      navType: n ? n.type : undefined, redirects: n ? n.redirectCount : undefined, status: n && n.responseStatus ? n.responseStatus : undefined,
    };
  };
  function schedSave() { clearTimeout(saveT); saveT = setTimeout(saveLog, 400); }
  function saveLog() {
    if (!prefs.persist) return;
    clearTimeout(saveT);
    for (const n of [300, 150, 60, 20].map((x) => Math.min(x, CONFIG.maxPersistEntries))) {
      try {
        sessionStorage.setItem(LOG_KEY, JSON.stringify({ v: 1, seq, pageNo, pages: pages.slice(-20), entries: entries.slice(-n), state: stateNow(), base: BASE }));
        return;
      } catch {}
    }
  }
  const removeLog = () => { try { sessionStorage.removeItem(LOG_KEY); } catch {} };
  function restoreLog() { // called once, at the end of the script
    let d = null;
    if (prefs.persist) try { d = JSON.parse(sessionStorage.getItem(LOG_KEY) || 'null'); } catch {}
    if (d && d.v === 1) {
      seq = d.seq || 0;
      pageNo = (d.pageNo || 1) + 1;
      pages.push(...(d.pages || []));
      BASE = d.base || null;
      (d.entries || []).forEach((e) => {
        if (e.pending) { e.pending = false; e.error = e.error || 'page changed before the response arrived'; }
        entries.push(e);
      });
      const cur = stateNow(); // what changed while the old page unloaded and this one loaded (redirect, server Set-Cookie)
      for (const a of ['cookies', 'localStorage', 'sessionStorage']) {
        if (d.state && d.state[a] && cur[a]) diffStateObjs(d.state[a], cur[a]).forEach((c) => pushStore({
          area: a === 'cookies' ? 'cookie' : a, via: 'across page load', ...c,
          note: `changed between page ${d.pageNo} unloading and page ${pageNo} loading (redirect, server Set-Cookie, or an early script)`,
        }));
      }
    }
    pages.push({ pg: pageNo, ...navInfo() });
    addEventListener('DOMContentLoaded', () => { const q = pages[pages.length - 1]; if (q) q.title = document.title; });
    if (prefs.persist) saveLog();
  }
  addEventListener('pagehide', saveLog);
  addEventListener('beforeunload', saveLog);
  document.addEventListener('visibilitychange', () => { if (document.visibilityState === 'hidden') saveLog(); });

  function pageSigs() { // a compact fingerprint of what is on the page: links, buttons, fields, headings, forms
    const out = new Set();
    const q = (sel) => { try { return [...document.querySelectorAll(sel)].filter((el) => !(host && host.contains(el))); } catch { return []; } };
    const label = (el) => trunc(String(txtOf(el) || el.value || el.getAttribute('aria-label') || el.getAttribute('title') || '').replace(/\s+/g, ' ').trim(), 50);
    q('a[href]').slice(0, 120).forEach((el) => out.add(`link "${label(el)}" → ${shortUrl(redactUrl(el.getAttribute('href') || ''))}`));
    q('button, [role=button], input[type=submit], input[type=button]').slice(0, 80).forEach((el) => out.add(`button "${label(el)}"`));
    q('input, select, textarea').slice(0, 80).forEach((el) => { if (el.type !== 'hidden') out.add(`field ${el.tagName.toLowerCase()}[type=${el.type || 'text'}] name=${el.name || el.id || ''}`); });
    q('h1, h2, h3').slice(0, 40).forEach((el) => out.add(`heading "${label(el)}"`));
    q('form').slice(0, 20).forEach((el) => out.add(`form ${String(el.method || 'get').toLowerCase()} ${trunc(el.getAttribute('action') || '', 60)}`));
    return [...out];
  }
  function signals() { // plain facts that usually differ between "login page" and "signed in" (not a verdict)
    const q = (sel) => { try { return [...document.querySelectorAll(sel)].filter((el) => !(host && host.contains(el))); } catch { return []; } };
    const label = (el) => String(txtOf(el) || el.value || el.getAttribute('aria-label') || el.getAttribute('title') || '').replace(/\s+/g, ' ').trim();
    const ctl = q('a, button, [role=button], input[type=submit], input[type=button]').map(label);
    const names = (m) => Object.keys(m || {}).filter((k) => /sess|auth|token|sid|jwt|login|user/i.test(k));
    const st = stateNow();
    return {
      passwordFields: q('input[type=password]').length,
      logoutControls: ctl.filter((t) => /log\s*-?\s*out|sign\s*-?\s*out|log\s*-?\s*off/i.test(t)).length,
      loginControls: ctl.filter((t) => /log\s*-?\s*in|sign\s*-?\s*in/i.test(t)).length,
      authLikeCookies: names(st.cookies),
      authLikeStorageKeys: names(st.localStorage).concat(names(st.sessionStorage)),
    };
  }
  function markBaseline() {
    BASE = { ts: new Date().toISOString(), pg: pageNo, url: redactUrl(location.href), title: document.title, lastId: seq, state: stateNow(), sigs: pageSigs(), signals: signals() };
    if (!prefs.persist) setPref('persist', true); else { saveLog(); draw(); }
    return true;
  }
  function clearBaseline() { BASE = null; saveLog(); draw(); }
  function compareNow() {
    if (!BASE) return null;
    const cur = stateNow(), now = pageSigs(), had = new Set(BASE.sigs), nowSet = new Set(now);
    const byOp = (l) => ({ added: l.filter((c) => c.op === 'add'), changed: l.filter((c) => c.op === 'change'), removed: l.filter((c) => c.op === 'remove') });
    const state = {};
    for (const a of ['cookies', 'localStorage', 'sessionStorage']) if (BASE.state[a] && cur[a]) state[a] = byOp(diffStateObjs(BASE.state[a], cur[a]));
    const url = redactUrl(location.href);
    return {
      baseline: { page: BASE.pg, at: BASE.ts, url: BASE.url, title: BASE.title },
      now: { page: pageNo, url, title: document.title },
      urlChanged: BASE.url !== url, titleChanged: BASE.title !== document.title,
      signals: { before: BASE.signals, after: signals() },
      state,
      elements: { appeared: now.filter((x) => !had.has(x)).slice(0, 60), disappeared: BASE.sigs.filter((x) => !nowSet.has(x)).slice(0, 60) },
      requestsSinceBaseline: entries.filter((e) => e.type === 'network' && e.id > BASE.lastId && !e.pending).slice(-40).map((e) => ({ method: e.method, url: shortUrl(e.url), result: e.error || e.status })),
    };
  }
  function compareReport() {
    const c = compareNow();
    if (!c) return [];
    const j = (v) => (Array.isArray(v) ? v.join(', ') || 'none' : String(v));
    const L = ['', `## Changes since baseline (marked on page ${c.baseline.page} at ${c.baseline.at})`, ''];
    L.push(`- URL: ${c.baseline.url}${c.urlChanged ? ` → ${c.now.url}` : ' (unchanged)'}`);
    L.push(`- Title: "${c.baseline.title}"${c.titleChanged ? ` → "${c.now.title}"` : ' (unchanged)'}`);
    Object.keys(c.signals.after).forEach((k) => L.push(`- ${k}: ${j(c.signals.before[k])} → ${j(c.signals.after[k])}`));
    for (const [a, d] of Object.entries(c.state)) {
      L.push(`- ${a}: ${d.added.length} added, ${d.changed.length} changed, ${d.removed.length} removed`);
      d.added.forEach((x) => L.push(`  - + \`${x.key}\` = ${trunc(x.newValue, 160)}`));
      d.changed.forEach((x) => L.push(`  - ~ \`${x.key}\`: ${trunc(x.oldValue, 80)} → ${trunc(x.newValue, 80)}`));
      d.removed.forEach((x) => L.push(`  - − \`${x.key}\` (was ${trunc(x.oldValue, 80)})`));
    }
    L.push(`- Appeared on the page (${c.elements.appeared.length}):`, ...c.elements.appeared.map((x) => `  - ${x}`));
    L.push(`- Disappeared from the page (${c.elements.disappeared.length}):`, ...c.elements.disappeared.map((x) => `  - ${x}`));
    L.push(`- Requests since baseline (${c.requestsSinceBaseline.length}):`, ...c.requestsSinceBaseline.map((r) => `  - ${r.method} ${r.url} → ${r.result}`));
    return L;
  }

  /* ---------- state refresh + change polling ---------- */
  function refreshSync() { // cheap snapshots that run synchronously (also called before every report)
    if (prefs.dom) domSnap = domNow();
    if (prefs.a11y) a11ySnap = a11yNow();
    if (prefs.appState) appSnap = appStateNow();
  }
  async function refresh() {
    refreshSync();
    const jobs = [];
    if (prefs.cookies && window.cookieStore && cookieStore.getAll) jobs.push(cookieStore.getAll().then((l) => { cookieMeta = l; }).catch(() => {}));
    if (prefs.indexedDB) jobs.push(snapIdb().then((r) => { idbSnap = r; }).catch((e) => { idbSnap = { takenAt: new Date().toISOString(), t: since(), dbs: [], note: String(e) }; }));
    if (prefs.cache) jobs.push(snapCaches().then((r) => { swSnap = r; }).catch((e) => { swSnap = { t: since(), regs: [], caches: [], note: String(e) }; }));
    if (prefs.env) jobs.push((async () => {
      try { if (navigator.storage && navigator.storage.estimate) { const e = await navigator.storage.estimate(); envAsync.quota = { usedMB: +(e.usage / 1048576).toFixed(1), quotaMB: Math.round(e.quota / 1048576) }; } } catch {}
      const perms = {};
      for (const name of ['geolocation', 'notifications', 'camera', 'microphone', 'clipboard-read', 'clipboard-write', 'persistent-storage']) {
        try { perms[name] = (await navigator.permissions.query({ name })).state; } catch {}
      }
      envAsync.perms = perms;
    })());
    await Promise.all(jobs);
    draw();
  }
  function baseline(k) {
    if (k === 'cookies') lastCookies = prefs.cookies ? cookieKeyed() : null;
    else if (k in lastArea) lastArea[k] = prefs[k] ? readArea(k) : null;
  }
  function setPref(k, v) {
    if (k !== 'redact' && !SOURCE_IDS.includes(k)) return;
    prefs[k] = !!v;
    savePrefs();
    baseline(k);
    if (k === 'perf' && v) startPerf();
    if (k === 'persist') { if (v) saveLog(); else removeLog(); }
    refresh();
  }
  function poll() { // catches server Set-Cookie, other tabs, and direct property writes (localStorage.x = 1)
    try {
      if (prefs.cookies) diffCookies().forEach((c) => pushStore({ area: 'cookie', via: 'external/poll', ...c }));
      for (const n of ['localStorage', 'sessionStorage']) if (prefs[n]) diffArea(n).forEach((c) => pushStore({ area: n, via: 'external/poll', ...c }));
      if (prefs.perf && performance.memory) { PERF.mem.push(+(performance.memory.usedJSHeapSize / 1048576).toFixed(1)); if (PERF.mem.length > 120) PERF.mem.shift(); }
    } catch {}
  }
  ['cookies', 'localStorage', 'sessionStorage'].forEach(baseline);
  if (prefs.perf) startPerf();
  setInterval(poll, CONFIG.pollMs);
  const storageNow = () => ({
    cookies: prefs.cookies ? cookiesNow() : undefined,
    localStorage: prefs.localStorage ? areaNow('localStorage') : undefined,
    sessionStorage: prefs.sessionStorage ? areaNow('sessionStorage') : undefined,
    indexedDB: prefs.indexedDB ? idbSnap : undefined,
  });

  /* ---------- report ---------- */
  function summarize() {
    const net = entries.filter((e) => e.type === 'network');
    const done = net.filter((e) => !e.pending && e.durationMs != null);
    const slowest = done.reduce((a, b) => (!a || b.durationMs > a.durationMs ? b : a), null);
    return {
      errors: entries.filter((e) => e.level === 'error').reduce((n, e) => n + (e.count || 1), 0),
      warnings: entries.filter((e) => e.level === 'warn').reduce((n, e) => n + (e.count || 1), 0),
      failedRequests: net.filter(isFailed).length,
      totalRequests: net.length,
      pending: net.filter((e) => e.pending).length,
      slowest: slowest && { ms: slowest.durationMs, label: `${slowest.method} ${shortUrl(slowest.url)}` },
      slowRequests: done.filter((e) => e.durationMs > CONFIG.slowMs).length,
    };
  }
  const shortUrl = (u) => {
    try { const x = new URL(u); return (x.origin === location.origin ? '' : x.host) + x.pathname + x.search; } catch { return u; }
  };
  const head = (e) => {
    if (e.type === 'network') {
      const st = e.pending ? 'pending' : (e.error || `${e.status} ${e.statusText || ''}`.trim());
      return `${e.method} ${shortUrl(e.url)} → ${st}${e.durationMs != null ? ` (${e.durationMs}ms)` : ''}`;
    }
    if (e.type === 'storage') {
      return `[${e.area}${e.level === 'error' ? '/error' : ''}] ${e.op}${e.key != null && e.key !== '' ? ' ' + trunc(e.key, 80) : ''}${e.count > 1 ? ` ×${e.count}` : ''}`;
    }
    if (e.type === 'realtime') return `[${e.chan}#${e.cid}${e.level === 'error' ? '/error' : ''}] ${e.dir}${e.dir === 'connect' || e.dir === 'open' ? ' ' + shortUrl(e.url) : e.data ? ': ' + trunc(String(e.data).replace(/\s+/g, ' '), 120) : ''}${e.count > 1 ? ` ×${e.count}` : ''}`;
    if (e.type === 'activity') return `[${e.kind}] ${trunc(e.msg, 160)}${e.count > 1 ? ` ×${e.count}` : ''}`;
    return `[${e.type}${e.level ? '/' + e.level : ''}] ${String(e.message).split('\n')[0].slice(0, 200)}${e.count > 1 ? ` ×${e.count}` : ''}`;
  };
  function fmt(e, full) {
    const f = (label, v) => (v == null || v === '' ? '' : `- ${label}: ${v}\n`);
    const block = (label, v) => {
      if (v == null || v === '' || (typeof v === 'object' && !Object.keys(v).length)) return '';
      return `- ${label}:\n${FENCE}\n${typeof v === 'string' ? v : JSON.stringify(v, null, 2)}\n${FENCE}\n`;
    };
    const lim = full ? CONFIG.maxBodyChars : 500;
    let out = `### #${e.id} ${head(e)}\n` + f('At', `${at(e)}`);
    if (e.type === 'network') {
      out += f('Via', e.via) + f('URL', e.url) + f('Error', e.error);
      out += block('Request headers', e.requestHeaders);
      out += block('Request body', e.requestBody && trunc(e.requestBody, lim));
      if (full) out += block('Response headers', e.responseHeaders);
      out += block('Response body', e.responseBody && trunc(e.responseBody, lim));
    } else if (e.type === 'storage') {
      out += f('Via', e.via) + f('Note', e.note) + f('Assignment', e.raw) + f('Error', e.level === 'error' ? e.message : '');
      out += block('Old value', e.oldValue != null ? trunc(e.oldValue, lim) : undefined);
      out += block('New value', e.newValue != null ? trunc(e.newValue, lim) : undefined);
      out += block('Stack', e.stack);
    } else if (e.type === 'realtime') {
      out += f('URL', e.url) + block('Data', e.data != null ? trunc(e.data, lim) : undefined);
    } else if (e.type === 'activity') {
      out += f('Detail', e.msg);
    } else {
      out += f('Source', e.source);
      const long = String(e.message).includes('\n') || String(e.message).length > 200;
      if (long && !(e.stack && String(e.stack).includes(String(e.message)))) out += block('Message', e.message);
      out += block('Stack', e.stack);
    }
    return out;
  }
  function storageReport() {
    const L = [], s = storageNow(), lim = CONFIG.maxStorageValueChars;
    const one = (v) => trunc(String(v ?? '').replace(/\s*\n\s*/g, ' '), lim);
    const capped = (arr, fn, max = 200) => arr.slice(0, max).map(fn).concat(arr.length > max ? [`- …and ${arr.length - max} more (see the .json download)`] : []);
    if (s.cookies) {
      L.push('', `## Cookies (${s.cookies.length}; JS-visible only, HttpOnly cookies are not shown)`, '');
      L.push(...(s.cookies.length ? capped(s.cookies, (c) => {
        const attrs = [
          c.domain && `domain=${c.domain}`, c.path && `path=${c.path}`,
          c.path ? (c.expires ? `expires=${new Date(c.expires).toISOString()}` : 'session') : '',
          c.secure && 'secure', c.sameSite && `sameSite=${c.sameSite}`, c.dup && 'DUPLICATE NAME',
        ].filter(Boolean);
        return `- \`${c.name}\` = ${one(c.value)}${attrs.length ? ` (${attrs.join(', ')})` : ''}`;
      }) : ['_None._']));
    }
    for (const n of ['localStorage', 'sessionStorage']) if (s[n]) {
      L.push('', `## ${n} (${s[n].length} keys)`, '');
      L.push(...(s[n].length ? capped(s[n], (x) => `- \`${x.key}\` (${x.size} chars) = ${one(x.value)}`) : ['_Empty._']));
    }
    if (prefs.indexedDB) {
      const d = s.indexedDB;
      if (!d) L.push('', '## IndexedDB', '', '_Not read yet: click ↻ Refresh in the overlay, or run `await __aiFeedback.refresh()` first._');
      else {
        L.push('', `## IndexedDB (${d.dbs.length} databases; snapshot at +${d.t}s, click ↻ Refresh for a newer one)`, '');
        if (d.note) L.push(`_${d.note}_`, '');
        d.dbs.forEach((db) => {
          L.push(`- **${db.name}** v${db.version}${db.error ? ` (could not open: ${db.error})` : ''}`);
          db.stores.forEach((st) => {
            L.push(`  - store \`${st.name}\`: ${st.count} records, keyPath=${safeStringify(st.keyPath)}${st.autoIncrement ? ', autoIncrement' : ''}${st.indexes.length ? `, indexes: ${st.indexes.join(', ')}` : ''}`);
            st.sample.forEach((r) => L.push(`    - ${one(r.key)} → ${one(r.valueText)}`));
          });
        });
      }
    }
    const ch = entries.filter((e) => e.type === 'storage').slice(-50);
    if (ch.length) {
      L.push('', '## Storage changes (last 50, earliest first)', '');
      ch.forEach((e) => L.push(`- #${e.id} ${at(e)} ${head(e)} via ${e.via}${e.note ? ` (${e.note})` : ''}${e.raw ? ` raw: ${one(e.raw)}` : ''}${e.oldValue != null ? ` old: ${one(e.oldValue)}` : ''}${e.newValue != null ? ` new: ${one(e.newValue)}` : ''}`));
    }
    return L;
  }
  function extrasReport() {
    const L = [], lim = CONFIG.maxStorageValueChars;
    const one = (v) => trunc(String(v ?? '').replace(/\s*\n\s*/g, ' '), lim);
    const kvLine = (o) => Object.entries(o || {}).filter(([, v]) => v !== undefined && v !== '' && v !== null).map(([k, v]) => `${k}=${typeof v === 'object' ? safeStringify(v) : v}`).join(', ');
    if (prefs.ws) {
      const rt = entries.filter((e) => e.type === 'realtime').slice(-40);
      L.push('', '## WebSocket / EventSource (last 40 events)', '');
      L.push(...(rt.length ? rt.map((e) => `- #${e.id} ${at(e)} [${e.chan}#${e.cid}] ${e.dir}${e.url && (e.dir === 'connect' || e.dir === 'open') ? ' ' + e.url : ''}${e.data ? ': ' + one(e.data) : ''}${e.count > 1 ? ` ×${e.count}` : ''}`) : ['_None._']));
    }
    if (prefs.input || prefs.nav || prefs.mutations || prefs.postMessage) {
      const act = entries.filter((e) => e.type === 'activity').slice(-60);
      L.push('', `## Activity trail (last 60; on: ${['input', 'nav', 'mutations', 'postMessage'].filter((k) => prefs[k]).join(', ')})`, '');
      if (prefs.mutations) L.push(`_Watching: ${watchers.length ? watchers.map((w) => `${w.selector} (${w.elements})`).join(', ') : 'nothing; call __aiFeedback.observe(selector)'}_`, '');
      L.push(...(act.length ? act.map((e) => `- ${at(e)} [${e.kind}] ${one(e.msg)}${e.count > 1 ? ` ×${e.count}` : ''}`) : ['_None._']));
    }
    if (prefs.perf) {
      const p = perfNow();
      L.push('', '## Performance', '');
      L.push(`- FCP: ${p.fcpMs ?? 'n/a'} ms; LCP: ${p.lcp ? `${p.lcp.ms} ms ${p.lcp.el || p.lcp.url}` : 'n/a'}; CLS: ${p.cls}; first input delay: ${p.firstInputDelayMs ?? 'n/a'} ms`);
      if (p.nav) L.push(`- Navigation: ${kvLine(p.nav)}`);
      L.push(`- Long tasks (>50 ms): ${p.longTasks.count}, total ${p.longTasks.totalMs} ms, worst ${p.longTasks.worstMs} ms`);
      L.push(`- JS heap: ${p.heap ? `${p.heap.nowMB} MB now, ${p.heap.peakMB} MB peak` : 'not exposed by this browser'}`);
      if (p.slowResources.length) L.push(`- Slow resources (> ${CONFIG.slowResourceMs} ms):`, ...p.slowResources.map((r) => `  - ${r.ms} ms ${r.type} ${shortUrl(r.url)} (${r.kb} KB)`));
    }
    if (prefs.env) {
      L.push('', '## Environment & permissions', '');
      Object.entries(envNow()).forEach(([g, o]) => { if (o) L.push(`- ${g}: ${kvLine(o)}`); });
    }
    if (prefs.cache) {
      L.push('', '## Service workers & caches', '');
      if (!swSnap) L.push('_Not read yet: click ↻ Refresh or run `await __aiFeedback.refresh()` first._');
      else {
        if (swSnap.note) L.push(`_${swSnap.note}_`);
        L.push(`- Page controlled by a service worker: ${swSnap.controlled ? 'yes' : 'no'}`);
        swSnap.regs.forEach((r) => L.push(`- registration ${r.scope} script=${r.script} state=${r.state}${r.waiting ? ' (update waiting)' : ''}${r.installing ? ' (installing)' : ''}`));
        swSnap.caches.forEach((c) => L.push(`- cache \`${c.name}\`: ${c.count} entries${c.urls.length ? ', e.g. ' + c.urls.slice(0, 5).map(shortUrl).join(', ') : ''}`));
      }
    }
    if (prefs.a11y && a11ySnap) {
      L.push('', '## Accessibility & DOM checks (basic, not a full audit)', '');
      L.push(...(a11ySnap.length ? a11ySnap.map((i) => `- ${i.rule}${i.count > 1 || i.examples.length ? ` ×${i.count}` : ''}${i.examples.length ? ` e.g. ${i.examples.join(', ')}` : ''}`) : ['_No issues found._']));
    }
    if (prefs.dom && domSnap) L.push('', `## DOM snapshot (${domSnap.nodes} elements, ${domSnap.chars} chars; scripts, secrets and data: URIs stripped)`, '', FENCE + 'html', domSnap.html, FENCE);
    if (prefs.appState && appSnap) {
      L.push('', '## App state', '');
      const names = Object.keys(appSnap);
      if (!names.length) L.push('_Nothing registered; the app needs __aiFeedback.watch(name, () => state)._');
      names.forEach((n) => L.push(`- **${n}**`, FENCE + 'json', appSnap[n], FENCE));
    }
    return L;
  }
  function buildReport({ errorsOnly = false } = {}) {
    refreshSync();
    const s = summarize();
    const problems = entries.filter(isProblem);
    const onIds = SOURCE_IDS.filter((k) => prefs[k]), offIds = SOURCE_IDS.filter((k) => !prefs[k]);
    const L = [
      '# App Debug Report', '',
      `- Captured: ${new Date().toISOString()}`,
      `- Page: ${location.href}`,
      `- User agent: ${navigator.userAgent}`,
      `- Viewport: ${innerWidth}x${innerHeight}`,
      `- Session uptime: ${since()}s`,
      `- Capture on: ${onIds.join(', ') || 'none'}; secrets ${prefs.redact ? 'redacted' : 'NOT redacted'}`,
      `- Capture off: ${offIds.map((k) => `${SOURCE_LABEL[k]} (${k})`).join('; ') || 'none'}`,
      ...(offIds.length ? ['- Need more? Ask the human to press Alt+Shift+D > Capture settings, tick what you need, reproduce the problem and Copy report again (or reopen the page with ?aidebug=<ids>,open).'] : []), '',
      '## Summary', '',
      `- Errors: ${s.errors}`, `- Warnings: ${s.warnings}`,
      `- Failed requests: ${s.failedRequests} of ${s.totalRequests}${s.pending ? ` (${s.pending} still pending)` : ''}`,
      ...(s.slowest ? [`- Slowest request: ${s.slowest.ms}ms (${s.slowest.label}); ${s.slowRequests} over ${CONFIG.slowMs}ms`] : []),
      ...(pages.length > 1 || prefs.persist ? ['', '## Page loads in this tab (p1 = first)', '', ...pages.map((q) => `- p${q.pg} ${q.ts} ${q.navType || ''}${q.redirects ? ` redirects=${q.redirects}` : ''}${q.status ? ` status=${q.status}` : ''} ${q.url}${q.referrer ? ` (from ${q.referrer})` : ''}${q.title ? ` "${q.title}"` : ''}`)] : []),
      '', '## Problems (earliest first; fix the earliest first)', '',
      problems.length ? problems.map((e) => fmt(e, true)).join('\n') : '_None captured._',
    ];
    if (!errorsOnly) {
      const warns = entries.filter((e) => e.level === 'warn');
      if (warns.length) L.push('', '## Warnings', '', warns.map((e) => fmt(e, false)).join('\n'));
      L.push(...compareReport(), ...storageReport(), ...extrasReport());
      const ok = entries.filter((e) => e.type === 'network' && !isProblem(e));
      L.push('', '## Other network requests', '', ok.length ? ok.map((e) => `- #${e.id} ${at(e)} ${head(e)}`).join('\n') : '_None._');
      const logs = entries.filter((e) => e.type === 'console' && (e.level === 'log' || e.level === 'info' || e.level === 'debug')).slice(-30);
      if (logs.length) L.push('', '## Recent console output (last 30)', '', logs.map((e) => `- ${at(e)} [${e.level}] ${trunc(e.message, 300).replace(/\n/g, ' ')}${e.count > 1 ? ` ×${e.count}` : ''}`).join('\n'));
    }
    L.push('', '_Bodies are truncated and sensitive keys redacted. Generated by the in-app debug overlay._');
    return L.join('\n') + '\n';
  }
  const buildJson = () => { refreshSync(); return safeStringify({
    meta: {
      captured: new Date().toISOString(), url: location.href, userAgent: navigator.userAgent, viewport: `${innerWidth}x${innerHeight}`, uptimeSec: +since(),
      capture: { cookies: prefs.cookies, localStorage: prefs.localStorage, sessionStorage: prefs.sessionStorage, indexedDB: prefs.indexedDB, redact: prefs.redact },
    },
    summary: summarize(),
    storage: storageNow(),
    pageLoads: pages,
    comparison: compareNow() || undefined,
    extras: {
      perf: prefs.perf ? perfNow() : undefined, environment: prefs.env ? envNow() : undefined, serviceWorkersAndCaches: prefs.cache ? swSnap : undefined,
      accessibility: prefs.a11y ? a11ySnap : undefined, dom: prefs.dom ? domSnap : undefined, appState: prefs.appState ? appSnap : undefined,
      watchedSelectors: prefs.mutations ? watchers : undefined,
    },
    entries,
  }); };

  /* ---------- overlay ---------- */
  let host, root, isOpen = false, positioned = false, tab = 'problems', query = '', timer, msgTimer;
  const ui = new Map(); // open/closed state of <details> blocks, keyed by data-k, so redraws keep them
  const isOn = (k, def) => (ui.has(k) ? ui.get(k) : def);
  const $ = (sel) => root.querySelector(sel);
  const TABS = [['problems', 'Problems'], ['network', 'Network'], ['console', 'Console'], ['cookies', 'Cookies'], ['storage', 'Storage'], ['realtime', 'Realtime'], ['activity', 'Activity'], ['perf', 'Perf'], ['page', 'Page'], ['compare', 'Compare']];

  function build() {
    host = document.createElement('div');
    host.id = '__ai-debug-overlay';
    root = host.attachShadow({ mode: 'open' });
    const cfgHtml = SOURCES.map(([g, items]) =>
      `<div class="grp"><b>${esc(g)}</b>` +
      items.map(([id, label, desc]) => `<label><input type="checkbox" data-pref="${id}"><span>${esc(label)} <small>${esc(desc)}</small></span></label>`).join('') +
      '</div>').join('');
    root.innerHTML = `
<style>
:host{all:initial}*{box-sizing:border-box}[hidden]{display:none!important}
.panel{position:fixed;right:12px;bottom:12px;width:min(660px,calc(100vw - 24px));height:min(78vh,740px);min-width:min(380px,100vw);min-height:260px;max-width:100vw;max-height:100vh;resize:both;display:flex;flex-direction:column;background:#111418;color:#e6e8eb;font:12px/1.45 ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;border:1px solid #2c333b;border-radius:10px;box-shadow:0 12px 40px rgba(0,0,0,.55);z-index:2147483647;overflow:hidden}
header{display:flex;justify-content:space-between;align-items:center;gap:8px;padding:8px 10px;border-bottom:1px solid #2c333b;font-weight:600;cursor:move;user-select:none;-webkit-user-select:none;touch-action:none;background:#161b21}
.hint{font-weight:400;color:#7d8794;white-space:nowrap}
#msg{font-weight:400;color:#7ee2a8}
.stats{display:flex;flex-wrap:wrap;gap:6px;padding:8px 10px}
.chip{padding:2px 8px;border-radius:999px;background:#1d232a}
.chip.bad{background:#5c1f24;color:#ffb3b8}.chip.warn{background:#4d3b0f;color:#ffd98a}.chip.info{background:#17304d;color:#a9d1ff}
.tabs{display:flex;flex-wrap:wrap;gap:4px;padding:0 10px 6px}
.bar{display:flex;gap:6px;padding:0 10px 6px}
input[type=search],input[type=text]{flex:1;min-width:0;background:#0b0d10;color:inherit;border:1px solid #2c333b;border-radius:6px;padding:4px 8px;font:inherit}
.cfg{padding:0 10px 6px;color:#aab3bd}.cfg>summary{cursor:pointer;color:#7d8794}
.cfgbody{max-height:34vh;overflow:auto;padding-right:4px}
.grp{margin-top:8px}.grp>b{color:#cdd5de}
.cfg label{display:flex;gap:6px;align-items:baseline;margin:3px 0;cursor:pointer}
.cfg small{color:#7d8794}
.cfgbtns{display:flex;flex-wrap:wrap;gap:6px;margin:8px 0 2px}
button{font:inherit;color:inherit;background:#1d232a;border:1px solid #2c333b;border-radius:6px;padding:4px 9px;cursor:pointer}
button:hover{background:#27303a}button.on{background:#2f5d9e;border-color:#2f5d9e}
.list{flex:1;overflow:auto;padding:4px 10px 8px}
.row{margin:0 0 4px;border-left:3px solid #2c333b;padding-left:8px}
.row.err{border-color:#e5484d}.row.warn{border-color:#f5a524}
summary{cursor:pointer;word-break:break-all}
pre{white-space:pre-wrap;word-break:break-all;margin:4px 0;color:#aab3bd}
.sec{margin:6px 0}.sec .sec{margin-left:12px}
.sec>summary{font-weight:600;padding:3px 0;color:#cdd5de;word-break:normal}
.kv{border-left:3px solid #2c333b;padding-left:8px;margin:0 0 3px}
.kv>summary{white-space:nowrap;overflow:hidden;text-overflow:ellipsis;word-break:normal}
.kv .k{color:#9ecbff}.kv .pv{color:#aab3bd}
.tag{display:inline-block;padding:0 6px;margin:0 2px;border-radius:999px;background:#1d232a;color:#9aa4b0;font-size:11px}
.tag.bad{background:#5c1f24;color:#ffb3b8}.tag.ok{background:#12351f;color:#8ee0ae}.tag.warn{background:#4d3b0f;color:#ffd98a}
.tbl{border-collapse:collapse;width:100%;margin:4px 0}
.tbl td{padding:2px 10px 2px 0;vertical-align:top;border-bottom:1px solid #1d232a;word-break:break-all}
.tbl td:first-child{color:#7d8794;white-space:nowrap;width:1%}
.spark{color:#7ee2a8;letter-spacing:-1px}
.note{color:#7d8794;margin:4px 0 8px}
.empty{color:#7d8794}
footer{display:flex;flex-wrap:wrap;gap:6px;padding:8px 10px;border-top:1px solid #2c333b}
textarea{display:block;width:calc(100% - 20px);height:90px;margin:0 10px 8px;background:#0b0d10;color:#e6e8eb;border:1px solid #2c333b;border-radius:6px;font:inherit}
textarea[hidden]{display:none}
</style>
<div class="panel" role="dialog" aria-label="Debug overlay">
  <header title="Drag to move; double-click to reset position"><span>AI Debug Overlay <span id="msg"></span></span><span class="hint">drag to move · Alt+Shift+D to close</span></header>
  <div class="stats" id="stats"></div>
  <div class="tabs">${TABS.map(([id, label]) => `<button data-tab="${id}">${label}</button>`).join('')}</div>
  <div class="bar"><input id="q" type="search" placeholder="Filter this tab…" autocomplete="off" spellcheck="false"><button data-act="refresh" title="Re-read cookie attributes, IndexedDB, caches, DOM, etc.">↻ Refresh</button></div>
  <div class="bar" id="obsbar" hidden><input id="obs" type="text" placeholder="CSS selector to watch for DOM changes, e.g. #board" autocomplete="off" spellcheck="false"><button data-act="observe">Watch</button></div>
  <details class="cfg"><summary>Capture settings: choose what to extract</summary>
    <div class="cfgbody">
      ${cfgHtml}
      <div class="grp"><label><input type="checkbox" data-pref="redact"><span>Redact secrets <small>cookie/storage/state values with secret-looking names or JWTs; resets to on every page load</small></span></label></div>
    </div>
    <div class="cfgbtns"><button data-act="all">Tick all</button><button data-act="none">Untick all</button><button data-act="link" title="Copies a URL that switches the ticked sources on from page load">Copy capture link</button></div>
    <div class="note">History-type sources (Realtime, Activity, Perf, state change logs) only record from the moment they are on. To record from page load, use the capture link.</div>
  </details>
  <div class="list" id="list"></div>
  <textarea id="fallback" hidden readonly></textarea>
  <footer>
    <button data-act="copy">Copy report</button><button data-act="copyErr">Copy errors only</button>
    <button data-act="dlMd">Download .md</button><button data-act="dlJson">Download .json</button>
    <button data-act="clear">Clear</button><button data-act="close">Close</button>
  </footer>
</div>`;
    const doObserve = () => {
      const el = $('#obs'), n = el.value.trim() ? observe(el.value.trim()) : 0;
      say(n ? `Watching ${n} element(s)` : 'No element matches that selector');
      draw();
    };
    root.addEventListener('click', (ev) => {
      const b = ev.target.closest('button');
      if (!b) return;
      if (b.dataset.tab) { tab = b.dataset.tab; if (['cookies', 'storage', 'perf', 'page'].includes(tab)) refresh(); draw(); }
      const act = b.dataset.act;
      if (act === 'copy') copy(buildReport());
      if (act === 'copyErr') copy(buildReport({ errorsOnly: true }));
      if (act === 'dlMd') download('debug-report.md', buildReport(), 'text/markdown');
      if (act === 'dlJson') download('debug-report.json', buildJson(), 'application/json');
      if (act === 'clear') { entries.length = 0; ui.clear(); draw(); }
      if (act === 'close') toggle(false);
      if (act === 'refresh') refresh().then(() => say('Refreshed ✓'));
      if (act === 'enable') setPref(b.dataset.what, true);
      if (act === 'observe') doObserve();
      if (act === 'mark') { markBaseline(); say('Baseline marked'); }
      if (act === 'unmark') clearBaseline();
      if (act === 'all' || act === 'none') {
        SOURCE_IDS.forEach((k) => { prefs[k] = act === 'all'; baseline(k); });
        savePrefs();
        if (act === 'all') startPerf();
        refresh();
      }
      if (act === 'link') {
        try {
          const u = new URL(location.href);
          u.searchParams.set('aidebug', SOURCE_IDS.filter((k) => prefs[k]).concat('open').join(','));
          copy(u.href);
        } catch { say('Could not build a link for this URL'); }
      }
    });
    root.addEventListener('change', (ev) => {
      const i = ev.target.closest && ev.target.closest('[data-pref]');
      if (i) setPref(i.dataset.pref, i.checked);
    });
    root.addEventListener('input', (ev) => { if (ev.target.id === 'q') { query = ev.target.value; draw(); } });
    root.addEventListener('toggle', (ev) => {
      const k = ev.target.dataset && ev.target.dataset.k;
      if (k) ui.set(k, ev.target.open);
    }, true);
    ['keydown', 'keyup', 'keypress'].forEach((t) => root.addEventListener(t, (ev) => { // keep typing away from the app's shortcuts
      ev.stopPropagation();
      if (t === 'keydown' && ev.key === 'Enter' && ev.target.id === 'obs') doObserve();
    }));
    initDrag();
    document.documentElement.appendChild(host);
  }

  /* ----- dragging ----- */
  function place(x, y) {
    const p = $('.panel'), w = p.offsetWidth;
    x = Math.min(Math.max(x, 100 - w), innerWidth - 100); // always keep part of the title bar reachable
    y = Math.min(Math.max(y, 0), innerHeight - 40);
    p.style.left = x + 'px'; p.style.top = y + 'px'; p.style.right = 'auto'; p.style.bottom = 'auto';
  }
  function initPos() {
    const r = $('.panel').getBoundingClientRect(), want = prefs.pos || { x: r.left, y: r.top };
    place(want.x, want.y);
    positioned = true;
  }
  function initDrag() {
    const p = $('.panel'), hdr = $('header');
    let d = null;
    hdr.addEventListener('pointerdown', (e) => {
      if (e.button !== 0 || e.target.closest('button')) return;
      const r = p.getBoundingClientRect();
      d = { dx: e.clientX - r.left, dy: e.clientY - r.top };
      try { hdr.setPointerCapture(e.pointerId); } catch {}
      e.preventDefault();
    });
    hdr.addEventListener('pointermove', (e) => { if (d) place(e.clientX - d.dx, e.clientY - d.dy); });
    const end = (e) => {
      if (!d) return;
      d = null;
      try { hdr.releasePointerCapture(e.pointerId); } catch {}
      const r = p.getBoundingClientRect();
      prefs.pos = { x: Math.round(r.left), y: Math.round(r.top) };
      savePrefs();
    };
    hdr.addEventListener('pointerup', end);
    hdr.addEventListener('pointercancel', end);
    hdr.addEventListener('dblclick', () => {
      prefs.pos = null; savePrefs();
      p.style.left = p.style.top = ''; p.style.right = '12px'; p.style.bottom = '12px';
      positioned = false; initPos();
    });
    addEventListener('resize', () => { if (positioned && isOpen) { const r = p.getBoundingClientRect(); place(r.left, r.top); } });
  }

  /* ----- view helpers ----- */
  const tag = (t, cls = '') => `<span class="tag ${cls}">${esc(t)}</span>`;
  const preview = (v, n = 90) => { const s = String(v).replace(/\s+/g, ' '); return s.length > n ? s.slice(0, n) + '…' : s; };
  const pretty = (v) => {
    const s = String(v);
    if (/^\s*[\[{]/.test(s)) try { return JSON.stringify(JSON.parse(s), null, 2); } catch {}
    const dec = safeDecode(s);
    return dec !== s ? `${s}\n\n— decoded —\n${dec}` : s;
  };
  const kvRow = (k, key, value, tags = '') => {
    const v = String(value == null ? '' : value);
    return `<details class="kv" data-k="${esc(k)}" ${isOn(k, false) ? 'open' : ''}><summary><b class="k">${esc(key || '(no name)')}</b> ${tags} <span class="pv">${esc(preview(v))}</span></summary><pre>${esc(trunc(pretty(v)))}</pre></details>`;
  };
  const section = (k, title, body, count, def = true) =>
    `<details class="sec" data-k="${esc('sec:' + k)}" ${isOn('sec:' + k, def) ? 'open' : ''}><summary>${esc(title)} ${count != null ? tag(count) : ''}</summary>${body}</details>`;
  const off = (what, msg) => `<p class="note">${esc(msg)} <button data-act="enable" data-what="${what}">Turn on</button></p>`;
  const table = (rows) => `<table class="tbl">${rows.map(([k, v]) => `<tr><td>${esc(k)}</td><td>${v}</td></tr>`).join('')}</table>`;
  const spark = (arr) => { const b = '▁▂▃▄▅▆▇█', a = arr.slice(-40), lo = Math.min(...a), hi = Math.max(...a); return a.map((v) => b[hi === lo ? 0 : Math.round((v - lo) / (hi - lo) * 7)]).join(''); };
  const matches = (q, ...parts) => !q || parts.join(' ').toLowerCase().includes(q);
  const expStr = (x) => { const d = new Date(x); return isNaN(d) ? String(x) : d.toISOString().slice(0, 16).replace('T', ' '); };

  function entryRows(list, q, max) {
    return list
      .filter((e) => !q || (head(e) + ' ' + fmt(e, false)).toLowerCase().includes(q))
      .slice(-max).reverse()
      .map((e) => {
        const cls = isProblem(e) ? 'err' : e.level === 'warn' ? 'warn' : '';
        const k = 'e:' + e.id;
        return `<details class="row ${cls}" data-k="${k}" ${isOn(k, false) ? 'open' : ''}><summary>${esc(head(e))}</summary><pre>${esc(fmt(e, true))}</pre></details>`;
      }).join('');
  }

  /* ----- views (one per tab) ----- */
  function viewEntries(q) {
    const pick = {
      problems: isProblem,
      network: (e) => e.type === 'network',
      console: (e) => !['network', 'storage', 'realtime', 'activity'].includes(e.type),
    }[tab];
    return (tab === 'network' ? pageLoadsHtml() : '') + (entryRows(entries.filter(pick), q, 200) || '<p class="empty">Nothing captured here.</p>');
  }
  function pageLoadsHtml() {
    if (pages.length < 2 && !prefs.persist) return '';
    return section('pages', 'Page loads in this tab', table(pages.slice().reverse().map((q) => [`p${q.pg}`,
      esc(`${String(q.ts).slice(11, 19)} ${q.navType || ''}${q.redirects ? ` redirects=${q.redirects}` : ''}${q.status ? ` status=${q.status}` : ''} ${shortUrl(q.url)}${q.referrer ? ` ← ${shortUrl(q.referrer)}` : ''}${q.title ? ` "${q.title}"` : ''}`)])), pages.length);
  }
  function viewCompare() {
    const btns = `<div class="cfgbtns"><button data-act="mark">${BASE ? 'Re-mark baseline here' : 'Mark baseline now'}</button>${BASE ? '<button data-act="unmark">Clear baseline</button>' : ''}</div>`;
    if (!BASE) return `<p class="note">See what changed between two moments, for example the login page and the page after signing in. Mark a baseline on the first page, do the thing (sign in), then come back to this tab or copy the report. Marking switches on "Keep log across page loads" so the baseline survives redirects and reloads. Tick cookies, localStorage and sessionStorage in Capture settings first if you want them compared.</p>${btns}`;
    const c = compareNow();
    const fv = (v) => esc(Array.isArray(v) ? v.join(', ') || 'none' : String(v));
    const chg = (d) => [...d.added.map((x) => ['+ ' + x.key, esc(x.newValue)]), ...d.changed.map((x) => ['~ ' + x.key, esc(`${x.oldValue} → ${x.newValue}`)]), ...d.removed.map((x) => ['− ' + x.key, esc(x.oldValue)])];
    let h = btns + `<p class="note">Baseline marked on page ${c.baseline.page} at ${esc(c.baseline.at.slice(11, 19))}. Values are masked; only names and masked values are compared.</p>`;
    h += table([
      ['URL', esc(c.urlChanged ? `${c.baseline.url} → ${c.now.url}` : `${c.now.url} (unchanged)`)],
      ['Title', esc(c.titleChanged ? `"${c.baseline.title}" → "${c.now.title}"` : `"${c.now.title}" (unchanged)`)],
      ...Object.keys(c.signals.after).map((k) => [k, `${fv(c.signals.before[k])} → ${fv(c.signals.after[k])}`]),
    ]);
    for (const [a, d] of Object.entries(c.state)) {
      const rows = chg(d);
      h += section('cmp-' + a, a, rows.length ? table(rows) : '<p class="empty">No changes.</p>', rows.length);
    }
    h += section('cmp-new', 'Appeared on the page', c.elements.appeared.length ? `<pre>${esc(c.elements.appeared.join('\n'))}</pre>` : '<p class="empty">Nothing.</p>', c.elements.appeared.length);
    h += section('cmp-gone', 'Disappeared from the page', c.elements.disappeared.length ? `<pre>${esc(c.elements.disappeared.join('\n'))}</pre>` : '<p class="empty">Nothing.</p>', c.elements.disappeared.length);
    h += section('cmp-req', 'Requests since baseline', c.requestsSinceBaseline.length ? table(c.requestsSinceBaseline.map((r) => [String(r.result), esc(`${r.method} ${r.url}`)])) : '<p class="empty">None.</p>', c.requestsSinceBaseline.length);
    return h;
  }
  function viewCookies(q) {
    if (!prefs.cookies) return off('cookies', 'Cookie capture is off.');
    const err = cookieError();
    let h = err ? `<p class="note">Cookies are not readable here (${esc(err)}). Sandboxed iframes block them.</p>` : '';
    const list = cookiesNow().filter((c) => matches(q, c.name, c.value));
    h += `<p class="note">${list.length} cookie(s) visible to JavaScript; HttpOnly cookies are never shown.${location.protocol === 'file:' ? ' Pages opened from file:// usually cannot store cookies: serve over http://localhost.' : ''}${window.cookieStore ? '' : ' Domain, path, expiry and SameSite need the Cookie Store API (Chromium, https or localhost).'}</p>`;
    h += list.map((c, i) => {
      const tags = [
        c.dup && tag('duplicate name', 'bad'),
        c.secure === true && tag('Secure', 'ok'),
        c.sameSite && tag(`SameSite=${c.sameSite}`),
        c.domain && tag(c.domain),
        c.path && tag(`path=${c.path}`),
        c.path && (c.expires ? tag('exp ' + expStr(c.expires)) : tag('session')),
        tag(`${c.size}B`),
      ].filter(Boolean).join(' ');
      return kvRow(`ck:${c.name}:${i}`, c.name, c.value, tags);
    }).join('') || '<p class="empty">No cookies.</p>';
    const ch = entryRows(entries.filter((e) => e.type === 'storage' && e.area === 'cookie'), q, 100);
    return h + section('cookie-log', 'Change log', ch || '<p class="empty">No cookie changes seen since the page loaded.</p>');
  }
  function viewStorage(q) {
    let h = '';
    for (const n of ['localStorage', 'sessionStorage']) {
      if (!prefs[n]) { h += section(n, n, off(n, `${n} capture is off.`)); continue; }
      const items = areaNow(n).filter((x) => matches(q, x.key, x.value));
      h += section(n, n, items.map((x) => kvRow(`${n}:${x.key}`, x.key, x.value, tag(`${x.size} chars`))).join('') || '<p class="empty">Empty.</p>', items.length);
    }
    if (!prefs.indexedDB) h += section('idb', 'IndexedDB', off('indexedDB', 'IndexedDB capture is off.'));
    else if (!idbSnap) h += section('idb', 'IndexedDB', '<p class="empty">Reading…</p>');
    else {
      let body = idbSnap.note ? `<p class="note">${esc(idbSnap.note)}</p>` : '';
      body += `<p class="note">Snapshot at +${esc(idbSnap.t)}s (sample of ${CONFIG.idbSampleRecords} records per store). Use ↻ Refresh for a newer one.</p>`;
      body += idbSnap.dbs.map((d) => section(`idb:${d.name}`, `${d.name} v${d.version}`,
        (d.error ? `<p class="note">Could not open: ${esc(d.error)}</p>` : '') +
        d.stores.map((st) => section(`idb:${d.name}/${st.name}`, st.name,
          `<p class="note">keyPath: ${esc(safeStringify(st.keyPath))}${st.autoIncrement ? ' · autoIncrement' : ''}${st.indexes.length ? ' · indexes: ' + esc(st.indexes.join(', ')) : ''}</p>` +
          (st.sample.filter((r) => matches(q, r.key, r.valueText)).map((r, i) => kvRow(`idbk:${d.name}/${st.name}:${i}`, r.key, r.valueText)).join('') || '<p class="empty">No records.</p>'),
          `${st.count} records`)).join(''),
        `${d.stores.length} stores`)).join('') || '<p class="empty">No databases.</p>';
      h += section('idb', 'IndexedDB', body, idbSnap.dbs.length);
    }
    const ch = entryRows(entries.filter((e) => e.type === 'storage' && e.area !== 'cookie'), q, 100);
    return h + section('storage-log', 'Change log', ch || '<p class="empty">No storage changes seen since the page loaded.</p>');
  }
  function viewRealtime(q) {
    if (!prefs.ws) return off('ws', 'WebSocket / EventSource capture is off. Only connections opened after it is on are seen; use the capture link to record from page load.');
    return entryRows(entries.filter((e) => e.type === 'realtime'), q, 200) || '<p class="empty">No WebSocket or EventSource activity yet.</p>';
  }
  function viewActivity(q) {
    const defs = [['input', 'Click / input trail'], ['nav', 'Page events'], ['postMessage', 'postMessage'], ['mutations', 'DOM mutations']];
    const offs = defs.filter(([k]) => !prefs[k]);
    let h = offs.map(([k, l]) => off(k, `${l} is off.`)).join('');
    if (prefs.mutations) h += `<p class="note">Watching: ${watchers.length ? esc(watchers.map((w) => `${w.selector} (${w.elements})`).join(', ')) : 'nothing yet, enter a selector above'}. Elements added later are not watched; call Watch again.</p>`;
    return h + (entryRows(entries.filter((e) => e.type === 'activity'), q, 200) || (offs.length < defs.length ? '<p class="empty">No activity yet.</p>' : ''));
  }
  function viewPerf(q) {
    if (!prefs.perf) return off('perf', 'Performance capture is off.');
    const p = perfNow();
    const rate = (v, good, poor) => (v == null ? '' : v <= good ? 'ok' : v > poor ? 'bad' : 'warn');
    const rows = [
      ['First contentful paint', p.fcpMs != null ? tag(`${p.fcpMs} ms`, rate(p.fcpMs, 1800, 3000)) : '—'],
      ['Largest contentful paint', p.lcp ? tag(`${p.lcp.ms} ms`, rate(p.lcp.ms, 2500, 4000)) + ' ' + esc(p.lcp.el || p.lcp.url) : '—'],
      ['Cumulative layout shift', tag(String(p.cls), rate(p.cls, 0.1, 0.25))],
      ['First input delay', p.firstInputDelayMs != null ? tag(`${p.firstInputDelayMs} ms`, rate(p.firstInputDelayMs, 100, 300)) : '—'],
      ...(p.nav ? [['Navigation', esc(`${p.nav.type}: TTFB ${p.nav.ttfbMs} ms · DOMContentLoaded ${p.nav.domContentLoadedMs} ms · load ${p.nav.loadMs} ms`)]] : []),
      ['Long tasks (>50 ms)', esc(`${p.longTasks.count} · total ${p.longTasks.totalMs} ms · worst ${p.longTasks.worstMs} ms`)],
      ['JS heap', p.heap ? esc(`${p.heap.nowMB} MB now · ${p.heap.peakMB} MB peak `) + `<span class="spark">${esc(spark(p.heap.samples))}</span>` : '<span class="empty">not exposed by this browser</span>'],
    ].filter(([k, v]) => matches(q, k, v));
    return table(rows) + section('slowres', `Slow resources (> ${CONFIG.slowResourceMs} ms)`,
      p.slowResources.length ? table(p.slowResources.map((r) => [`${r.ms} ms`, esc(`${r.type} ${shortUrl(r.url)} (${r.kb} KB)`)])) : '<p class="empty">None.</p>', p.slowResources.length);
  }
  function viewPage(q) {
    let h = ['env', 'cache', 'a11y', 'dom', 'appState'].filter((k) => !prefs[k]).map((k) => off(k, `${SOURCE_LABEL[k]} is off.`)).join('');
    if (prefs.env) {
      const rows = [];
      Object.entries(envNow()).forEach(([g, o]) => {
        if (o) Object.entries(o).forEach(([k, v]) => { if (v !== undefined && v !== '' && v !== null) rows.push([`${g}.${k}`, esc(typeof v === 'object' ? safeStringify(v) : String(v))]); });
      });
      h += section('env', 'Environment & permissions', table(rows.filter(([k, v]) => matches(q, k, v))));
    }
    if (prefs.cache) {
      let b = '';
      if (!swSnap) b = '<p class="empty">Reading…</p>';
      else {
        b = (swSnap.note ? `<p class="note">${esc(swSnap.note)}</p>` : '') +
          `<p class="note">Page is ${swSnap.controlled ? '' : 'not '}controlled by a service worker · ${swSnap.regs.length} registration(s) · ${swSnap.caches.length} cache(s)</p>` +
          swSnap.regs.map((r, i) => kvRow(`sw:${i}`, r.scope, `${r.script}\nstate: ${r.state}${r.waiting ? ' (update waiting)' : ''}${r.installing ? ' (installing)' : ''}`, tag(r.state || 'unknown', r.state === 'activated' ? 'ok' : 'warn'))).join('') +
          swSnap.caches.filter((c) => matches(q, c.name, c.urls.join(' '))).map((c) => kvRow(`cache:${c.name}`, c.name, c.urls.join('\n') + (c.count > c.urls.length ? `\n…and ${c.count - c.urls.length} more` : ''), tag(`${c.count} entries`))).join('');
      }
      h += section('cache', 'Service workers & caches', b);
    }
    if (prefs.a11y) {
      const items = (a11ySnap || []).filter((i) => matches(q, i.rule, i.examples.join(' ')));
      h += section('a11y', 'Accessibility & DOM checks', a11ySnap
        ? (items.map((i) => kvRow(`a11y:${i.rule}`, i.rule, `${i.hint || ''}\n\n${i.examples.join('\n')}`, tag(`×${i.count}`, 'warn'))).join('') || '<p class="empty">No issues found by these basic checks (this is not a full audit).</p>')
        : '<p class="empty">Checking…</p>', a11ySnap ? a11ySnap.length : null);
    }
    if (prefs.dom) {
      h += section('dom', 'DOM snapshot', domSnap
        ? `<p class="note">${domSnap.nodes} elements, ${domSnap.chars} chars${domSnap.chars > CONFIG.maxDomChars ? ' (truncated)' : ''}. Scripts, secrets and data: URIs are stripped; typed input values are not included.</p><pre>${esc(domSnap.html)}</pre>`
        : '<p class="empty">Reading…</p>', null, false);
    }
    if (prefs.appState) {
      const names = Object.keys(appSnap || {});
      h += section('state', 'App state', names.length
        ? names.filter((n) => matches(q, n, appSnap[n])).map((n) => kvRow(`app:${n}`, n, appSnap[n])).join('')
        : '<p class="note">Nothing registered. In the app: __aiFeedback.watch("store", () => store.getState())</p>', names.length);
    }
    return h;
  }

  function draw() {
    if (!isOpen) return;
    const s = summarize();
    const chip = (txt, cls) => `<span class="chip ${cls || ''}">${esc(txt)}</span>`;
    const cnt = (t) => entries.filter((e) => e.type === t).length;
    $('#stats').innerHTML =
      chip(`${s.errors} errors`, s.errors ? 'bad' : '') +
      chip(`${s.warnings} warnings`, s.warnings ? 'warn' : '') +
      chip(`${s.failedRequests} failed / ${s.totalRequests} requests`, s.failedRequests ? 'bad' : '') +
      (s.pending ? chip(`${s.pending} pending`) : '') +
      (s.slowest ? chip(`slowest ${s.slowest.ms}ms`, s.slowRequests ? 'warn' : '') : '') +
      (prefs.cookies ? chip(`${readCookies().length} cookies`, 'info') : '') +
      (prefs.localStorage ? chip(`${readArea('localStorage').size} localStorage`, 'info') : '') +
      (prefs.sessionStorage ? chip(`${readArea('sessionStorage').size} sessionStorage`, 'info') : '') +
      (prefs.indexedDB && idbSnap ? chip(`${idbSnap.dbs.length} IndexedDB`, 'info') : '') +
      (prefs.ws ? chip(`${cnt('realtime')} realtime`, 'info') : '') +
      (prefs.input || prefs.nav || prefs.mutations || prefs.postMessage ? chip(`${cnt('activity')} activity`, 'info') : '') +
      (prefs.a11y && a11ySnap && a11ySnap.length ? chip(`${a11ySnap.length} a11y issues`, 'warn') : '');
    root.querySelectorAll('[data-tab]').forEach((b) => b.classList.toggle('on', b.dataset.tab === tab));
    root.querySelectorAll('[data-pref]').forEach((i) => { i.checked = !!prefs[i.dataset.pref]; });
    $('#obsbar').hidden = !(tab === 'activity' && prefs.mutations);
    const q = query.trim().toLowerCase();
    const views = { cookies: viewCookies, storage: viewStorage, realtime: viewRealtime, activity: viewActivity, perf: viewPerf, page: viewPage, compare: viewCompare };
    const html = (views[tab] || viewEntries)(q);
    const list = $('#list'), st = list.scrollTop;
    list.innerHTML = html;
    list.scrollTop = st;
  }
  function render() {
    if (!isOpen) return;
    clearTimeout(timer);
    timer = setTimeout(draw, 120);
  }
  function say(m) {
    if (!isOpen) return;
    $('#msg').textContent = m;
    clearTimeout(msgTimer);
    msgTimer = setTimeout(() => { if (isOpen) $('#msg').textContent = ''; }, 2500);
  }
  async function copy(text) {
    let ok = false;
    try { await navigator.clipboard.writeText(text); ok = true; } catch {}
    if (!ok) {
      try {
        const ta = document.createElement('textarea');
        ta.value = text; ta.style.cssText = 'position:fixed;opacity:0';
        document.body.appendChild(ta); ta.select();
        ok = document.execCommand('copy'); ta.remove();
      } catch {}
    }
    const fb = $('#fallback');
    if (ok) { fb.hidden = true; say('Copied ✓'); }
    else { fb.hidden = false; fb.value = text; fb.focus(); fb.select(); say('Clipboard blocked: press Ctrl/Cmd+C'); }
    return ok;
  }
  function download(name, text, type) {
    const a = document.createElement('a');
    a.href = URL.createObjectURL(new Blob([text], { type }));
    a.download = name;
    document.body.appendChild(a); a.click(); a.remove();
    setTimeout(() => URL.revokeObjectURL(a.href), 1000);
  }
  function toggle(force) {
    if (!host) build();
    isOpen = typeof force === 'boolean' ? force : !isOpen;
    host.style.display = isOpen ? '' : 'none';
    if (isOpen) { if (!positioned) initPos(); draw(); refresh(); }
  }

  addEventListener('keydown', (e) => {
    const h = CONFIG.hotkey;
    if (e.code === h.code && e.altKey === h.alt && e.shiftKey === h.shift && e.ctrlKey === h.ctrl && e.metaKey === h.meta) {
      e.preventDefault(); toggle();
    } else if (e.key === 'Escape' && isOpen) toggle(false);
  }, true);
  if (autoOpen) addEventListener('DOMContentLoaded', () => toggle(true));
  restoreLog();

  /* Programmatic access for AI agents with their own browser tooling */
  window.__aiFeedback = {
    report: buildReport,
    json: buildJson,
    entries: () => entries.slice(),
    summary: summarize,
    storage: storageNow,        // current cookie / Web Storage / IndexedDB state (only the sources that are switched on)
    refresh,                    // async: re-reads cookie attributes, IndexedDB, caches, environment; await it before report()/storage()
    setCapture: setPref,        // setCapture('ws', true); ids: cookies, localStorage, sessionStorage, indexedDB, cache, appState, ws, postMessage, input, nav, mutations, perf, env, a11y, dom, plus 'redact'
    sources: () => SOURCE_IDS.filter((k) => prefs[k]),
    mark: markBaseline,         // mark a baseline (e.g. on the login page); survives reloads/redirects. Then sign in and call compare() or report()
    compare: compareNow,        // what changed since the baseline: URL/title, cookie + storage names, elements, signals, requests
    clearBaseline,
    watch: (name, getter) => { watches.set(String(name), getter); }, // register app state (Redux/Pinia/any object) so the human or AI can ask for it
    observe,                    // observe('#board'): log DOM mutations under matching elements (needs the 'mutations' source)
    clear: () => { entries.length = 0; draw(); },
    toggle,
  };
})();
</script>
```

## Customizing

Adjust `CONFIG` at the top of the script instead of rewriting logic:

- `hotkey`: change `code` (e.g. `'KeyE'`) or modifiers if the app already uses Alt+Shift+D.
- `capture`: which sources start on (all except cookies are off by default). Set a flag here when you need history from page load. The human's checkbox choices, once made, are remembered in `localStorage.__aiDebugPrefs` and override these defaults; `?aidebug=…` in the URL switches sources on for that load.
- `maxBodyChars`, `maxStorageValueChars`, `idbSampleRecords`, `maxDomChars`, `maxPersistEntries`: report size controls. Raise them when debugging large payloads; lower them if reports get too long to paste. `caps` limits how many realtime/activity/storage entries are kept so noisy sources cannot push out errors and requests.
- `redactKeys` / `redactHeaders`: extend with app-specific secret names (e.g. `/ssn|card|iban/i`). `redactStorageKeys` is the equivalent list for cookie, storage and state names.
- `slowResourceMs`, `slowMs`: thresholds for the Perf tab and the slow-request chip. `pollMs`: how often cookies and Web Storage are diffed.
- To add a source, add it to `SOURCES` (this creates the checkbox and the `?aidebug=` id), gate its listeners on `prefs.<id>`, and add a view and a report section.

Other request layers (axios, ky, SWR, React Query, Apollo) all sit on top of `fetch` or `XMLHttpRequest`, so they are captured with no extra work. Storage libraries (js-cookie, localforage, idb, Dexie, zustand/redux persist) sit on `document.cookie`, `localStorage` or `IDBObjectStore`, so they are captured too. Socket.io and similar libraries run over `WebSocket`, so their frames appear under Realtime.

## Verifying the install

Before telling the user it's ready, confirm it works. If you can run a browser, load the page, trigger an error (`setTimeout(() => { throw new Error('overlay-test') })`) and a failing request (`fetch('/__nope')`), then check that `window.__aiFeedback.report()` lists both under **Problems**. If the app uses cookies or storage, also run `__aiFeedback.setCapture('localStorage', true); document.cookie = 'overlay_test=1; path=/'; localStorage.setItem('overlay_test', '1')` and check that both appear under **Storage changes**. If the app has a login, check that `__aiFeedback.mark()` followed by `__aiFeedback.compare()` runs without error. If you cannot run a browser, re-read the page's `<head>` to be sure the script is the first script and is not marked `defer`, `async`, or `type="module"`.
