---
title: "Scraping Pitfalls — Bug Spotting"
date: 2026-05-06
updated: 2026-05-06
tags: [bug-spotting, scraping, http, async, robots-txt, sitemap, ja3, captcha]
---

# Scraping Pitfalls — Bug Spotting

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `bug-spotting` `scraping` `http` `async` `robots-txt` `sitemap` `ja3` `captcha`

---

## How to Use

Each numbered bug below has broken code. Try to spot the issue before clicking the hint. Once you've formed a hypothesis, jump to the matching numbered entry in `## Solutions` for the root cause and the fix.

Bugs are organized by difficulty:
- **Easy** — the bug is in plain sight; common beginner mistakes
- **Subtle** — looks correct on first read; reveals on careful inspection
- **Senior trap** — looks correct even after careful review; requires specific domain knowledge to spot

Every bug cites a real reference (RFC, library issue, postmortem, CVE).

---

## Bugs

### Easy

#### Bug 1 — Per-request session

```python
import requests

URLS = ["https://shop.example.com/login", "https://shop.example.com/orders"]

def scrape(creds):
    requests.post(URLS[0], data=creds)
    return requests.get(URLS[1]).text  # expects logged-in HTML
```

<details><summary>Hint</summary>Where does the cookie jar live across these two calls?</details>

#### Bug 2 — Forgotten await

```python
import asyncio
import httpx

async def fetch_all(urls):
    async with httpx.AsyncClient() as client:
        results = []
        for u in urls:
            results.append(client.get(u))   # not awaited
        return results
```

<details><summary>Hint</summary>What type are the items in `results`?</details>

#### Bug 3 — Unbounded gather

```python
async def crawl(urls):
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(*(client.get(u) for u in urls))
```

<details><summary>Hint</summary>Imagine `urls` has 200,000 entries.</details>

#### Bug 4 — Default infinite timeout

```python
import requests

resp = requests.get("https://flaky.example.com/api")
print(resp.json())
```

<details><summary>Hint</summary>What does `requests` do if the server accepts the connection but never sends bytes?</details>

#### Bug 5 — Brotli claimed but not installed

```python
import requests

headers = {
    "User-Agent": "Mozilla/5.0",
    "Accept-Encoding": "gzip, deflate, br",
}
resp = requests.get("https://example.com", headers=headers)
print(resp.text)  # garbled bytes
```

<details><summary>Hint</summary>What decoder ships with stock `urllib3` for `br`?</details>

#### Bug 6 — Naive exponential backoff

```python
import time, random

def fetch_with_retry(url, max_attempts=8):
    for attempt in range(max_attempts):
        try:
            return requests.get(url).json()
        except Exception:
            time.sleep(2 ** attempt)
    raise RuntimeError("gave up")
```

<details><summary>Hint</summary>Now imagine a thousand workers running this same loop after a shared 503.</details>

#### Bug 7 — DOM scrape on stale element

```javascript
const handle = await page.$("a.next");
await handle.click();
const text = await handle.innerText();   // grab title after navigation
```

<details><summary>Hint</summary>What happens to the JS handle after the page navigates?</details>

#### Bug 8 — Headless Chrome `cdc_*` leak

```python
from selenium import webdriver

driver = webdriver.Chrome()  # vanilla driver
driver.get("https://target.example.com")
```

<details><summary>Hint</summary>Inspect `window` in DevTools.</details>

---

### Subtle

#### Bug 9 — Cross-host redirect leaking auth

```python
import requests

resp = requests.get(
    "https://api.example.com/private",
    headers={"Authorization": "Bearer secret123"},
    allow_redirects=True,
)
```

<details><summary>Hint</summary>What if `api.example.com` 302s to `evil.example.net`?</details>

#### Bug 10 — robots.txt longest-match

```python
# robots.txt:
#   User-agent: *
#   Disallow: /
#   Allow: /public/

import urllib.robotparser

rp = urllib.robotparser.RobotFileParser()
rp.set_url("https://example.com/robots.txt")
rp.read()
print(rp.can_fetch("MyBot", "https://example.com/public/page"))  # ?
```

<details><summary>Hint</summary>Read RFC 9309 §2.2.2 on rule precedence.</details>

#### Bug 11 — Sitemap index loop

```python
def crawl_sitemap(url, seen=set()):
    if url in seen:
        return
    seen.add(url)
    xml = requests.get(url).text
    for child in re.findall(r"<loc>(.*?)</loc>", xml):
        if child.endswith(".xml"):
            crawl_sitemap(child, seen)   # recurses forever on broken sites
        else:
            enqueue(child)
```

<details><summary>Hint</summary>Mutable default argument is one bug; the recursion has another.</details>

#### Bug 12 — Same-IP retry on 429

```python
def fetch(url, proxies):
    proxy = random.choice(proxies)
    while True:
        r = requests.get(url, proxies={"https": proxy})
        if r.status_code != 429:
            return r
        time.sleep(2)   # retry through same proxy
```

<details><summary>Hint</summary>What is the upstream service signalling with 429, and to whom?</details>

#### Bug 13 — Reused CAPTCHA token

```python
def submit(form_data):
    token = get_recaptcha_token()  # solved once at startup
    return requests.post(URL, data={**form_data, "g-recaptcha-response": token})
```

<details><summary>Hint</summary>How long does Google say a reCAPTCHA token is valid?</details>

#### Bug 14 — Decompress twice

