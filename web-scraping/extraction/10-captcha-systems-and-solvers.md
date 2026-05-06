---
title: "CAPTCHA Systems and Solvers"
date: 2026-05-06
updated: 2026-05-06
tags: [captcha, recaptcha, hcaptcha, turnstile, funcaptcha, geetest, captcha-solvers]
---

# CAPTCHA Systems and Solvers

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `captcha` `recaptcha` `hcaptcha` `turnstile` `funcaptcha` `geetest` `captcha-solvers`

---

## Table of Contents

1. What CAPTCHAs Actually Do
2. Google reCAPTCHA v2
3. Google reCAPTCHA v3
4. Google reCAPTCHA Enterprise
5. hCaptcha
6. Cloudflare Turnstile
7. FunCaptcha (Arkose Labs)
8. Geetest
9. AWS WAF CAPTCHA and Bot Control
10. Older and Niche CAPTCHA Systems
11. CAPTCHA Solver Services
12. Self-Hosted ML Solvers
13. The Audio Challenge Bypass History
14. Terms-of-Service and Legal Implications
15. Detecting CAPTCHA in Your Scraper
16. Cost Analysis
17. Ethics

## Summary

CAPTCHAs are not a single thing: they range from binary "prove you can read this image" challenges to invisible risk-scoring systems that watch mouse trajectories, IP reputation, browser fingerprints, and behavioral telemetry to produce a score that the protected site combines with its own logic. This doc walks through the major providers a scraper actually encounters in 2026 (Google reCAPTCHA v2/v3/Enterprise, hCaptcha, Cloudflare Turnstile, Arkose FunCaptcha, Geetest, AWS WAF), how their token model works, the solver-service ecosystem that resells human and ML labor by the thousand, the limits of self-hosted ML, and the ToS, CFAA, and ethical territory you walk into the moment you wire any of this into a production scraper.

---

## 1. What CAPTCHAs Actually Do

A CAPTCHA is a Turing test the server forces between "user requests sensitive endpoint" and "server processes that request." Two broad shapes:

- **Binary challenges** — the user must complete a visible task (read distorted text, pick squares with bicycles, rotate a 3D animal upright). Output: pass or fail.
- **Risk-scoring systems** — the script collects telemetry passively and produces a continuous score (usually 0.0–1.0). The site decides what to do with low scores — block, soft-block, ask for a step-up challenge, or just log.

Modern providers run a hybrid: collect telemetry first, only show a visible challenge if the passive signal is suspicious. Google calls these tiers "v3 invisible," "v2 checkbox," and "advanced challenge" (the image grid). hCaptcha, Turnstile, and FunCaptcha follow the same pattern.

The token returned by the widget is **opaque** to the page. The page POSTs the token along with the form, and the **server** calls the provider's `siteverify` endpoint (with its secret key) to learn the score and the action. That round-trip is what lets a scraper "solve" a CAPTCHA out-of-band: any token issued for the right sitekey + URL + action is acceptable, regardless of who or what produced it. This is the architectural seam every solver service exploits.

Key terms used throughout:

| Term | Meaning |
|------|---------|
| sitekey | Public key embedded in the page; identifies the protected site to the provider |
| secret key | Server-side key used to verify tokens against the provider |
| token | Opaque string the widget returns; valid for one verification |
| action | Optional string ("login", "checkout") that scopes a v3 token |
| siteverify | Provider endpoint the protected server calls to validate a token |

## 2. Google reCAPTCHA v2

reCAPTCHA v2 is the "I'm not a robot" checkbox plus optional image grid fallback. Two visible variants: checkbox v2 and invisible v2 (no checkbox; runs on a button click).

**Flow:**

1. Page includes `https://www.google.com/recaptcha/api.js` and a `<div class="g-recaptcha" data-sitekey="...">`.
2. User clicks the checkbox. The widget evaluates browser/behavior signals.
3. If the score is high, the widget marks itself solved and emits a `g-recaptcha-response` token (a long base64-ish string ~400–1900 chars) into a hidden form field.
4. If the score is low or signals are missing, the user gets the image grid challenge (3x3 or 4x4 tiles, "select all crosswalks").
5. After the form POSTs, the server verifies:

```ts
// Server-side verification (Node/TS)
const res = await fetch("https://www.google.com/recaptcha/api/siteverify", {
  method: "POST",
  headers: { "content-type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({
    secret: process.env.RECAPTCHA_SECRET!,
    response: token,
    remoteip: clientIp,
  }),
});
const body = await res.json() as {
  success: boolean;
  challenge_ts: string;
  hostname: string;
  "error-codes"?: string[];
};
```

A token is valid for ~2 minutes and **single-use**. Tokens are scoped to the sitekey but not to a specific action (that's a v3 thing).

## 3. Google reCAPTCHA v3

v3 has no checkbox, no challenge UI. The script runs in the background, collects signals, and produces a token with an **action** label. The site requests a score from siteverify and combines it with its own logic:

- Score `0.9` — almost certainly human, allow.
- Score `0.3` — probably bot, deny or step up.
- Score `0.7` — apply additional friction (email verification, lower rate limit, hold for review).

```html
<script src="https://www.google.com/recaptcha/api.js?render=SITE_KEY"></script>
<script>
  grecaptcha.ready(() => {
    grecaptcha.execute('SITE_KEY', { action: 'submit' })
      .then(token => { document.getElementById('token').value = token; });
  });
</script>
```

Server side, the response includes:

```json
{
  "success": true,
  "score": 0.9,
  "action": "submit",
  "challenge_ts": "2026-05-06T12:34:56Z",
  "hostname": "example.com"
}
```

Tokens expire at **120 seconds** and are single-use. The action **must** match what the server expected, otherwise the token is treated as suspect even if the score is high. Scrapers solving v3 must therefore know the exact action string the target site uses, which is usually visible in the page source.

## 4. Google reCAPTCHA Enterprise

The paid tier. Same widget UX, but the assessment API is richer: `expectedAction`, password-leak signals, account-defender integration, and a `riskAnalysis.reasons` array that surfaces categorical reasons ("AUTOMATION," "UNEXPECTED_ENVIRONMENT," "TOO_MUCH_TRAFFIC," "LOW_CONFIDENCE_SCORE").

Server side, instead of `siteverify` you call the Enterprise API:

```ts
// Pseudocode using @google-cloud/recaptcha-enterprise
const [assessment] = await client.createAssessment({
  parent: `projects/${projectId}`,
  assessment: {
    event: { token, siteKey, expectedAction: "checkout" },
  },
});
const score = assessment.riskAnalysis?.score ?? 0;
const reasons = assessment.riskAnalysis?.reasons ?? [];
```

For scrapers this means two things: (a) the protected site has more granular signal so naive headless setups fail more often, and (b) solver services charge a premium for "Enterprise" tokens because they need to maintain higher-quality residential proxy pools and behavioral emulation.

## 5. hCaptcha

hCaptcha (Intuition Machines) is the most common reCAPTCHA replacement. Same architectural shape:

- Widget script: `https://js.hcaptcha.com/1/api.js`.
- Sitekey on the `<div class="h-captcha">`.
- Solved token in a hidden field named `h-captcha-response`.
- Server calls `https://api.hcaptcha.com/siteverify` with `secret` and `response`.

It markets itself on privacy (no cookie linkage to a Google ad profile) and on paying website operators for image labels (the early "you're solving ML training data" pitch). For scrapers, the integration looks identical to reCAPTCHA v2 except for the endpoints and key names. Cloudflare adopted hCaptcha as its CAPTCHA option in 2020 before switching to Turnstile in 2022.

There is also hCaptcha Enterprise with extra signals; pricing varies and the verify response can include `score` and `score_reason` similar to reCAPTCHA Enterprise.

## 6. Cloudflare Turnstile

Turnstile is Cloudflare's response to "CAPTCHAs are user-hostile." It's designed to be cookie-less and almost always invisible. The widget runs a series of non-interactive checks (browser API attestation, behavioral signals, proof-of-work fallbacks) and returns a token without showing a challenge in the median case. When it does show one, it's a single click ("verify you are human").

```html
<div class="cf-turnstile" data-sitekey="YOUR_SITE_KEY"></div>
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" defer></script>
```

Server verification:

```ts
const verify = await fetch(
  "https://challenges.cloudflare.com/turnstile/v0/siteverify",
  {
    method: "POST",
    headers: { "content-type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      secret: process.env.TURNSTILE_SECRET!,
      response: token,
      remoteip: clientIp,
    }),
  },
);
```

Turnstile tokens are **single-use** and expire after ~300 seconds. Notably, Turnstile's threat signal blends with Cloudflare's bot management — the same backbone that already sees most internet traffic — so a scraper that gets past Turnstile usually still has to deal with WAF rules, JS challenges, and IP reputation tuning behind it. Turnstile alone is not the moat; it's the visible front door.

## 7. FunCaptcha (Arkose Labs)

Arkose's FunCaptcha shows interactive 3D challenges: rotate the animal so it's upright, pick the image where the dice add to seven, drag the puzzle piece. LinkedIn, Twitter/X, Roblox, and several major banks use it. It's intentionally expensive to solve at scale because:

- Each challenge is procedurally generated from a 3D scene.
- The token model couples the challenge to a session.
- Arkose actively rotates challenge types and adds new ones when bypass methods appear.

Tokens are valid for **one submission** and a short window (commonly under a minute). The Arkose JS bundle includes heavy obfuscation, behavioral telemetry, and integrity checks that detect Playwright/Puppeteer drivers. Solver services charge 5–10× more for FunCaptcha than for reCAPTCHA v2 because they run dedicated farms with custom automation.

## 8. Geetest

Geetest is the dominant CAPTCHA in mainland China and is widely deployed across APAC. It pioneered the **slider puzzle** (drag the piece into the gap), and v3/v4 add click-to-rotate, icon-order, and "click in order" variants. The challenge interface has a distinctive look: dark blue gradient, animated piece, "Drag to complete the puzzle" instruction.

Geetest verification is multi-step:

1. Page calls Geetest's "register" endpoint to get a challenge ID.
2. Widget renders, user solves, JS posts encrypted answer parameters.
3. Page sends `geetest_challenge`, `geetest_validate`, `geetest_seccode` to the protected server.
4. Server calls Geetest's `validate.php` endpoint to verify.

For solvers, the slider has historically been the easiest interactive CAPTCHA to defeat with computer vision (find the gap, generate a believable human drag trajectory). Geetest counters by adding behavioral fingerprinting on the drag itself — too-perfect curves fail.

## 9. AWS WAF CAPTCHA and Bot Control

AWS WAF added a CAPTCHA action in 2022 and a Challenge action shortly after. Sites running AWS WAF can route requests matching specific rules through an interstitial: a Cloudflare-style "checking your browser" page with an optional CAPTCHA. The CAPTCHA action issues a signed token in a cookie (`aws-waf-token`) that's valid for a configurable duration (default 5 minutes for the Challenge action). Bot Control adds heuristic and ML-driven bot scoring on top of the WAF.

Practically: scrapers see this as a 405/403 with a body containing AWS WAF JS, or a redirect to a page that loads `https://*.awswaf.com/.../challenge.js`. Solving requires running the JS to get the token cookie, then replaying it on subsequent requests. Solver services have started supporting this because more APIs are sitting behind it.

## 10. Older and Niche CAPTCHA Systems

- **KeyCAPTCHA** — drag-piece-to-position; mostly legacy installs.
- **BotDetect** — image-based custom CAPTCHA, ASP.NET ecosystem; weak by modern standards.
- **Solve Media** — text-based ad CAPTCHAs, largely defunct.
- **Custom image CAPTCHAs** — sites that roll their own (rotate this image, type these characters, math problems). These are the easiest to defeat with off-the-shelf OCR or a small CNN; nobody serious uses them as their only line of defense in 2026.
- **Text-based "what's the third word in this sentence"** — accessibility-friendly but trivially defeated by an LLM. Showing up more often as a complementary layer rather than the primary defense.

## 11. CAPTCHA Solver Services

The market is mature. The main players:

| Service | Notes |
|---------|-------|
| 2Captcha (RuCaptcha) | Oldest, biggest human farm; broad CAPTCHA support |
| Anti-Captcha | Similar to 2Captcha; long history |
| CapSolver | ML-first, faster on reCAPTCHA v3 / Turnstile |
| CapMonster Cloud | From the makers of XEvil; ML-heavy |
| DeathByCaptcha | Smaller; legacy integrations |

**How they work:** a hybrid of human labor (low-paid workers on dashboards in countries with low wages) and ML models (CNNs for image grids, audio transcribers, behavioral emulators for Turnstile). When you submit an image-CAPTCHA the worker sees it, types the answer, and you get the response. When you submit a reCAPTCHA token request, the service spins up a real browser, navigates to your URL, solves it (with a worker if it falls to image grid), and returns the token.

**Pricing** (2026, approximate, subject to change):

- Image CAPTCHA: $0.50–$1 per 1000.
- reCAPTCHA v2: $1–$3 per 1000.
- reCAPTCHA v3: $2–$3 per 1000.
- reCAPTCHA Enterprise: $3–$5 per 1000.
- hCaptcha: $1–$3 per 1000.
- Turnstile: $1–$2 per 1000.
- FunCaptcha: $5–$15 per 1000.
- Geetest: $2–$5 per 1000.

**Integration pattern (2Captcha-style):**

```ts
// Submit a reCAPTCHA v2 task
const create = await fetch("https://2captcha.com/in.php", {
  method: "POST",
  body: new URLSearchParams({
    key: process.env.TWOCAPTCHA_KEY!,
    method: "userrecaptcha",
    googlekey: SITEKEY_FROM_TARGET,
    pageurl: TARGET_URL,
    json: "1",
  }),
});
const { request: taskId } = await create.json();

// Poll for the result (typical solve time 15–60 seconds)
async function pollToken(id: string): Promise<string> {
  const POLL_INTERVAL_MS = 5_000;
  const MAX_ATTEMPTS = 24;
  for (let i = 0; i < MAX_ATTEMPTS; i++) {
    await new Promise((r) => setTimeout(r, POLL_INTERVAL_MS));
    const r = await fetch(
      `https://2captcha.com/res.php?key=${process.env.TWOCAPTCHA_KEY}&action=get&id=${id}&json=1`,
    );
    const body = await r.json() as { status: 0 | 1; request: string };
    if (body.status === 1) return body.request;
    if (body.request !== "CAPCHA_NOT_READY") {
      throw new Error(`solver error: ${body.request}`);
    }
  }
  throw new Error("solver timeout");
}

