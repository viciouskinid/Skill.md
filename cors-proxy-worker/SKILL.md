---
name: cors-proxy-worker
description: Use when browser JavaScript needs to call a third-party API that blocks cross-origin requests (CORS errors, "No Access-Control-Allow-Origin header"), or when the user mentions their corsproxy / Cloudflare Worker proxy at corsproxy.viciouskinid.workers.dev. Covers how to call the proxy, its limits, and how to diagnose failures.
---

# CORS proxy Worker

A Cloudflare Worker that forwards requests to a target URL and adds CORS headers to the response, so browser code can read it.

## Endpoint

    https://corsproxy.viciouskinid.workers.dev/?url=<ENCODED_TARGET_URL>

The target URL MUST be passed through `encodeURIComponent`. It also accepts the unencoded form `?https://example.com/path`, but that breaks when the target has its own query string, so always use `?url=`.

## Calling it

```js
const PROXY = "https://corsproxy.viciouskinid.workers.dev/?url=";
const target = "https://api.example.com/data?x=1&y=2";
const res = await fetch(PROXY + encodeURIComponent(target));
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
```

POST/PUT/PATCH/DELETE work: method, body and most request headers are forwarded.

## Behaviour

- Only `http:` and `https:` targets. Anything else returns 400.
- Missing/invalid `url` returns 400 with a usage message.
- Upstream network failure returns 502 with the error text.
- Upstream status codes and bodies pass through unchanged (a 404 from the target is a 404 here).
- Stripped before forwarding: `Origin`, `Referer`, `Cookie`, `Host`. `Set-Cookie` is stripped from responses.
- Preflight (`OPTIONS`) is answered by the Worker with 204.
- If `ALLOWED_ORIGINS` is set in the Worker, other origins get 403. A page opened from `file://` sends `Origin: null` and will be rejected.

## Limits

- Free plan: 100,000 requests/day, shared across everything on the account.
- Cannot send cookies or credentials to the target. Use for public data only.
- Some targets block Cloudflare IPs or require specific headers; failures there look like 403/5xx from upstream.

## Error reporting in pages that use the proxy

Any page or tool built on this proxy MUST show a copyable diagnostics panel when a load fails, so the user can paste it back for diagnosis instead of describing the problem. On any failure (network error, non-2xx status, JSON parse error, or valid JSON missing the expected data) replace the output area with:

- an `<h2>` ("Couldn't load ...") and the short error message
- a read-only `<textarea>` holding a JSON report
- a "Copy diagnostics" button (use `navigator.clipboard.writeText`, fall back to selecting the textarea and `document.execCommand("copy")`) and a "Try again" button

The JSON report must include:

- `error` (name and message) and `failedAt` (the stage: fetch, parse JSON, read data, render)
- `target` (the decoded upstream URL) and `requestUrl` (the full proxy URL)
- `status` and all `responseHeaders` (`Object.fromEntries(res.headers.entries())`)
- `bodyPreview`: the first ~600 characters of the response text, captured before parsing, plus `jsonKeys` when the JSON parsed but lacked the expected data
- `page`: `location.origin` and `location.href` (reveals `file://` / `Origin: null`)
- `time`, `userAgent`, and a `hint` for timeouts (`AbortError`) or `TypeError` (network/CORS)

Use an `AbortController` timeout (~20s) on the fetch. Read the body with `res.text()` first, then `JSON.parse`, so the preview is available even when parsing fails.

## Diagnosing failures

Ask the user for: the full request URL, HTTP status, response headers, first ~600 characters of the body, the page origin, and the browser console error.

| Symptom | Likely cause |
|---|---|
| `TypeError: Failed to fetch`, no status | Browser blocked it: Worker 403 without CORS headers, offline, or ad-blocker. Check Origin vs `ALLOWED_ORIGINS`. |
| 400 "Invalid URL" | Target not encoded, or missing scheme. |
| 403 "Origin not allowed" | Page origin not in `ALLOWED_ORIGINS` (including `null` for `file://`). |
| 502 | Worker could not reach the target (DNS, TLS, timeout). |
| 200 but JSON parse error | Target returned HTML (bot check, error page). Inspect body preview. |
| 429 / 1015 | Daily quota or target rate-limit hit. |

Test directly in a terminal to separate browser issues from proxy issues:

    curl -i -H "Origin: https://yoursite.com" "https://corsproxy.viciouskinid.workers.dev/?url=https%3A%2F%2Fexample.com"
