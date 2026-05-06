---
title: "Chrome DevTools for Scraping — Breakpoints and Network Forensics"
date: 2026-05-06
updated: 2026-05-06
tags: [devtools, chrome, breakpoints, har, network-panel, debugger, reverse-engineering]
---

# Chrome DevTools for Scraping — Breakpoints and Network Forensics

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `devtools` `chrome` `breakpoints` `har` `network-panel` `debugger` `reverse-engineering`

---

## Table of Contents

1. The DevTools workflow for reverse-engineering scrapes
2. The Network panel — your primary forensics surface
3. HAR export and replay
4. The Sources panel — pretty-printing, formatting, persistent breakpoints
5. Breakpoint types in DevTools
6. Blackboxing scripts (Ignore List)
7. Workspaces and Overrides — local edits that survive reload
8. The Performance panel — call trees for opaque event chains
9. The Memory panel — heap snapshots and closure state
10. The Application panel — cookies, storage, IndexedDB, cache
11. Console tricks for live introspection
12. The Recorder panel — replay and Puppeteer/Playwright export
13. End-to-end: reverse-engineering a signed/encrypted request
14. The Initiator column and call-stack tracing
15. Source maps in DevTools (quick use)
16. Shadow DOM, iframes, web workers, service workers
17. When DevTools is too noisy — handing off to mitmproxy / Burp

## Summary

Chrome DevTools is the single most productive tool for reverse-engineering modern web applications. Most scrapers fail not because the target is technically impregnable but because the engineer never used the debugger. This doc walks the panels in the order you actually use them on a real job — Network to spot the request, Sources to find where it is built, breakpoints to freeze the runtime mid-construction, and the call stack to follow the data backwards to its source. It also covers HAR export for handing evidence between tools, Workspaces and Overrides for live edits, the Recorder for capturing flows as Puppeteer scripts, and the Application panel for debugging session state. Every menu path, keyboard shortcut, and breakpoint type is what a working reverse engineer actually uses; nothing is theoretical.

The audience is a TS/Node backend engineer comfortable with code but not necessarily fluent in DevTools. By the end, given a page that loads a signed JSON payload via XHR, you should be able to (1) find the request, (2) freeze the runtime where the signature is built, (3) walk the stack to the seed function, and (4) replicate the logic in Node. That is the loop. Everything else is decoration.

---

## 1. The DevTools workflow for reverse-engineering scrapes

The panels chain in a specific order on real reverse-engineering work:

```
Network ──▶ Sources ──▶ Console ──▶ Application ──▶ Performance
   │            │           │              │              │
   │            │           │              │              └─ when an event
   │            │           │              │                 chain is opaque
   │            │           │              │                 and you need a
   │            │           │              │                 call tree
   │            │           │              └─ cookies / storage /
   │            │           │                 cache / IndexedDB
   │            │           │                 (session state)
   │            │           └─ live evaluation, $0, copy(),
   │            │              monitorEvents
   │            └─ the file, pretty-printed,
   │               with breakpoints set
   └─ "what request am I targeting?"
      filter, copy as cURL, HAR
```

The mental loop:

1. **Network** — identify the request that returns the data you want. Filter to XHR/Fetch. Note headers, payload, query params.
2. **Sources** — find the function that constructs that request. XHR breakpoint with the URL substring is the fastest way in.
3. **Console** — inspect locals and globals at the breakpoint. Use `$0`, `copy()`, evaluate small expressions.
4. **Application** — verify cookies, storage, IndexedDB hold what you expect. Modify values to test session handling.
5. **Performance** — only when the event chain is opaque (e.g., a click triggers ten handlers across three frameworks and you cannot tell which one fires the request).

You rarely need all five on a single job. Network → Sources → Console covers 80% of reverse-engineering. Application enters the picture for session-bound APIs. Performance enters for SPAs with heavy event delegation.

---

## 2. The Network panel — your primary forensics surface

Open DevTools (`Cmd+Opt+I` on macOS, `F12` on Windows/Linux), select **Network**.

### 2.1 Filters

The filter bar is more powerful than most people use:

| Filter | Effect |
|--------|--------|
| `XHR` / `Fetch` button | Restrict to XHR + Fetch (named `Fetch/XHR` in current Chrome) |
| Plain text in filter box | Substring match on URL |
| `/regex/` syntax | Regex match against URL |
| `-string` | Negative match (exclude rows containing `string`) |
| `domain:api.example.com` | All requests to this host |
| `has-response-header:x-csrf-token` | Only rows whose response carried this header |
| `larger-than:50k` | Size threshold (bytes, `k`, `m`) |
| `method:POST` | HTTP method filter |
| `mime-type:application/json` | MIME type filter |
| `status-code:401` | Status filter |
| `is:running` | In-flight requests only |
| `cookie-domain:example.com` | Filter by cookie scope |
| `set-cookie-name:sessionid` | Rows that set a specific cookie |

These can be combined: `has-response-header:content-encoding mime-type:application/json larger-than:10k -ads`.

### 2.2 Capture controls

- **Preserve log** — keep entries across navigations. **Always on** for reverse engineering. Without it, a redirect or full-page navigation flushes the panel and you lose the OAuth callback or login POST.
- **Disable cache** — disables HTTP cache while DevTools is open. Turn on so you see the real request flow each time, not a 304.
- **Throttling dropdown** — simulate `Slow 3G`, `Fast 3G`, `Offline`, or a custom profile. Useful for catching race conditions where the scraper assumes synchronous availability of a resource. Bandwidth and latency only — not a CPU governor (that lives in the Performance panel).
- **Big request rows** (right-side ⋮) — toggles two-line rows so you see size + time without hovering.

### 2.3 Blocking patterns

Right-click a row → **Block request URL** or **Block request domain**. This adds an entry to the **Network request blocking** drawer (also reachable via `Cmd+Shift+P` → "Show network request blocking"). Useful for:

- Confirming a tracker isn't load-bearing before you skip it in your scraper.
- Forcing a fallback path you suspect exists.
- Reproducing the experience of a blocked CDN.

### 2.4 Copy as ...

Right-click a row → **Copy** offers:

- `Copy as cURL (bash)` — full reproduction including cookies and headers. The macOS "cmd" form quotes correctly for zsh/bash.
- `Copy as cURL (cmd)` — Windows cmd-quoted variant.
- `Copy as PowerShell` — Invoke-WebRequest form.
- `Copy as fetch` — browser-side `fetch()` call. Useful for pasting into the Console to confirm it still works in-page.
- `Copy as fetch (Node.js)` — Node 18+ global `fetch`. The export you usually want for a TS scraper.
- `Copy as HAR (with sensitive data)` — single-entry HAR for sharing with another tool.
- `Copy response` — body only.

Pasting `Copy as cURL` into a terminal and watching it succeed proves the request is reproducible without the browser. That is the moment you know a scraper is feasible.

### 2.5 Reading a row

Click a row → side panel opens with tabs: **Headers**, **Payload**, **Preview**, **Response**, **Initiator**, **Timing**, **Cookies**.

- **Headers** shows request and response headers, plus the request line and remote address.
- **Payload** breaks form/JSON bodies into a tree.
- **Preview** is rendered (image, JSON tree, HTML).
- **Response** is raw text.
- **Initiator** (covered in §14) is the call stack of what triggered the request — the most important tab on this list.
- **Timing** shows queueing, blocked, DNS, SSL, TTFB, and content-download phases. Useful to distinguish "server is slow" from "we're blocking on something else first".

---

## 3. HAR export and replay

A HAR file (HTTP Archive) is a JSON document containing every request/response pair captured in the Network panel. The original spec is by Jan Odvarko (2007) and was later maintained by the W3C Web Performance Working Group.

### 3.1 Exporting

In the Network panel, right-click the empty area or any row → **Save all as HAR (with content)** or **Save all as HAR (with sensitive data)**. Modern Chrome distinguishes the two — the latter includes cookies and Authorization headers, the former redacts them. Pick the version that matches who will see the file.

### 3.2 What HAR contains

The top-level shape (per the original 1.2 specification):