```python
resp = requests.get(url, headers={"Accept-Encoding": "gzip"}, stream=True)
raw = resp.raw.read()                 # what level is this at?
text = gzip.decompress(raw).decode()  # we decompress
```

<details><summary>Hint</summary>What does `requests` do with `Content-Encoding: gzip` by the time `resp.raw.read()` returns?</details>

#### Bug 15 — Bloom filter dedup acceptance

```python
def visit(url):
    if url in bloom:
        return                # treat as already visited
    bloom.add(url)
    enqueue(url)
```

<details><summary>Hint</summary>What is a Bloom filter allowed to lie about, and in which direction?</details>

#### Bug 16 — Date.parse ambiguity

```javascript
function isFresh(item) {
    const d = Date.parse(item.published);   // "03/04/2025" from a US source
    return Date.now() - d < 24 * 3600 * 1000;
}
```

<details><summary>Hint</summary>Is that March 4 or April 3?</details>

#### Bug 17 — Token-bucket race

```python
class RateLimiter:
    def __init__(self, rate):
        self.tokens = rate
    def try_take(self):
        if self.tokens > 0:
            self.tokens -= 1
            return True
        return False
```

<details><summary>Hint</summary>Two coroutines call `try_take()` simultaneously.</details>

#### Bug 18 — Cookie jar pickled

```python
import pickle, requests

s = requests.Session()
# ... login ...
with open("cookies.pkl", "wb") as f:
    pickle.dump(s.cookies, f)
```

<details><summary>Hint</summary>What happens after a Python minor-version bump or a `requests` upgrade?</details>

#### Bug 19 — Connection pool exhaustion

```python
async def worker(url):
    async with httpx.AsyncClient() as client:   # new client per task
        return await client.get(url)

await asyncio.gather(*(worker(u) for u in 100_000_urls))
```

<details><summary>Hint</summary>What does each `AsyncClient()` instantiate, and how is it bounded?</details>

#### Bug 20 — No max-redirect cap

```go
client := &http.Client{
    CheckRedirect: nil,           // use default
    Timeout:       30 * time.Second,
}
resp, err := client.Get(url)
```

<details><summary>Hint</summary>The default isn't infinite, but check what `CheckRedirect: nil` actually means in Go's `net/http`.</details>

#### Bug 21 — TZ-naïve ISO compare

```python
from datetime import datetime

def newer_than(record_iso, cutoff_iso):
    return datetime.fromisoformat(record_iso) > datetime.fromisoformat(cutoff_iso)

newer_than("2026-05-06T12:00:00", "2026-05-06T10:00:00+00:00")
```

<details><summary>Hint</summary>One side has tzinfo, the other doesn't.</details>

#### Bug 22 — Content-hash with timestamp

```python
def fingerprint(html):
    return hashlib.sha256(html.encode()).hexdigest()
```

<details><summary>Hint</summary>What does the page footer usually contain?</details>

#### Bug 23 — URL canonicalization

```python
def canonical(u):
    p = urlparse(u)
    return f"{p.scheme}://{p.netloc}{p.path}".lower()    # lowercases everything
```

<details><summary>Hint</summary>RFC 3986 §6.2.2: which URL components are case-insensitive?</details>

#### Bug 24 — SSE without resume

```javascript
const es = new EventSource("/events");
es.onmessage = (e) => process(JSON.parse(e.data));
```

<details><summary>Hint</summary>Server restarts at 02:00 UTC. What about events emitted between disconnect and reconnect?</details>

---

### Senior trap

#### Bug 25 — JA3/JA4 fingerprint mismatch

```python
import requests

headers = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/124.0.0.0 Safari/537.36"
    ),
    "sec-ch-ua": '"Chromium";v="124", "Not.A/Brand";v="24"',
}
r = requests.get("https://protected.example.com", headers=headers)
```

<details><summary>Hint</summary>The header layer says Chrome. What does the TLS ClientHello say?</details>

#### Bug 26 — H/2 SETTINGS frame fingerprint

```python
import httpx

# httpx with http2=True — looks like a browser request
with httpx.Client(http2=True, headers=BROWSER_HEADERS) as c:
    r = c.get("https://akamai-protected.example.com")
```

<details><summary>Hint</summary>HTTP/2 fingerprinting goes beyond TLS.</details>

#### Bug 27 — Pinned cert behind TLS-terminating proxy

```python
import httpx, ssl

ctx = ssl.create_default_context(cafile="origin-cert.pem")
ctx.check_hostname = True
client = httpx.Client(verify=ctx, proxy="http://corp-proxy:8080")
client.get("https://example.com")  # fails: cert mismatch
```

<details><summary>Hint</summary>What cert does the proxy actually present to your client?</details>

#### Bug 28 — GraphQL alias amplification

```python
query = "{ " + " ".join(f"a{i}: user(id: {i}) {{ posts {{ comments {{ author {{ name }} }} }} }}"
                       for i in range(100)) + " }"
requests.post(GRAPHQL, json={"query": query})
```

<details><summary>Hint</summary>You think you sent one query. What did the backend just resolve?</details>

#### Bug 29 — WebSocket idle death

```javascript
const ws = new WebSocket(url);
ws.onmessage = (m) => handle(JSON.parse(m.data));
// never sends a ping
```

<details><summary>Hint</summary>Every load balancer between you and the origin has an idle-connection timeout.</details>

#### Bug 30 — Source-map leak from your scraper

```javascript
// scraper bundle built with default webpack config
// production deploy ships dist/scraper.js + dist/scraper.js.map
```