const token = await pollToken(taskId);
// Now POST to target with g-recaptcha-response = token
```

CapSolver/Anti-Captcha use a similar create-task / get-result shape with JSON bodies and a `taskType` discriminator.

A crucial nuance: for **v3 / Enterprise / Turnstile / FunCaptcha**, the solver runs the widget on its own infrastructure with its own browser and IP. The token it returns is bound to that origin, not to your scraper's IP. Most providers don't bind the token to client IP for v3 specifically, which is what makes the model work — but Enterprise and FunCaptcha are tightening this. You may need to pay extra for "proxy-tied" solving where you supply the residential proxy and the solver runs the widget through it.

## 12. Self-Hosted ML Solvers

Open-source options exist and are useful for educational work and resilience-testing:

- **Buster** ([github.com/dessant/buster](https://github.com/dessant/buster)) — browser extension that solves reCAPTCHA's audio challenge by downloading the audio file and running it through a speech-to-text service. Works in Chrome/Firefox; intended as an accessibility aid.
- **Open-source CV** — many GitHub repos for reCAPTCHA image grids using YOLO or fine-tuned ResNet on the "select all bicycles" categories. They work for a while, then break when Google rotates the image set.
- **OCR for custom CAPTCHAs** — Tesseract or a small CNN solves most legacy text CAPTCHAs trivially.

The honest limit: a self-hosted solver is a treadmill. Providers update challenges, you retrain. For one-shot research it's fine; for production scraping it's a maintenance liability that solver services already pay for.

## 13. The Audio Challenge Bypass History

reCAPTCHA v2 has an accessibility option: a small audio icon that plays distorted speech of digits. From ~2017 to 2020, feeding that audio to a free speech-to-text service (Google Speech-to-Text, ironically; or Wit.ai, or IBM Watson) returned the right digits with high enough accuracy to defeat the challenge. The "unCaptcha" research papers (Bock et al., 2017 and unCaptcha2, 2019) demonstrated >90% success rates against the audio challenge using nothing but commercial STT APIs.

Google has since hardened the audio challenge (added background noise, made it longer, started detecting STT-style request patterns at the file-download step). It's still possible — Buster is a working example — but not as cleanly. The episode is the canonical example of a defense that worked on the assumption "audio is hard to recognize" outliving its assumption.

## 14. Terms-of-Service and Legal Implications

The Google reCAPTCHA Terms of Service explicitly prohibit:

> "Programmatically submit responses to verify human users... or otherwise use the Service in connection with any application that... attempts to defeat the purpose of the Service."

(See `https://policies.google.com/terms` and the reCAPTCHA-specific terms linked from the developer site.)