```json
{
  "log": {
    "version": "1.2",
    "creator": { "name": "WebInspector", "version": "537.36" },
    "pages": [ { "startedDateTime": "...", "id": "page_1", "title": "..." } ],
    "entries": [
      {
        "startedDateTime": "2026-05-06T10:32:10.123Z",
        "time": 184.7,
        "request":  { "method": "POST", "url": "...", "headers": [...], "postData": {...} },
        "response": { "status": 200, "headers": [...], "content": { "text": "..." } },
        "cache": {},
        "timings": { "blocked": 0.4, "dns": -1, "connect": -1, "send": 0.2, "wait": 110, "receive": 73.7 },
        "_initiator": { "type": "script", "stack": {...} }
      }
    ]
  }
}
```

`time` is total request duration in ms. `_initiator` is a Chrome extension (underscore-prefixed by HAR convention for vendor fields).

### 3.3 Replaying a HAR

- **Postman** — File → Import → drag the HAR. Postman creates a collection with one request per entry. Folders preserve hostname.
- **Insomnia** — Application menu → Import/Export → Import Data → From File → select HAR. Insomnia produces a request collection.
- **Burp Suite** — Tools (Burp Pro) ship a HAR import via the Logger or an extension. The community route is to replay individual entries by right-clicking and "Send to Repeater".
- **chrome-har-capturer** — Node tool by `cyrus-and` that drives Chrome over the DevTools Protocol and captures HARs programmatically. Also useful for reading existing HARs into JS for diffing.
- **har-to-postman** / **har-to-curl** — small CLIs to convert HAR entries into runnable shell scripts.

The reverse-engineering value of HAR is that it freezes the entire session — every request, every response, every header and cookie — into one file. You can hand it to a teammate, replay it tomorrow when the site has changed, or diff two runs to find what's different.

---

## 4. The Sources panel — pretty-printing, formatting, persistent breakpoints

### 4.1 Layout

Sources has three panes:

- **Left** — file navigator (Page, Filesystem, Overrides, Snippets, Content scripts, sometimes Workspace).
- **Center** — file editor with line numbers, syntax highlighting, the gutter for breakpoints.
- **Right** — debugger panels: Watch, Breakpoints, Scope, Call Stack, XHR/Fetch Breakpoints, DOM Breakpoints, Global Listeners, Event Listener Breakpoints, CSP Violation Breakpoints.

### 4.2 Finding the file

`Cmd+P` (or `Ctrl+P`) opens a file fuzzy-finder across all loaded resources. Type part of the name (`auth`, `signature`, the request path you saw in Network) and Chrome jumps to the file. `Cmd+Shift+P` opens the command palette (everything DevTools can do).

`Cmd+Shift+F` runs full-text search across all loaded resources — invaluable for finding "what file builds the `X-Signature` header?" by searching for the literal string `X-Signature`.

### 4.3 Pretty-print

Minified JS is unreadable. The `{ }` curly-braces icon at the bottom-left of the editor pretty-prints the file. The pretty-printed view persists across reloads as long as DevTools is open. Once formatted, line numbers in the editor and in error traces refer to the prettified source — not the original — which is sometimes confusing when comparing to logged stack traces.

For TypeScript or framework-built bundles, pretty-print is the floor; the ceiling is source maps (§15) where available.

### 4.4 Persistent breakpoints

Breakpoints set in the Sources panel survive across reloads. Even better, they survive across sessions if the URL of the file is stable. For files served with a content-hashed name (`main.a1b2c3.js`), the breakpoint is lost the next deploy — workaround in §7 is to use Overrides so you control the file content and its name.

---

## 5. Breakpoint types in DevTools

This is where reverse engineering happens. There are seven breakpoint types and each has a specific use.

### 5.1 Line breakpoints

Click a line number in the gutter. The runtime pauses when execution reaches that line. Standard.

Right-click the breakpoint marker for further options:
- **Edit breakpoint** — convert to conditional or logpoint.
- **Disable breakpoint** — temporarily off without losing the position.
- **Remove breakpoint**.

### 5.2 Conditional breakpoints

Right-click a line number → **Add conditional breakpoint**. Enter an expression. The breakpoint pauses only when the expression evaluates truthy.

Examples:
```js
// pause only when the request URL contains 'sign'
url.includes('sign')

// pause only on user 12345
user.id === 12345

// pause every 10th call
window._n = (window._n || 0) + 1, window._n % 10 === 0
```

