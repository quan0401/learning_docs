---
title: "Headless Browsers — Playwright, Puppeteer, Selenium"
date: 2026-05-06
updated: 2026-05-06
tags: [headless, playwright, puppeteer, selenium, undetected-chromedriver, stealth, browser-automation]
---

# Headless Browsers — Playwright, Puppeteer, Selenium

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `headless` `playwright` `puppeteer` `selenium` `undetected-chromedriver` `stealth` `browser-automation`

---

## Table of Contents

1. [When You Actually Need a Headless Browser](#1-when-you-actually-need-a-headless-browser)
2. [Playwright — The Modern Default](#2-playwright--the-modern-default)
3. [Puppeteer — Node-Native, Chrome-Specific](#3-puppeteer--node-native-chrome-specific)
4. [Selenium 4 — The Cross-Language Veteran](#4-selenium-4--the-cross-language-veteran)
5. [`undetected-chromedriver` and Stealth Plugins](#5-undetected-chromedriver-and-stealth-plugins)
6. [Request Interception and Resource Blocking](#6-request-interception-and-resource-blocking)
7. [Concurrency, Contexts, and Memory](#7-concurrency-contexts-and-memory)
8. [Cost and Performance Reality](#8-cost-and-performance-reality)
9. [Hybrid Pattern — Headless for Auth, HTTP for Bulk](#9-hybrid-pattern--headless-for-auth-http-for-bulk)

## Summary

Headless browsers run a real Chromium/Firefox/WebKit instance under programmatic control. They render JavaScript, execute the full page lifecycle, and let you query the resulting DOM — at the cost of being roughly 30–100× more expensive than raw HTTP per page. Reach for them only when JS rendering is essential, or when the API behind the page is signed/captcha-gated in a way that's harder to replay than to drive the browser. **Playwright** (Microsoft, 2020) is the modern default — single API across Chromium/Firefox/WebKit, Python/Node/Java/.NET bindings, robust auto-waiting, built-in trace viewer. **Puppeteer** (Google, 2017) is Node + Chromium only but is the original of this generation. **Selenium 4** is the cross-language veteran with WebDriver BiDi and the broadest ecosystem. **`undetected-chromedriver`** and **playwright/puppeteer-stealth** patch the well-known automation tells the browser leaks. The unavoidable trade-off: more realistic browser → more CPU + RAM → fewer concurrent crawls per VM.

---

## 1. When You Actually Need a Headless Browser

Reach for headless when the data-extraction problem is *genuinely* a rendering problem:

- The data is computed client-side (charts, tables built from a WebSocket feed)
- The site requires complex JS-driven authentication (anti-CSRF tokens generated in JS, browser-only TOTP)
- Anti-bot is so layered that replaying the API costs more reverse-engineering than driving the browser
- You need a screenshot or PDF rendering of the page itself
- The API the page uses is unstable or generates per-request signed payloads from JS that's intentionally hard to reproduce (doc 13)

**Don't** reach for headless when:

- The data is in initial HTML (just use HTTP)
- The data is in a JSON API call you can replay
- The page uses Next.js/Nuxt/Remix server-side rendering — the data is in `__NEXT_DATA__` etc.

---

## 2. Playwright — The Modern Default

[Playwright](https://playwright.dev/) (Microsoft, 2020) is built by the team that originally wrote Puppeteer. It controls Chromium, Firefox, and WebKit through one API.

### 2.1 Bindings

- **Node/TypeScript** — primary
- **Python** — first-party, [`playwright-python`](https://playwright.dev/python/)
- **Java** — first-party
- **.NET** — first-party
- (Go has community bindings)

### 2.2 Core Concepts

```ts
import { chromium } from "playwright";

const browser = await chromium.launch({ headless: true });
const context = await browser.newContext({
  viewport: { width: 1280, height: 720 },
  userAgent: "Mozilla/5.0 ...",
  locale: "en-US",
  timezoneId: "America/New_York",
});
const page = await context.newPage();

await page.goto("https://example.com/login");
await page.locator('input[name="email"]').fill("user@example.com");
await page.locator('input[name="password"]').fill("...");
await page.locator('button[type="submit"]').click();
await page.waitForURL("**/dashboard");

const data = await page.locator(".dashboard-data").textContent();
await browser.close();
```

The hierarchy: **Browser** (one Chromium process) → **BrowserContext** (isolated cookie jar / storage / cache, like an incognito window) → **Page**. Multiple contexts in one browser = parallelism without spawning multiple Chromium processes. Each context is fully isolated.

### 2.3 Auto-Waiting

Playwright's `locator` API auto-waits for actionability — visibility, attached, stable, enabled — before clicking/filling. This eliminates most `sleep()` calls Selenium taught a generation. You almost never need explicit waits.

```ts
// Wrong (race-prone) — Selenium-style
await page.click(".submit");
await page.waitForTimeout(2000);
const result = await page.textContent(".result");

// Right — Playwright auto-waits
await page.locator(".submit").click();
const result = await page.locator(".result").textContent();
```

### 2.4 Trace Viewer

`playwright trace.zip` opens an interactive trace viewer with DOM snapshots, network log, console, and source per action. For debugging flaky scrapes this is invaluable.

### 2.5 Codegen

`npx playwright codegen https://example.com` opens a browser; every action you perform generates the corresponding Playwright code. Useful for building login flows quickly.

---

## 3. Puppeteer — Node-Native, Chrome-Specific

[Puppeteer](https://pptr.dev/) (Google) is older, Node-only, and Chrome/Chromium-only. It's mature, lighter than Playwright, and has the most community plugins.

### 3.1 Modern Puppeteer

```js
import puppeteer from "puppeteer";

const browser = await puppeteer.launch({ headless: "new" });
const page = await browser.newPage();
await page.goto("https://example.com");
await page.waitForSelector("h1");
const title = await page.$eval("h1", (el) => el.textContent);
await browser.close();
```

### 3.2 Headless Modes

Puppeteer 22+ defaults to `headless: "shell"` (the new lightweight mode) or `headless: true` (full Chrome). The old "true headless" mode was distinguishable from real Chrome via numerous tells; "new headless" runs the real Chrome binary in a no-window mode and is much closer to a real browser.

### 3.3 Puppeteer vs Playwright

| Dimension | Puppeteer | Playwright |
|-----------|-----------|------------|
| Languages | Node only | Node, Python, Java, .NET |
| Browsers | Chrome/Chromium | Chromium, Firefox, WebKit |
| Auto-wait | Limited | Yes |
| Trace viewer | No | Yes |
| Plugin ecosystem | Mature | Growing |
| Stealth tooling | `puppeteer-extra-plugin-stealth` | `playwright-stealth` |

For new work, prefer Playwright. Puppeteer is fine for existing codebases.

---

## 4. Selenium 4 — The Cross-Language Veteran

[Selenium](https://www.selenium.dev/) is the oldest. Selenium 4 (2021) added WebDriver BiDi (the bidirectional successor to the W3C [WebDriver protocol](https://www.w3.org/TR/webdriver/)) and modernized the API. It still has the broadest language support — Java, Python, C#, Ruby, JavaScript, Kotlin — and the largest ecosystem of grid services (Selenium Grid, BrowserStack, Sauce Labs).

### 4.1 When to Pick Selenium

- You're integrating with an existing Java/.NET/Ruby test suite
- You need a managed grid with cross-browser/cross-OS coverage
- The team already knows it

### 4.2 Modern Selenium API

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

driver = webdriver.Chrome()
driver.get("https://example.com")
WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.TAG_NAME, "h1")))
print(driver.find_element(By.TAG_NAME, "h1").text)
driver.quit()
```

### 4.3 The Webdriver Tells

Selenium uses ChromeDriver, which by default leaves multiple JavaScript-visible signals:

- `navigator.webdriver === true`
- `window.chrome.runtime` differences
- `Permissions API` defaults that differ from real Chrome
- CDP-injected JavaScript hooks that real Chrome doesn't have

Modern bot detectors (doc 09) check all of these. The fix is `undetected-chromedriver`.

---

## 5. `undetected-chromedriver` and Stealth Plugins

### 5.1 `undetected-chromedriver`

[`undetected-chromedriver`](https://github.com/ultrafunkamsterdam/undetected-chromedriver) (Python) patches the ChromeDriver binary to remove the `cdc_*` JS variables, `navigator.webdriver`, and other automation tells. It uses Selenium's API but starts an effectively-clean browser.

```python
import undetected_chromedriver as uc
driver = uc.Chrome(version_main=124)
driver.get("https://example.com")
```

### 5.2 `puppeteer-extra-plugin-stealth`

[Berstend's `puppeteer-extra-plugin-stealth`](https://github.com/berstend/puppeteer-extra/tree/master/packages/puppeteer-extra-plugin-stealth) is the Puppeteer equivalent — a bundle of evasions for known automation fingerprints:

```js
import puppeteer from "puppeteer-extra";
import StealthPlugin from "puppeteer-extra-plugin-stealth";
puppeteer.use(StealthPlugin());

const browser = await puppeteer.launch({ headless: "new" });
```

The plugin patches `navigator.webdriver`, `chrome.runtime`, plugin lists, language arrays, WebGL vendor/renderer strings, and ~20 other surface-level tells.

### 5.3 Playwright Stealth

[`playwright-stealth`](https://github.com/AtuboDad/playwright_stealth) ports the same evasions to Playwright. Coverage is somewhat behind the Puppeteer plugin.

### 5.4 What Stealth Plugins Don't Fix

Stealth plugins fix the **JavaScript-observable surface** of the browser. They do not fix:

- **TLS fingerprint (JA3/JA4)** — Chromium with stealth still presents Chromium's TLS fingerprint (which is fine when you're impersonating Chromium); but Python/Go HTTP clients calling under the hood still leak
- **HTTP/2 settings fingerprint** — same; the browser is fine, your script isn't
- **Behavioral fingerprints** — mouse movement, typing cadence, time-on-page (doc 09)
- **IP reputation** — the IP you're coming from (doc 08)
- **CAPTCHA challenges** — those check anti-fraud signals beyond browser identity (doc 10)

Stealth solves a fraction of the problem. The rest is doc 08 (proxies), doc 09 (deeper fingerprinting), and doc 10 (CAPTCHA).

---

## 6. Request Interception and Resource Blocking

### 6.1 Why Block Resources

A typical news article page loads:

- 1 HTML document (~50 KB)
- 50+ images (~5 MB)
- 20+ third-party tracking scripts (~3 MB)
- Several large CSS files
- Web fonts

If you only want the article text, blocking images/fonts/styles cuts page-load by ~80% and saves bandwidth and CPU.

### 6.2 Playwright

```ts
await page.route("**/*", (route) => {
  const type = route.request().resourceType();
  if (["image", "font", "stylesheet", "media"].includes(type)) {
    return route.abort();
  }
  return route.continue();
});
```

### 6.3 Puppeteer

```js
await page.setRequestInterception(true);
page.on("request", (req) => {
  if (["image", "font", "stylesheet"].includes(req.resourceType())) {
    return req.abort();
  }
  req.continue();
});
```

### 6.4 Selenium / CDP

Selenium 4 has CDP-based interception via `driver.execute_cdp_cmd("Network.setBlockedURLs", {...})`. Verbose; Playwright/Puppeteer are nicer for this.

### 6.5 Caveat — Detection of Aborted Resources

Anti-bot can check whether the browser ever fetched the analytics tracker. If your scraper aborts every analytics request, that's a tell. Usually fine; sometimes flagged.

---

## 7. Concurrency, Contexts, and Memory

### 7.1 Per-Process Memory

A bare Chromium process runs ~80–150 MB. Each open page adds ~50–100 MB depending on the page. So `N` concurrent Playwright pages ≈ `100 + 80*N` MB.

For a `4 GB` VM with everything else taken into account, ~30–40 concurrent pages is a realistic upper bound. Beyond that, swap or OOM.

### 7.2 Context-per-Job vs Page-per-Job

Two concurrency models:

- **One context per job, one page per context** — strongest isolation; each job has clean cookies/storage/cache. Most expensive.
- **One context, many pages** — pages share cookies and storage; cheaper; works when each job *should* share session state (e.g., logged-in scraping)
- **One browser, many contexts, one page each** — middle ground; multiple sessions in one Chromium

Pick by isolation requirement. Most production scrapers use *one browser per worker* with *one context per session* and *one page per task*.

### 7.3 Browser Pool

For long-running services, run a pool of pre-warmed browsers and check pages in/out:

```ts
import { chromium, Browser } from "playwright";

class BrowserPool {
  private pool: Browser[] = [];
  async checkout(): Promise<Browser> {
    if (this.pool.length === 0) {
      return chromium.launch({ headless: true });
    }
    return this.pool.pop()!;
  }
  async checkin(browser: Browser) {
    this.pool.push(browser);
  }
}
```

Recycle browsers periodically; long-running Chromium processes leak memory under heavy load.

### 7.4 Crawlee's PlaywrightCrawler

[Crawlee](https://crawlee.dev/) (doc 06) handles browser pooling, session cookies per worker, and proxy rotation transparently. For Node teams, it's the easiest production setup.

---

## 8. Cost and Performance Reality

### 8.1 Per-Page Cost Comparison

For a typical product page (rendered):

| Approach | CPU sec | RAM peak | Wall time | Concurrent on 4-vCPU |
|----------|---------|----------|-----------|----------------------|
| `requests` HTTP-only | ~0.05 | <50 MB | 0.5–2s | 100+ |
| Playwright headless, no resource block | ~1–3 | ~150 MB | 2–8s | ~10 |
| Playwright + image/font block | ~0.5–1 | ~120 MB | 1–3s | ~20–30 |
| Playwright + stealth + proxy | ~1–4 | ~150 MB | 3–10s | ~10–15 |

For 100k pages/day:

- HTTP: 100k × 1s ÷ 100 concurrent ≈ ~17 min on one box
- Headless: 100k × 5s ÷ 15 concurrent ≈ ~9 hours on one box

The cost ratio is real. Reserve headless for genuinely-required pages.

### 8.2 Cloud Browser Services

Services that run headless browsers as infrastructure:

- [Browserless](https://www.browserless.io/) — paid; managed Chromium farm
- [Apify](https://apify.com/) — paid; broader scraping platform
- [ScrapingBee](https://www.scrapingbee.com/) — paid; "render any URL" API
- [BrowserCat](https://www.browsercat.com/) — paid; Chromium API

The pricing is roughly $0.001–$0.01 per page, which is 10× the cost of self-hosting once you have steady volume — but worth it during prototyping or low-volume work.

---

## 9. Hybrid Pattern — Headless for Auth, HTTP for Bulk

The most efficient production pattern:

1. **Use headless once** to perform login (or the JS-heavy initialization), then export the cookies and `Authorization` tokens.
2. **Reuse those cookies/tokens with raw HTTP** for the bulk of the crawl.
3. **Refresh the headless session periodically** when tokens expire.

```python
# 1. Headless login
with sync_playwright() as p:
    browser = p.chromium.launch()
    context = browser.new_context()
    page = context.new_page()
    page.goto("https://example.com/login")
    page.locator('input[name="email"]').fill(EMAIL)
    page.locator('input[name="password"]').fill(PWD)
    page.locator('button[type="submit"]').click()
    page.wait_for_url("**/dashboard")
    cookies = context.cookies()
    storage = context.storage_state()  # cookies + localStorage
    browser.close()

# 2. HTTP scraping with the harvested session
session = requests.Session()
for c in cookies:
    session.cookies.set(c["name"], c["value"], domain=c["domain"], path=c["path"])
for url in target_urls:
    r = session.get(url)
    # parse...
```

This pattern combines headless's strength (handling complex auth) with HTTP's strength (cheap parallelism). It works for any site where the post-login API is stateless modulo the session cookie.

---

## Related

- [HTTP Scraping Fundamentals](06-http-scraping-fundamentals.md) — read first; headless is the fallback
- [Proxies and IP Rotation](08-proxies-and-ip-rotation.md) — headless without rotated IPs is detectable
- [Bot Detection Internals](09-bot-detection-internals.md) — what stealth plugins try to evade
- [Chrome DevTools for Scraping](../reverse-engineering/11-chrome-devtools-for-scraping.md) — the manual version of what Playwright automates
- [API Reverse-Engineering](../reverse-engineering/13-api-reverse-engineering.md) — the alternative to driving the browser

## References

- [Playwright documentation](https://playwright.dev/)
- [Puppeteer documentation](https://pptr.dev/)
- [W3C WebDriver specification](https://www.w3.org/TR/webdriver/)
- [W3C WebDriver BiDi specification](https://w3c.github.io/webdriver-bidi/)
- [`undetected-chromedriver`](https://github.com/ultrafunkamsterdam/undetected-chromedriver)
- [`puppeteer-extra-plugin-stealth`](https://github.com/berstend/puppeteer-extra/tree/master/packages/puppeteer-extra-plugin-stealth)
- [Playwright vs Puppeteer comparison](https://playwright.dev/docs/why-playwright)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) — the underlying control protocol
- [Crawlee — PlaywrightCrawler reference](https://crawlee.dev/api/playwright-crawler/class/PlaywrightCrawler)