Practical reading:

- Using a solver service to bypass reCAPTCHA on a Google-protected property is a clear ToS violation.
- Doing so against a third-party site that uses reCAPTCHA implicates both Google's ToS and the target site's ToS.
- In the US this can intersect with the Computer Fraud and Abuse Act (CFAA), particularly after **hiQ Labs v. LinkedIn** (9th Cir., remanded 2022): scraping public data without authentication likely is not CFAA-violating, but bypassing technical access controls is closer to "exceeds authorized access." A solver service circumventing a CAPTCHA arguably is an access-control bypass.
- In the EU, the Database Directive and the Computer Misuse Act analogues in member states have similar contours.

This is not legal advice. The safe rule for non-lawyers: **CAPTCHA solving against properties you don't own or don't have written authorization to test is high-risk territory.** See [Production Scraping Hygiene and Legal](../reverse-engineering/14-production-scraping-hygiene-and-legal.md).

## 15. Detecting CAPTCHA in Your Scraper

A scraper needs to know it hit a CAPTCHA before it can decide what to do. Heuristics in order of cost:

**Cheap status-code/header checks:**

- HTTP `403` or `429` with a tiny body — likely a WAF challenge.
- HTTP `200` but body is a few KB and contains `cf-turnstile`, `g-recaptcha`, `h-captcha`, or `awswaf` markers.
- Redirects through `*.awswaf.com`, `challenges.cloudflare.com`, or back to a `/challenge` path.