The expression runs in the local scope, so you have access to function parameters and local variables. Side effects in the expression are real and persist (the increment in the third example actually mutates `window._n`).

### 5.3 Logpoints

Right-click a line number → **Add logpoint**. Enter an expression. The runtime evaluates the expression and prints the result to the Console without pausing.

```js
// log message instead of pausing
'sig built', signature, ' from ', body
```

Logpoints are a non-invasive `console.log` for code you do not own. You don't have to redeploy or even reload — set the logpoint, trigger the action, read the output. Indispensable for production debugging on a site you cannot edit.

### 5.4 DOM breakpoints

In the **Elements** panel, right-click any node → **Break on**:

- **Subtree modifications** — pauses when any descendant is added or removed.
- **Attribute modifications** — pauses when an attribute on this node changes.
- **Node removal** — pauses when this node is removed.

Useful when a value appears on the page and you cannot find what wrote it. Set "Subtree modifications" on the parent container, watch for the pause, walk up the call stack.

DOM breakpoints are listed in the right pane of Sources under **DOM Breakpoints** so you can disable them later.

### 5.5 XHR/Fetch breakpoints

Sources panel → right pane → **XHR/Fetch Breakpoints** → click `+` → enter a URL substring (or leave blank for "any XHR").

When the next request whose URL contains that substring fires, the runtime pauses **at the call site that issued the request** (or in the framework wrapper, which you'll then step out of). This is the single most useful breakpoint for reverse engineering. You see exactly what code was about to make the request and which arguments were being passed.

The substring match is a plain `String.prototype.includes` check — no regex.

### 5.6 Event listener breakpoints

Sources panel → right pane → **Event Listener Breakpoints**. A tree of every event the page can react to:

- Mouse: `click`, `mousedown`, `mouseup`, `wheel`, ...
- Keyboard: `keydown`, `keyup`, `keypress`.
- Touch.
- XHR: `readystatechange`, `load`, `loadstart`, `loadend`, `progress`, `abort`, `error`, `timeout`.
- Worker.
- Window: `load`, `beforeunload`, `popstate`, `hashchange`.
- Timer: `setTimeout`, `setInterval`, `animationFrame`.
- Many more.

Tick a box and the runtime pauses inside the matching listener the next time it fires. Use case: a hidden global click handler intercepts every click, attaches a tracking pixel, then proceeds. Tick `Mouse → click`, click anywhere, and you land in the handler.

### 5.7 Exception breakpoints

In the right pane of Sources, two checkboxes:

- **Pause on uncaught exceptions** — break on errors that escape all `try`/`catch`.
- **Pause on caught exceptions** — break even on errors that are swallowed.

The second is what you want when a site silently catches `fetch` failures and you cannot tell what went wrong. Once you find the catch, set a logpoint on the catch line and turn the exception breakpoint back off.

---

## 6. Blackboxing scripts (Ignore List)

Frameworks (React, Vue, jQuery, Angular runtime) generate enormous call stacks. Stepping through them is a waste of time. DevTools' **Ignore List** (formerly "Blackboxing") tells the debugger to skip stepping through configured scripts. The runtime still executes them — DevTools just doesn't surface them in the call stack or step into them.

Three ways to add to the Ignore List:

1. **Right-click in the call stack** → **Add script to ignore list**. Done.
2. **Right-click a file in the file tree** → **Add script to ignore list**.
3. **Settings → Ignore List** (gear icon). Add patterns by regex. Default patterns include `node_modules`, `bower_components`, anonymous scripts, content scripts.

Recommended additions for reverse engineering:
- `node_modules` (default)
- `\.min\.js$`
- `react(-dom)?(\.production|\.development)?\.js`
- `vendor\.[a-f0-9]+\.js` (Webpack vendor chunk)
- Any framework chunk name you keep stepping through

The behavior change is dramatic. A 30-frame call stack collapses to 3-4 frames of *your target's actual code*, which is what you want.

---

## 7. Workspaces and Overrides — local edits that survive reload

### 7.1 Workspaces

Sources → **Filesystem** tab → **Add folder to workspace**. Pick a local folder. Chrome maps URL paths to local files; edits in DevTools save to disk. Used for live-editing your own dev server.

For scraping reverse engineering, Workspaces are less central than Overrides because you usually don't have the source.

### 7.2 Overrides

Sources → **Overrides** tab → **Select folder for overrides** → grant permission. Now any file you edit in Sources can be saved with **right-click → Save for overrides**. Reload the page — Chrome serves your edited version instead of the network response.

What this lets you do during reverse engineering:

- Inject a `console.trace()` at the top of a function you want to fingerprint without modifying the build.
- Replace a complex obfuscated function with a simpler version that exports the inputs and outputs to `window`.
- Force a feature flag on by editing `if (config.experimental)` to `if (true)`.
- Patch out an anti-debugging trick (e.g., `setInterval(() => debugger, 100)`) by deleting the offending line and saving as override.

Overrides are scoped per-host. The folder structure mirrors the URL: `cdn.example.com/main.abc123.js` → `<override-folder>/cdn.example.com/main.abc123.js`. Chrome shows a purple dot next to overridden files in the file tree.

Overrides survive across DevTools sessions. They do not survive across deploys when filenames are content-hashed — clear and redo.

---

## 8. The Performance panel — call trees for opaque event chains

When you click a button and twelve things happen across three frameworks and you can't tell what fired the request, record a Performance trace.

Steps:

1. Open Performance panel.
2. Click the **Record** button (or `Cmd+E`).
3. Perform the interaction — click the button.
4. Stop recording.
5. Find the request in the **Network** track of the timeline.
6. Click the request's send phase. The **Bottom-Up** / **Call Tree** / **Event Log** tabs at the bottom show what code was running.
7. The **Initiator** column on a network request from the recorded session links back to the function that issued it.

The Performance panel also exposes CPU throttling (`4x slowdown`, `6x slowdown`) which is useful for finding race conditions that are masked on a fast machine.

For deep CPU work the Performance Insights panel (newer) and the underlying Trace Event Format are referenced in the DevTools docs.

---

## 9. The Memory panel — heap snapshots and closure state

Modern SPAs hide state in closures. A token, a signing key, or a session secret may live on the closure of a function that returns nothing. The Memory panel can find it.

- **Heap snapshot** — full snapshot of every reachable object. Use the **Containment** view to inspect closures (scope chains appear as `Closure (functionName)` nodes with their captured variables visible underneath).
- **Allocation sampling** — lighter, useful to see what's being allocated over time.
- **Allocation instrumentation on timeline** — record while interacting; see exactly when objects are allocated.

Worked technique for closure-hidden tokens:
1. Take a heap snapshot before login.
2. Log in.
3. Take a heap snapshot after login.
4. Filter to "objects allocated between snapshot 1 and 2".
5. Search for the token shape (long base64 string) in the snapshot's strings list.
6. Right-click → "Reveal in Containment view" to see what closure holds it.

Less common than the Sources panel but unmatched for "where is this value actually stored?".

---

## 10. The Application panel — cookies, storage, IndexedDB, cache

Application panel (formerly Resources) is where session state lives.

- **Cookies** — per-domain. Editable inline. Right-click a row to delete or copy. The columns include `HttpOnly`, `Secure`, `SameSite`, `Partitioned`, expiry, and Priority. For scraping you care which cookies are `HttpOnly` (you cannot read them from JS, only from the network), and which are `SameSite=Strict` / `Lax` / `None`.
- **Local Storage** — per-origin. Plain text key-value. Editable inline.
- **Session Storage** — per-tab origin.
- **IndexedDB** — structured database. Useful when a site stores tokens or app state in IDB.
- **Cache Storage** — service worker caches. Inspect to see what the SW is intercepting.
- **Service Workers** — list, unregister, `Update on reload`, `Bypass for network`. The latter is essential when an SW is rewriting your fetches.
- **Manifest** / **Background Services** / **Frames** / **Storage** (root summary).

The reverse-engineering pattern: change a cookie value to test session validation; clear local storage to see if a token rebuilds; bypass the service worker to confirm it's not the one signing requests.

---

## 11. Console tricks for live introspection

The Console runs JS in the page's main frame's global scope (or a selected iframe via the context dropdown).

| Helper | Purpose |
|--------|---------|
| `$0` | Last selected element in the Elements panel |
| `$1` … `$4` | Previously selected elements |
| `$_` | Last evaluated expression result |
| `$('selector')` / `$$('selector')` | `querySelector` / `querySelectorAll` (only when no library shadows them) |
| `copy(value)` | Copy to clipboard. Best for `copy(JSON.stringify(obj))` |
| `keys(obj)`, `values(obj)` | Like `Object.keys`/`Object.values` |
| `monitorEvents(window, ['click', 'scroll'])` | Log every event of those types fired on the target |
| `unmonitorEvents(window)` | Stop monitoring |
| `monitor(fn)` | Log every call of `fn` with its args. Note: this helper has been deprecated in newer Chrome but is still present in many channels — verify before relying on it |
| `unmonitor(fn)` | Stop |
| `getEventListeners(node)` | Returns listeners attached to a node, grouped by event type |
| `queryObjects(Constructor)` | Lists all live instances of a class (needs Profile context) |
| `debug(fn)` | Auto-set a breakpoint at the start of a function the next time it runs |
| `undebug(fn)` | Remove |
| `inspect(value)` | Reveal in Elements (DOM) or Sources (function) |
| `clear()` | Clear console |

The Console also accepts top-level `await`, so `await fetch('/api/me').then(r => r.json())` works directly.

The **Recorder** option in the Console settings (gear icon) lets you preserve console output across navigations — helpful for tracking auth flows.

---

## 12. The Recorder panel — replay and Puppeteer/Playwright export

DevTools includes a **Recorder** panel (added 2022, stable since). Use it to:

1. Open the Recorder panel.
2. Click **Create a new recording**.
3. Name it, click **Start recording**.
4. Perform the user flow (login, click through pages, submit forms).
5. Stop recording. The flow is now a JSON document of steps.

Useful actions:

- **Replay** — re-runs the recording. Optional speed control. Fastest way to validate a flow is reproducible.
- **Edit** — modify selectors, add waits, add assertions.
- **Export** — JSON, Puppeteer, Puppeteer (for Firefox), Puppeteer Replay, or **Playwright Test**. Choosing Playwright Test gives you a `.spec.ts` file that already runs in your existing Playwright project with minor tweaks.

For scraping, this turns hours of "what did the user actually click to reach this state?" into a five-minute capture-and-export. The exported Puppeteer/Playwright code is the skeleton; you replace the assertions with extraction logic.

---

## 13. End-to-end: reverse-engineering a signed/encrypted request

A common protected endpoint pattern: the request includes an `X-Signature` header derived from `(timestamp, nonce, body, secret)` via HMAC-SHA256 or similar. Replicate the construction in Node.

The walk:

**a. Network panel — identify the protected request.**

Filter `Fetch/XHR`, find `POST /api/v2/orders`. Headers show `X-Timestamp`, `X-Nonce`, `X-Signature`. The signature changes every call so it's not a static token.

**b. XHR breakpoint on the URL substring.**

Sources → XHR/Fetch Breakpoints → `+` → enter `/api/v2/orders`. Click the action that triggers the request again. Runtime pauses at the call site. The call stack on the right shows the chain. Frequently the top frame is in `axios` or `fetch` wrapper — uninteresting. Step **out** (Shift+F11) until you reach your target's code. Add the wrapper file to the Ignore List so you skip it next time.

**c. Walk up the call stack.**

In the Call Stack panel, click each frame in turn. The Scope panel updates to that frame's locals. You're looking for a frame where `signature` was just assigned. When you find it, observe the line above:

```js
const signature = sign(timestamp + nonce + JSON.stringify(body), secret)
```

**d. Find `sign` and `secret`.**

`Cmd+click` on `sign` to jump to its definition. It's likely a small helper that calls `crypto.subtle` or a CryptoJS function. Note the hash algorithm (HMAC-SHA256, plain SHA256, etc.) and the encoding (`hex` vs `base64`).

`secret` is harder. It might be:
- A literal string in the bundle (search for it with `Cmd+Shift+F`).
- A value computed at boot from `document.cookie` or `localStorage`.
- Returned by an earlier `/api/config` request (check the Network panel earlier in the session).
- An obfuscated derivation. If so, set logpoints inside the derivation and read the output.

Modify the code temporarily via Overrides to log the secret on each call:

```js
console.log('sign called', { input, secret })
const signature = sign(input, secret)
```

Save as override, reload, perform the action, read the secret in the Console.

**e. Replicate in Node.**

```ts
import { createHmac } from 'node:crypto'

const sign = (input: string, secret: string): string =>
  createHmac('sha256', secret).update(input).digest('hex')

const callApi = async (body: unknown, secret: string) => {
  const timestamp = Date.now().toString()
  const nonce = crypto.randomUUID()
  const payload = JSON.stringify(body)
  const signature = sign(timestamp + nonce + payload, secret)

  const res = await fetch('https://example.com/api/v2/orders', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Timestamp': timestamp,
      'X-Nonce': nonce,
      'X-Signature': signature,
    },
    body: payload,
  })
  return res.json()
}
```

Send it. If the server rejects with `401`, you got the algorithm or input order slightly wrong — go back, set a logpoint at the exact `sign` call, log the *exact bytes* being signed, and diff against your Node version.

This is the loop. Every signed/encrypted request reduces to: find it, freeze it, walk it, replicate it.

---

## 14. The Initiator column and call-stack tracing

In the Network panel, right-click the column header bar → enable **Initiator**. The column shows the file and line that triggered each request. Click any cell to open the **Initiator** tab on the right with the full call stack at the moment the request was created.

This is extremely useful for the bulk pattern of "I want to know what code fired this request" without setting an XHR breakpoint. For one-off forensics, the Initiator column tells you in one click. For repeated debugging, the XHR breakpoint is more flexible because it freezes the runtime and lets you inspect locals.

A single click in the Initiator stack opens the corresponding Sources view at that line. The chain leads back through wrappers (axios → an interceptor → your code) — Ignore List filters the framework frames so you only see your target's code.

---

## 15. Source maps in DevTools (quick use)

If a `.js` file declares `//# sourceMappingURL=...`, DevTools automatically loads the source map and presents the *original* source (TypeScript, JSX, etc.) in the Sources panel. Breakpoints set in the original source are translated to the deployed JS at runtime.

Quick checks:

- **Settings → Preferences → Sources** must have **Enable JavaScript source maps** checked. (Default on.)
- A file with a successfully loaded source map shows with a green badge in the Sources tree.
- Right-click a `.js` file → **Open original** (when a map is detected).

If the source map URL is `localhost:3000/main.js.map`, it's a leaked dev-build map — common in production by mistake. Treat it as a gift. Full coverage of source map extraction, chunked-bundle reassembly, and using `unsmap`/`source-map` programmatically is in [doc 12](12-source-maps-and-chunked-bundles.md).

---

## 16. Shadow DOM, iframes, web workers, service workers

### 16.1 Shadow DOM

Elements panel shows shadow roots inline as `#shadow-root (open)` or `#shadow-root (closed)`. Open shadow roots are inspectable. Closed shadow roots are not directly inspectable from the Elements panel — but you can sometimes reach them via JS at the parent's `attachShadow` call by intercepting in Sources. For most public sites this isn't a barrier; closed shadow DOM is rare on pages that also serve scrapeable content.

The Console respects shadow DOM boundaries — `document.querySelector` does not pierce them. Use `host.shadowRoot.querySelector(...)` from within the host element.

### 16.2 Iframes

The execution context selector at the top of the Console (a dropdown labeled `top` by default) lets you switch into the context of any iframe. Change it before running expressions, otherwise `window` refers to the top frame.

Network requests from iframes appear in the same Network panel as the parent. The iframe's source URL appears as a row at the top.

### 16.3 Web workers and shared workers

Sources panel left pane includes **Threads** when workers are active. Click a worker to open its scope as if it were a separate page. Set breakpoints, inspect scope, all normal.

Workers have their own Console context — the dropdown in the Console lets you target a specific worker.

### 16.4 Service workers

Application panel → **Service Workers**:
- **Update on reload** — forces SW update on every reload (turn on while debugging).
- **Bypass for network** — skip the SW entirely (turn on when you suspect it's intercepting).
- **Inspect** link next to a registered SW opens DevTools attached to the SW thread.

The Network panel marks SW-served responses with a gear icon in the **Status** or **Type** area. If you see `(ServiceWorker)` in the size column, the SW served a cached version, not the network.

---

## 17. When DevTools is too noisy — handing off to mitmproxy / Burp

DevTools shows what *Chrome* sees. It does not show:

- TLS-decrypted traffic from a native app or mobile app.
- Traffic from a non-Chromium browser without setting up DevTools there.
- Long-running unattended captures across hours where DevTools' in-memory log would balloon.
- Traffic outside the page lifecycle (e.g., a background extension's network).

For these, drop down to a system-level proxy:

- **mitmproxy** — Python, scriptable, terminal UI. Run as `mitmproxy -p 8080`, point Chrome at it (`--proxy-server=http://localhost:8080`), trust the mitmproxy CA. Scripts in `mitmproxy.io/addons` can rewrite, log, or replay.
- **Burp Suite Community / Pro** — Java, GUI, the de facto pen-test proxy. Repeater, Intruder, and Logger are scraping-relevant.
- **Charles Proxy** — macOS/Windows, polished UI, paid.
- **Proxyman** — macOS native, designer-friendly UI.

Workflow: capture with DevTools first (faster, in-context). When the request is complex enough that you want a programmable proxy (e.g., you want to record 10,000 requests, or rewrite headers on the fly), move to mitmproxy. Full setup, certificate trust, scripts, and replay automation are in [doc 13](13-api-reverse-engineering.md).

---

## Related

- [Headless Browsers](../extraction/07-headless-browsers.md)
- [Bot Detection Internals](../extraction/09-bot-detection-internals.md)
- [Source Maps and Chunked Bundles](12-source-maps-and-chunked-bundles.md)
- [API Reverse-Engineering](13-api-reverse-engineering.md)
- [Production Scraping Hygiene and Legal](14-production-scraping-hygiene-and-legal.md)

## References

- Chrome DevTools documentation — https://developer.chrome.com/docs/devtools/
- Chrome DevTools Protocol — https://chromedevtools.github.io/devtools-protocol/
- DevTools "What's New" — https://developer.chrome.com/blog/new-in-devtools-XXX (replace XXX with the Chrome version, e.g., `120`, `121`, `122`)
- Network panel reference — https://developer.chrome.com/docs/devtools/network
- Network features reference — https://developer.chrome.com/docs/devtools/network/reference
- Sources panel reference — https://developer.chrome.com/docs/devtools/sources
- JavaScript breakpoints — https://developer.chrome.com/docs/devtools/javascript/breakpoints
- Ignore List — https://developer.chrome.com/docs/devtools/javascript/ignore-list
- Local Overrides — https://developer.chrome.com/docs/devtools/overrides
- Workspaces — https://developer.chrome.com/docs/devtools/workspaces
- Recorder panel — https://developer.chrome.com/docs/devtools/recorder
- Performance panel — https://developer.chrome.com/docs/devtools/performance
- Memory panel — https://developer.chrome.com/docs/devtools/memory-problems
- Application panel — https://developer.chrome.com/docs/devtools/application
- Console reference — https://developer.chrome.com/docs/devtools/console/reference
- Console utilities API — https://developer.chrome.com/docs/devtools/console/utilities
- HAR 1.2 specification — Jan Odvarko, http://www.softwareishard.com/blog/har-12-spec/
- W3C Web Performance HAR archive — https://w3c.github.io/web-performance/specs/HAR/Overview.html
- chrome-har-capturer — https://github.com/cyrus-and/chrome-har-capturer
- Mathias Bynens (Chrome DevRel) DevTools posts — https://mathiasbynens.be/notes
- Addy Osmani DevTools posts — https://addyosmani.com/blog/
- Sofia Emelianova / Crystal Larssen (Chrome DevTools team) writeups — published on https://developer.chrome.com/blog/ under their bylines
- Puppeteer Recorder export documentation — https://pptr.dev/guides/chrome-extension/
- mitmproxy — https://mitmproxy.org/
- Burp Suite — https://portswigger.net/burp