<details><summary>Hint</summary>What does `//# sourceMappingURL=` reveal to a WAF that scrapes back?</details>

#### Bug 31 — gzip-bomb defense missing

```python
resp = requests.get(url, stream=True)
data = resp.raw.read(decode_content=True)   # auto-inflates
process(data)
```

<details><summary>Hint</summary>1 KB of zeros gzipped.</details>

#### Bug 32 — H/2 lowercase header requirement

```python
import httpx
client = httpx.Client(http2=True)
r = client.get(url, headers={"X-Custom-Header": "value", "Content-Length": "0"})
```

<details><summary>Hint</summary>Two issues, both rooted in RFC 9113.</details>

---

## Solutions

### Bug 1 — Per-request session

**Root cause:** `requests.get` and `requests.post` at module level each create a new transient `Session` internally, and that session is discarded when the call returns. The `Set-Cookie` from the login response never survives to the next call. The second request goes out as anonymous.

**Fix:**

```python
import requests

with requests.Session() as s:
    s.post(URLS[0], data=creds)
    html = s.get(URLS[1]).text
```

**Reference:** [`requests` advanced usage — Session objects](https://requests.readthedocs.io/en/latest/user/advanced/#session-objects).

### Bug 2 — Forgotten await

**Root cause:** `client.get(u)` returns a coroutine object. The list contains 200 unawaited coroutines. Python emits `RuntimeWarning: coroutine '...' was never awaited` and you get back a list of `<coroutine>` objects, not responses. Worse, no HTTP traffic actually happens.

**Fix:**

```python
import asyncio, httpx

async def fetch_all(urls):
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(*(client.get(u) for u in urls))
```

**Reference:** [PEP 492 — Coroutines with async/await syntax](https://peps.python.org/pep-0492/).

### Bug 3 — Unbounded gather

**Root cause:** `asyncio.gather` schedules every coroutine immediately. With 200k URLs you spawn 200k pending HTTP tasks, exhaust the event loop's connection pool, blow past file-descriptor limits, and OOM. `gather` is fine for small fan-out; for crawl-scale you need a bounded worker pool or a semaphore.

**Fix:**

```python
async def crawl(urls, concurrency=64):
    sem = asyncio.Semaphore(concurrency)
    async with httpx.AsyncClient() as client:
        async def one(u):
            async with sem:
                return await client.get(u)
        return await asyncio.gather(*(one(u) for u in urls))
```

**Reference:** [`httpx` advanced — async-client pool limits](https://www.python-httpx.org/advanced/resource-limits/) and [`asyncio.Semaphore`](https://docs.python.org/3/library/asyncio-sync.html#semaphore).

### Bug 4 — Default infinite timeout

**Root cause:** `requests.get` has no default timeout. A misbehaving server that accepts the connection but never sends bytes will leave your worker hung forever. `httpx` is the same: passing `timeout=None` (or relying on it implicitly when overriding) means infinite wait.

**Fix:**

```python
resp = requests.get("https://flaky.example.com/api", timeout=(5, 30))
```

The tuple is `(connect_timeout, read_timeout)`. For `httpx` use `httpx.Timeout(5.0, connect=2.0)`.

**Reference:** [`requests` — Timeouts](https://requests.readthedocs.io/en/latest/user/quickstart/#timeouts) (the docs say explicitly: "Nearly all production code should use this parameter in nearly all requests").

### Bug 5 — Brotli claimed but not installed

**Root cause:** You advertised `br` but `urllib3` only ships gzip and deflate by default. The server returns Brotli-encoded bytes; `urllib3` can't decode them, so `resp.text` decodes raw Brotli payload as UTF-8 and gives you garbage. Either install `brotli` (`pip install brotli`) or remove `br` from `Accept-Encoding`.

**Fix:**

```python
# Either:
# pip install brotli   (or 'brotlicffi' on PyPy)
# or:
headers["Accept-Encoding"] = "gzip, deflate"
```

**Reference:** [`urllib3` brotli/zstd support changelog](https://urllib3.readthedocs.io/en/stable/changelog.html) — Brotli is an optional extra (`urllib3[brotli]`).

### Bug 6 — Naive exponential backoff

**Root cause:** Without jitter, all clients that hit the same 503 wake up in lockstep and retry at the same instant. The thundering herd defeats backoff entirely. AWS's classic write-up shows full jitter (`sleep = random.uniform(0, base * 2**n)`) consistently outperforms plain exponential.

**Fix:**

```python
import random, time

def fetch_with_retry(url, max_attempts=8, base=0.5, cap=30.0):
    for attempt in range(max_attempts):
        try:
            return requests.get(url, timeout=(5, 30)).json()
        except Exception:
            sleep = random.uniform(0, min(cap, base * (2 ** attempt)))
            time.sleep(sleep)
    raise RuntimeError("gave up")
```

**Reference:** [AWS Architecture Blog — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/).

### Bug 7 — DOM scrape on stale element

**Root cause:** Once the page navigates, every `ElementHandle` from the previous document is detached. `handle.innerText()` raises `Error: Element is not attached to the DOM`. Re-query after navigation, or use `Promise.all([waitForNavigation, click])` semantics.

**Fix:**

```javascript
await Promise.all([
  page.waitForNavigation({ waitUntil: "domcontentloaded" }),
  page.click("a.next"),
]);
const text = await page.locator("h1").innerText();
```

In Playwright, prefer `Locator` over `ElementHandle` — locators auto-retry against the live DOM.

**Reference:** [Playwright — Locators vs ElementHandles](https://playwright.dev/docs/locators#locators-vs-elementhandles).

### Bug 8 — Headless Chrome `cdc_*` leak

**Root cause:** Stock ChromeDriver injects runtime properties whose names start with `cdc_` (visible via `Object.getOwnPropertyNames(window)`). Bot-detection scripts look for any key matching `/^cdc_/`. `undetected-chromedriver` patches the driver binary to randomize those names; vanilla Selenium does not.

**Fix:**

```python
import undetected_chromedriver as uc
driver = uc.Chrome(headless=True, use_subprocess=True)
```

**Reference:** [`undetected-chromedriver` README — "rename cdc\_ properties"](https://github.com/ultrafunkamsterdam/undetected-chromedriver) and the related ChromeDriver source: `cdc_adoQpoasnfa76pfcZLmcfl_*`.

### Bug 9 — Cross-host redirect leaking auth

**Root cause:** Older `requests` would forward `Authorization` and `Cookie` headers across host boundaries. After security fix in `requests` 2.32 the library now strips `Authorization` on cross-host redirect — but only if you let `requests` build the redirected request. If you set the header on a `Session` via `s.headers["Authorization"]`, it is reapplied to every request including the redirect target. Strip per-request, not on the session, when the destination might 30x.

**Fix:**

```python
resp = requests.get(
    "https://api.example.com/private",
    headers={"Authorization": "Bearer secret123"},
    allow_redirects=False,
)
if resp.is_redirect:
    target = resp.headers["location"]
    if urlparse(target).netloc == "api.example.com":
        resp = requests.get(target, headers={"Authorization": "Bearer secret123"})
```

**Reference:** [`requests` issue #6020 / CVE-2024-35195 — `Session` verifies once](https://github.com/psf/requests/security/advisories/GHSA-9wx4-h78v-vm56) and the long-standing [`requests` issue #4848 on header rebinding](https://github.com/psf/requests/issues/4848).

### Bug 10 — robots.txt longest-match

**Root cause:** RFC 9309 §2.2.2 says the most specific (longest match by character count of the path pattern) rule wins, with `Allow` beating `Disallow` only on ties. `Disallow: /` is one character; `Allow: /public/` is eight. The `Allow` wins, so `/public/page` is fetchable. Many old robots parsers (including older versions of Python's stdlib parser) implemented "first match wins" instead and would block `/public/`.

**Fix:** Use a parser that implements RFC 9309 longest-match. Python's `urllib.robotparser` was updated, but for production crawlers prefer Google's open-source [`robotstxt`](https://github.com/google/robotstxt) (the reference implementation that the RFC was built from), or its Python binding `reppy`/`protego`.

```python
from protego import Protego

rp = Protego.parse(robots_text)
rp.can_fetch("https://example.com/public/page", "MyBot")  # True
```

**Reference:** [RFC 9309 — Robots Exclusion Protocol §2.2.2](https://www.rfc-editor.org/rfc/rfc9309.html#name-the-allow-and-disallow-line).

### Bug 11 — Sitemap index loop

**Root cause:** Two issues. First, the mutable default `seen=set()` is shared across calls — a classic Python gotcha. Second, even with that fixed, real sites occasionally publish a sitemap-index whose `<loc>` references its own URL or another index that loops back. The recursion has no depth cap, no per-host limit, and trusts the remote XML to be acyclic.

**Fix:**

```python
def crawl_sitemap(root_url, max_depth=4, max_indexes=50):
    seen = set()
    queue = [(root_url, 0)]
    while queue:
        url, depth = queue.pop()
        if url in seen or depth > max_depth or len(seen) > max_indexes:
            continue
        seen.add(url)
        xml = requests.get(url, timeout=(5, 30)).text
        for child in re.findall(r"<loc>(.*?)</loc>", xml):
            if child.endswith(".xml"):
                queue.append((child, depth + 1))
            else:
                enqueue(child)
```

**Reference:** [sitemaps.org protocol — sitemap index files](https://www.sitemaps.org/protocol.html#index) and [Common Python gotchas — mutable default arguments](https://docs.python-guide.org/writing/gotchas/#mutable-default-arguments).

### Bug 12 — Same-IP retry on 429

**Root cause:** A `429 Too Many Requests` from upstream is keyed on whichever identity the upstream chose to track — usually the source IP (or proxy IP). Retrying through the same proxy hits the same bucket and you stay rate-limited indefinitely. Worse, you may earn an extended ban for ignoring the signal. Rotate the proxy on 429.

**Fix:**

```python
def fetch(url, proxy_pool, max_attempts=5):
    proxies = list(proxy_pool)
    random.shuffle(proxies)
    for proxy in proxies[:max_attempts]:
        r = requests.get(url, proxies={"https": proxy}, timeout=(5, 30))
        if r.status_code == 429:
            retry_after = int(r.headers.get("Retry-After", "0"))
            time.sleep(retry_after)
            continue
        return r
    raise RuntimeError("all proxies rate-limited")
```

**Reference:** [RFC 6585 §4 — 429 status](https://www.rfc-editor.org/rfc/rfc6585#section-4) and the `Retry-After` semantics in [RFC 9110 §10.2.3](https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after).

### Bug 13 — Reused CAPTCHA token

**Root cause:** reCAPTCHA v2 and v3 tokens are valid for two minutes and are single-use on `siteverify`. Reusing a token returns `"error-codes": ["timeout-or-duplicate"]` and the request silently fails authorization. Solve once per submission, not once per process. Also: if two threads request a token in the same window from a captcha-solving service, you can race and submit one token twice before the API returns the second.

**Fix:**

```python
def submit(form_data):
    token = get_recaptcha_token()  # fresh per submission
    return requests.post(URL, data={**form_data, "g-recaptcha-response": token})
```

For solver services, request the token inside the critical section that submits, and serialize per-form.

**Reference:** [Google reCAPTCHA v3 docs — "tokens are valid for 2 minutes"](https://developers.google.com/recaptcha/docs/v3) (Verifying the user's response section).

### Bug 14 — Decompress twice

**Root cause:** `requests` decodes `Content-Encoding: gzip` at the `urllib3` layer. By the time `resp.content` or `resp.text` returns, the body is already plaintext. `resp.raw.read()` is normally pre-decode, but the gotcha is that `stream=True` plus `resp.raw.read()` returns the still-compressed bytes, while `resp.content` does the decode for you. Mixing them, or calling `gzip.decompress` on `resp.content`, double-decodes.

**Fix:**

```python
resp = requests.get(url, headers={"Accept-Encoding": "gzip"})
text = resp.text   # already decoded
```

If you genuinely need the raw bytes (e.g. to forward), pass `stream=True` and read with `decode_content=False`:

```python
resp = requests.get(url, stream=True)
raw = resp.raw.read(decode_content=False)
```

**Reference:** [`requests` advanced — body content workflow](https://requests.readthedocs.io/en/latest/user/advanced/#body-content-workflow) and the `urllib3` `decode_content` flag in [`HTTPResponse.read`](https://urllib3.readthedocs.io/en/stable/reference/urllib3.response.html#urllib3.response.HTTPResponse.read).

### Bug 15 — Bloom filter dedup

**Root cause:** A Bloom filter has false positives but never false negatives. The check `url in bloom` can return `True` for a URL that has never been added — and your code then skips a real, never-visited URL. For dedup of the URL frontier this means missing pages. Use a Bloom filter as a fast "probably seen" prefilter and confirm against an authoritative store on hit, or use a cuckoo filter / exact set when correctness matters more than memory.

**Fix:**

```python
def visit(url):
    if url in bloom and url in authoritative_seen:   # confirm on hit
        return
    bloom.add(url)
    authoritative_seen.add(url)
    enqueue(url)
```

**Reference:** [Burton Bloom — "Space/Time Trade-offs in Hash Coding with Allowable Errors"](https://dl.acm.org/doi/10.1145/362686.362692) (CACM 1970), the original paper that defines the false-positive guarantee.

### Bug 16 — Date.parse ambiguity

**Root cause:** ECMAScript only mandates support for the simplified ISO 8601 format. Other strings (`"03/04/2025"`, `"Mar 04 2025"`, etc.) are implementation-defined. V8 reads `MM/DD/YYYY` as US, SpiderMonkey may differ, and the same code parses to different days across runtimes. Never feed `Date.parse` non-ISO input.

**Fix:**

```javascript
const d = Temporal.PlainDate.from(item.published);   // when available
// or use a strict parser like luxon:
const d = DateTime.fromFormat(item.published, "MM/dd/yyyy", { zone: "America/New_York" });
```

**Reference:** [MDN — `Date.parse()` "non-standard string parsing is implementation-defined"](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/parse) and [ECMA-262 §21.4.3.2](https://tc39.es/ecma262/#sec-date.parse).

### Bug 17 — Token-bucket race

**Root cause:** In a single-threaded asyncio program the read-modify-write of `self.tokens` is atomic across `await` boundaries — but `try_take` has no awaits, so it is atomic *here*. The bug appears the moment you move to threads, multiple processes, or subdivide via `asyncio.to_thread`. A correct token bucket either uses an `asyncio.Lock` (or `threading.Lock`), or relies on an atomic primitive (Redis `INCR`, `Semaphore`).

**Fix:**

```python
class RateLimiter:
    def __init__(self, rate):
        self.tokens = rate
        self._lock = asyncio.Lock()
    async def try_take(self):
        async with self._lock:
            if self.tokens > 0:
                self.tokens -= 1
                return True
            return False
```

**Reference:** [`asyncio` synchronization primitives docs](https://docs.python.org/3/library/asyncio-sync.html) and the classic [Lamport — "Time, Clocks, and the Ordering of Events"](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) on read-modify-write hazards.

### Bug 18 — Cookie jar pickled

**Root cause:** `pickle` serializes Python class identity. After upgrading `requests` (where `RequestsCookieJar` may relocate or change `__slots__`) or Python (where `cookiejar.Cookie` evolved), the pickle file unpickles to a broken object or fails outright. Persist cookies as JSON or Mozilla-format cookies.txt — both are stable schemas.

**Fix:**

```python
# write
with open("cookies.json", "w") as f:
    json.dump([{"name": c.name, "value": c.value, "domain": c.domain, "path": c.path}
               for c in s.cookies], f)

# or use http.cookiejar.MozillaCookieJar which has a documented on-disk format
```

**Reference:** [Python `pickle` docs — "warning: never unpickle untrusted, also class layout sensitive"](https://docs.python.org/3/library/pickle.html#what-can-be-pickled-and-unpickled) and [`http.cookiejar.MozillaCookieJar`](https://docs.python.org/3/library/http.cookiejar.html#http.cookiejar.MozillaCookieJar).

### Bug 19 — Connection pool exhaustion

**Root cause:** Each `httpx.AsyncClient()` opens its own connection pool, DNS cache, and TLS context. Per-task instantiation means N pools for N tasks, and the underlying file descriptors leak until GC. Share one client across the crawl and bound concurrency externally.

**Fix:**

```python
async def crawl(urls, concurrency=64):
    sem = asyncio.Semaphore(concurrency)
    limits = httpx.Limits(max_connections=200, max_keepalive_connections=50)
    async with httpx.AsyncClient(limits=limits) as client:
        async def one(u):
            async with sem:
                return await client.get(u)
        return await asyncio.gather(*(one(u) for u in urls))
```

**Reference:** [`httpx` advanced — resource limits and pool reuse](https://www.python-httpx.org/advanced/resource-limits/).

### Bug 20 — No max-redirect cap

**Root cause:** `CheckRedirect: nil` does not mean "follow infinitely" in Go; it means "use the default policy" which caps at 10 redirects (`net/http.defaultCheckRedirect`). That is fine for most cases but easy to misread. The actual trap is that a misbehaving server can send 10 redirects each with a `Set-Cookie` that grows the header list past your size limit, or that you might want a tighter cap for crawl politeness.

**Fix:**

```go
client := &http.Client{
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        if len(via) >= 5 {
            return http.ErrUseLastResponse
        }
        return nil
    },
    Timeout: 30 * time.Second,
}
```

**Reference:** [Go `net/http` — `Client.CheckRedirect` field docs](https://pkg.go.dev/net/http#Client) and [the `defaultCheckRedirect` source](https://github.com/golang/go/blob/master/src/net/http/client.go).

### Bug 21 — TZ-naïve ISO compare

**Root cause:** `datetime.fromisoformat("2026-05-06T12:00:00")` returns a tz-naïve datetime; the second call returns tz-aware. Comparing the two raises `TypeError: can't compare offset-naive and offset-aware datetimes`. If your code uses `<` against `datetime.utcnow()` (also naïve), the comparison silently uses local-machine time, and you ship code that drifts across DST changes and across deployments in different regions.

**Fix:**

```python
from datetime import datetime, timezone

def parse(s):
    d = datetime.fromisoformat(s)
    if d.tzinfo is None:
        d = d.replace(tzinfo=timezone.utc)   # explicit decision
    return d

def newer_than(record_iso, cutoff_iso):
    return parse(record_iso) > parse(cutoff_iso)
```

Treat tz-naïve input as a parse error in new code; never use `datetime.utcnow()` (deprecated in 3.12) — use `datetime.now(timezone.utc)`.

**Reference:** [Python `datetime` docs — naïve vs aware](https://docs.python.org/3/library/datetime.html#aware-and-naive-objects) and [PEP 615 — IANA tz support via `zoneinfo`](https://peps.python.org/pep-0615/).

### Bug 22 — Content-hash with timestamp

**Root cause:** Many pages render `<footer>Generated at 2026-05-06T12:34:56Z</footer>` or include CSRF tokens, ad-slot IDs, and request-ID echo. Hashing the raw HTML never matches across fetches even when the meaningful content is identical. Normalize before hashing: strip script/style, collapse whitespace, remove known-volatile fragments, then hash. Better: hash the extracted record, not the page.

**Fix:**

```python
from selectolax.parser import HTMLParser

VOLATILE_SELECTORS = ["script", "style", ".ad", ".csrf-token", "footer .timestamp"]

def fingerprint(html):
    tree = HTMLParser(html)
    for sel in VOLATILE_SELECTORS:
        for node in tree.css(sel):
            node.decompose()
    text = " ".join(tree.text().split())
    return hashlib.sha256(text.encode()).hexdigest()
```

**Reference:** [Common Crawl's content-deduplication approach](https://commoncrawl.org/blog/march-2024-crawl-archive-now-available) and [Manku, Jain, Sarma — "Detecting Near-Duplicates for Web Crawling" (WWW 2007)](https://www2007.cpsc.ucalgary.ca/papers/paper215.pdf) (simhash for near-dup, but also describes normalization).

### Bug 23 — URL canonicalization

**Root cause:** RFC 3986 §6.2.2 says scheme and host are case-insensitive; the path is case-sensitive (subject to scheme-specific rules). Lowercasing `/Public/Page` to `/public/page` may point to a different resource on case-sensitive servers (most non-Windows hosts). Lowercase scheme and host only.

**Fix:**

```python
from urllib.parse import urlparse, urlunparse

def canonical(u):
    p = urlparse(u)
    return urlunparse((p.scheme.lower(), p.netloc.lower(), p.path, p.params, p.query, ""))
```

For full canonicalization (percent-encoding normalization, default-port stripping, dot-segment removal) follow RFC 3986 §6.2 syntax-based normalization.

**Reference:** [RFC 3986 §6.2.2.1 — Case Normalization](https://www.rfc-editor.org/rfc/rfc3986#section-6.2.2.1).

### Bug 24 — SSE without resume

**Root cause:** EventSource auto-reconnects, but the server only knows what to replay if you send `Last-Event-ID`. Browsers do this automatically; ad-hoc client code often does not. Without it, the server resumes from "now" and any events emitted between disconnect and reconnect are lost forever.

**Fix:** Use a real EventSource implementation that honors `Last-Event-ID`, or write the reconnect logic explicitly:

```javascript
let lastId = null;
function connect() {
  const es = new EventSource("/events" + (lastId ? `?last=${lastId}` : ""));
  es.onmessage = (e) => { lastId = e.lastEventId; process(JSON.parse(e.data)); };
  es.onerror = () => setTimeout(connect, 1000);
}
connect();
```

The server must respect `Last-Event-ID` and persist a replay buffer. For richer guarantees use a server with explicit ack semantics (e.g. SSE-over-Kafka, or switch to WebSocket + your own protocol).

**Reference:** [WHATWG HTML Living Standard — Server-sent events, `Last-Event-ID`](https://html.spec.whatwg.org/multipage/server-sent-events.html#dispatchMessage).

### Bug 25 — JA3/JA4 fingerprint mismatch

**Root cause:** `requests` and `httpx` use Python's `ssl` stack (OpenSSL with the cipher order Python configures). Browsers use BoringSSL or NSS with a very different ClientHello: cipher list, extensions, extension order, GREASE values, ALPN entries. JA3 hashes those into a fingerprint; modern bot-detection (Cloudflare, Akamai, PerimeterX) maintains an allowlist of browser fingerprints. Setting a Chrome `User-Agent` while emitting Python's TLS fingerprint is a guaranteed flag.

**Fix:** Use a TLS impersonation client. Production scraping uses `curl_cffi` (libcurl-impersonate, ships browser fingerprints) or `tls-client` (Go-backed, exposes ClientHello presets):

```python
from curl_cffi import requests as cf_requests

r = cf_requests.get("https://protected.example.com",
                    impersonate="chrome124",
                    headers=BROWSER_HEADERS)
```

**Reference:** [Salesforce JA3 paper (2017)](https://github.com/salesforce/ja3) and the [JA4+ spec by FoxIO](https://github.com/FoxIO-LLC/ja4) which extends JA3 to TLS 1.3 stability.

### Bug 26 — H/2 SETTINGS frame fingerprint

**Root cause:** Akamai's HTTP/2 fingerprint hashes the `SETTINGS` frame parameters (`HEADER_TABLE_SIZE`, `INITIAL_WINDOW_SIZE`, `MAX_CONCURRENT_STREAMS`, etc.) plus `WINDOW_UPDATE` increments, the `PRIORITY` order of `HEADERS` frames, and the order of pseudo-headers. `httpx` with `http2=True` uses `h2` library defaults — distinct from any real browser. Setting `User-Agent: Chrome` while emitting `httpx`'s SETTINGS frame is detectable.

**Fix:** Same family of tools as Bug 25 — `curl_cffi` impersonates the H/2 stack as well, not just TLS. For deeper control, use `tls-client` which exposes H/2 settings explicitly. For browser-grade fidelity, just drive a real headless browser via Playwright or `nodriver`.

```python
# curl_cffi handles both TLS and H/2 fingerprints together
r = cf_requests.get(url, impersonate="chrome124")
```

**Reference:** [Akamai's HTTP/2 fingerprinting whitepaper "Passive Fingerprinting of HTTP/2 Clients" (Shuster, Black Hat EU 2017)](https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf).

### Bug 27 — Pinned cert behind TLS-terminating proxy

**Root cause:** A corporate or cloud-egress proxy that terminates TLS presents *its own* certificate (signed by the corp CA) to the client, then opens a fresh TLS connection to the origin. Pinning the origin cert means the chain validation runs against a cert your client never sees. Either trust the corp CA explicitly (and acknowledge you are no longer pinned to origin), or route around the terminating proxy (CONNECT tunneling) so the origin cert reaches the client.

**Fix (pick one):**

```python
# Option A: use CONNECT-tunneling proxy (no termination)
client = httpx.Client(verify=ctx, proxy="http://corp-proxy:8080")
# requires the proxy to support CONNECT for HTTPS — most do

# Option B: trust the corp CA but layer your own integrity check
ctx = ssl.create_default_context(cafile="corp-ca.pem")
client = httpx.Client(verify=ctx, proxy="http://corp-proxy:8080")
# then pin via app-layer signature, not TLS cert
```

**Reference:** [RFC 9110 §17 — Security considerations on intermediaries](https://www.rfc-editor.org/rfc/rfc9110.html#section-17) and [OWASP — Pinning Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Pinning_Cheat_Sheet.html) (the "MITM proxy" caveat).

### Bug 28 — GraphQL alias amplification

**Root cause:** GraphQL aliases let one query call the same field 100 times under different names. If `user(id)` resolves a 4-level deep tree and you alias it 100 times, the server runs 100 full subtree resolutions per request — easy way to multiply backend cost by 100 to 10,000 from a single HTTP call. This is a known DoS / cost-amplification class. Production servers should set query-cost limits (e.g. graphql-cost-analysis) and reject queries above threshold; clients should respect that and avoid alias amplification by accident when batching.

**Fix:**

```python
# batch with arrays instead of aliases (one root field, list arg)
query = """
query ($ids: [ID!]!) {
  users(ids: $ids) { id posts { comments { author { name } } } }
}
"""
requests.post(GRAPHQL, json={"query": query, "variables": {"ids": list(range(100))}})
```

The server then resolves `users` once, with a single DataLoader batch.

**Reference:** [GraphQL spec — Field Alias](https://spec.graphql.org/October2021/#sec-Field-Alias) and [Apollo blog — "Securing your GraphQL API from malicious queries" (cost analysis)](https://www.apollographql.com/blog/securing-your-graphql-api-from-malicious-queries).

### Bug 29 — WebSocket idle death

**Root cause:** AWS ALB defaults to 60-second idle timeout, GCP HTTPS LB to 600 seconds, NGINX to 60 seconds. After the idle window any intermediary closes the connection without sending a close frame. Clients only notice on the next send. For long-lived subscriptions you need application-level pings inside the idle budget.

**Fix:**

```javascript
const ws = new WebSocket(url);
const pingInterval = setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) ws.send(JSON.stringify({type: "ping"}));
}, 30_000);
ws.onclose = () => clearInterval(pingInterval);
ws.onmessage = (m) => handle(JSON.parse(m.data));
```

For RFC-6455 control frames specifically (not application-level pings), use a library that exposes `ws.ping()` directly (e.g. Node's `ws` package). Browser `WebSocket` can't send protocol-level pings.

**Reference:** [RFC 6455 §5.5.2 — Ping frame](https://www.rfc-editor.org/rfc/rfc6455#section-5.5.2) and [AWS ALB idle timeout docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#connection-idle-timeout).

### Bug 30 — Source-map leak from your scraper

**Root cause:** Default webpack/Vite production builds emit `dist/scraper.js.map` and append `//# sourceMappingURL=scraper.js.map` to the bundle. If you publish the bundle to a public CDN (or a worker URL that the target site eventually scrapes back), defenders fetch the map and recover your variable names, request templates, proxy list rotation logic, and selector strategies. Disable source maps for production scraper builds, or put them behind authn.

**Fix:**

```javascript
// webpack.config.js
module.exports = {
  mode: "production",
  devtool: false,   // disables source maps
};
```

For Vite: `build: { sourcemap: false }`. If you need source maps for your own debugging, ship them only to authenticated dashboards, never alongside the public bundle.

**Reference:** [webpack — `devtool` docs](https://webpack.js.org/configuration/devtool/) and the broader threat documented in [OWASP "Information Exposure Through Source Maps"](https://owasp.org/www-community/Improper_Error_Handling) (information disclosure category).

### Bug 31 — gzip-bomb defense missing

**Root cause:** A 1 KB gzip-compressed payload of zeros expands to roughly 1 GB. Calling `read(decode_content=True)` with no size cap inflates the entire stream into memory. Hostile servers (or bot-detection counter-attacks) ship deliberate decompression bombs at known scrapers. Cap both compressed and decompressed size, or stream with a running counter.

**Fix:**

```python
MAX_INFLATED = 50 * 1024 * 1024   # 50 MB

resp = requests.get(url, stream=True)
buf = bytearray()
for chunk in resp.iter_content(chunk_size=64 * 1024, decode_unicode=False):
    buf.extend(chunk)
    if len(buf) > MAX_INFLATED:
        raise ValueError("decompressed payload too large")
process(bytes(buf))
```

**Reference:** [CVE-2018-1000517 (BusyBox wget)](https://nvd.nist.gov/vuln/detail/CVE-2018-1000517) is a classic real-world example of this class; [OWASP "Decompression Bomb" entry](https://owasp.org/www-community/attacks/Decompression_Bomb_Vulnerability).

### Bug 32 — H/2 lowercase header requirement

**Root cause:** RFC 9113 §8.2.1 mandates that header field names in HTTP/2 be lowercase; sending `X-Custom-Header` triggers a connection error (`PROTOCOL_ERROR`) on strict servers. `httpx` lowercases automatically, but a server may reject anyway if your code constructs headers via low-level h2 API. Second issue: `Content-Length` is allowed in H/2 only if it matches the actual body byte count, and `Transfer-Encoding: chunked` is forbidden entirely (chunking is replaced by DATA frames). Sending `Content-Length: 0` on a GET is harmless; sending it on a request with a body of different length is a protocol error.

**Fix:**

```python
# httpx normalizes for you, but be explicit:
client = httpx.Client(http2=True)
r = client.get(url, headers={"x-custom-header": "value"})  # lowercase
# omit Content-Length unless you set the body and the length matches
```

For raw `h2` library work, normalize with `name.lower()` and never copy `Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, or `Upgrade` from H/1 headers.

**Reference:** [RFC 9113 §8.2.1 — Field Validity](https://www.rfc-editor.org/rfc/rfc9113.html#name-field-validity) and §8.2.2 on connection-specific header fields.

---

## Related

- [HTTP Scraping Fundamentals](06-http-scraping-fundamentals.md)
- [Headless Browsers](07-headless-browsers.md)
- [Proxies and IP Rotation](08-proxies-and-ip-rotation.md)
- [Bot Detection Internals](09-bot-detection-internals.md)
- [CAPTCHA Systems and Solvers](10-captcha-systems-and-solvers.md)
- [Sitemap, robots.txt, and Crawl Surface](../discovery/03-sitemap-robots-and-crawl-surface.md)

## References

- [RFC 9309 — Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309.html)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html)
- [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [RFC 6585 — Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)
- [RFC 3986 — URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)
- [AWS Architecture Blog — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Salesforce JA3](https://github.com/salesforce/ja3) and [FoxIO JA4+](https://github.com/FoxIO-LLC/ja4)
- [Shuster — Passive Fingerprinting of HTTP/2 Clients (Black Hat EU 2017)](https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf)
- [`requests` Session and redirect docs](https://requests.readthedocs.io/)
- [`httpx` resource limits](https://www.python-httpx.org/advanced/resource-limits/)
- [`undetected-chromedriver`](https://github.com/ultrafunkamsterdam/undetected-chromedriver)
- [`curl_cffi`](https://github.com/lexiforest/curl_cffi)