**DOM probes (in headless mode):**

```ts
async function detectCaptcha(page: Page) {
  return await page.evaluate(() => {
    const sel = (s: string) => document.querySelector(s) != null;
    return {
      recaptchaV2: sel("[data-sitekey].g-recaptcha"),
      recaptchaV3Script: !!document.querySelector('script[src*="recaptcha/api.js"]'),
      hcaptcha: sel(".h-captcha"),
      turnstile: sel(".cf-turnstile"),
      funcaptcha: sel("#FunCaptcha") || !!document.querySelector('script[src*="funcaptcha"]'),
      geetest: sel(".geetest_btn") || sel(".geetest_holder"),
      awsWaf: !!document.querySelector('script[src*="awswaf"]'),
    };
  });
}
```

**Response strategy:**

| Signal | Strategy |
|--------|----------|
| First time seeing CAPTCHA on this domain | Rotate proxy, retry once |
| Persistent CAPTCHA across rotations | Submit to solver, supply sitekey + URL |
| FunCaptcha or Enterprise detected | Pay solver for the higher-cost task type |
| 5+ retries fail | Stop hammering; back off for hours |

Treat CAPTCHA as a **circuit-breaker signal**, not just a per-request annoyance. If your scraper triggers CAPTCHA on every request to a host, the host has already classified your traffic; pushing through with solvers will just drain budget. See [Bot Detection Internals](09-bot-detection-internals.md) for what's likely tripping it.

## 16. Cost Analysis

A back-of-envelope for a high-volume scraper hitting a reCAPTCHA-protected endpoint:

| Volume | Cost @ $2/1000 | Cost @ $5/1000 (Enterprise/FunCaptcha) |
|--------|----------------|----------------------------------------|
| 10,000/day | $0.60/day, $18/month | $1.50/day, $45/month |
| 100,000/day | $6/day, $180/month | $15/day, $450/month |
| 1,000,000/day | $60/day, $1,800/month | $150/day, $4,500/month |

That's just CAPTCHA. Add residential proxies (commonly $5–$15/GB; a million HTML page loads can be 200–500 GB) and the bill compounds fast. A scraper crossing $1k/month in CAPTCHA costs alone is a strong signal to revisit the architecture: are you hitting the site too often, can you move to an authenticated API (paid or partner), can you cache better, can you batch?

Solver costs also matter for **failure-rate accounting**. A 10% solver failure rate on Enterprise-tier solving (the pricier ones) doubles your effective per-success cost because you re-submit. Track success ratio per CAPTCHA type per target and reroute to a different solver if one consistently underperforms.

## 17. Ethics

Where CAPTCHA solving is reasonable:

- **Your own properties.** You forgot to whitelist your own scraper IP and need to pull data from your own dashboard during an incident.
- **Consented automation.** A client paid you to monitor their listings on their behalf; they have an account; CAPTCHA shows up because the third-party system can't tell the difference between automation and the client logging in from a new device.
- **Accessibility.** Buster exists because audio CAPTCHAs are bad for users with disabilities, and the alternative for them is "give up on the form." Tools that help an actual human pass a CAPTCHA they're entitled to pass are different from tools that pretend to be a human at industrial scale.
- **Security research with disclosure.** Demonstrating a bypass for a CVE-style report is a legitimate use as long as you stay within scope and disclose responsibly.

Where it's not:

- **Mass account creation.** Spinning up thousands of accounts to spam, scrape protected feeds, or game free-tier limits.
- **Account takeover.** Solving CAPTCHA to brute-force or credential-stuff.
- **Targeted financial fraud.** Solving CAPTCHA to defeat anti-fraud at checkout, ticket-buying bots that beat real users to purchases, sneakerbots, etc.
- **Platforms whose ToS forbid it and where you have no consent or partnership.** Google, LinkedIn, and Twitter/X all have explicit anti-automation terms, and solving their CAPTCHAs is an active circumvention.

The line that holds up under cross-examination is whether **a real user with a legitimate purpose could do what you're automating**. If yes, you're just a (gray-zone) accessibility bypass. If no — if the only point of your automation is doing something at scale that one human couldn't reasonably do — you're across the line, regardless of the technical sophistication of the solver.

---

## Related

- [HTTP Scraping Fundamentals](06-http-scraping-fundamentals.md)
- [Headless Browsers](07-headless-browsers.md)
- [Proxies and IP Rotation](08-proxies-and-ip-rotation.md)
- [Bot Detection Internals](09-bot-detection-internals.md)
- [API Reverse-Engineering](../reverse-engineering/13-api-reverse-engineering.md)
- [Production Scraping Hygiene and Legal](../reverse-engineering/14-production-scraping-hygiene-and-legal.md)

## References

**Provider documentation:**

- Google reCAPTCHA — [developers.google.com/recaptcha](https://developers.google.com/recaptcha)
- Google reCAPTCHA Enterprise — [cloud.google.com/recaptcha/docs](https://cloud.google.com/recaptcha/docs)
- hCaptcha — [docs.hcaptcha.com](https://docs.hcaptcha.com)
- Cloudflare Turnstile — [developers.cloudflare.com/turnstile](https://developers.cloudflare.com/turnstile)
- Arkose Labs / FunCaptcha — [arkoselabs.com](https://www.arkoselabs.com)
- Geetest — [docs.geetest.com](https://docs.geetest.com)
- AWS WAF CAPTCHA action — [docs.aws.amazon.com/waf/latest/developerguide/waf-captcha.html](https://docs.aws.amazon.com/waf/latest/developerguide/waf-captcha.html)

**Solver service APIs:**

- 2Captcha — [2captcha.com/2captcha-api](https://2captcha.com/2captcha-api)
- CapSolver — [docs.capsolver.com](https://docs.capsolver.com)
- Anti-Captcha — [anti-captcha.com/apidoc](https://anti-captcha.com/apidoc)
- DeathByCaptcha — [deathbycaptcha.com/api](https://www.deathbycaptcha.com/api)
- CapMonster Cloud — [capmonster.cloud/en/apidoc](https://capmonster.cloud/en/apidoc)

**Open-source / self-hosted:**

- Buster (audio reCAPTCHA solver browser extension) — [github.com/dessant/buster](https://github.com/dessant/buster)

**Research papers on CAPTCHA bypass:**

- Bock, Patel, Hughey, Levin — "unCaptcha: A Low-Resource Defeat of reCAPTCHA's Audio Challenge" — USENIX WOOT 2017.
- Bock, Patel, Hughey, Levin — "unCaptcha2" follow-up, 2019.
- Sivakorn, Polakis, Keromytis — "I'm Not a Human: Breaking the Google reCAPTCHA" — Black Hat Asia / IEEE EuroS&P 2016 (picture-based reCAPTCHA defeat).
- Hossen, Mansurov, Mendoza, Hei — "An Object Detection based Solver for Google's Image reCAPTCHA v2," RAID 2020.
- Akrout et al. — "Hacking Google reCAPTCHA v3 using Reinforcement Learning," 2019 arXiv preprint (2019.03.01516).

**Legal background referenced:**

- Google Terms of Service — [policies.google.com/terms](https://policies.google.com/terms)
- hiQ Labs v. LinkedIn — Ninth Circuit decisions and 2022 remand.
- Computer Fraud and Abuse Act (18 U.S.C. § 1030).
