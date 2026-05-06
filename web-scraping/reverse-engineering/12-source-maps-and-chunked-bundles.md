---
title: "Source Maps and Chunked Bundles — Practical Use"
date: 2026-05-06
updated: 2026-05-06
tags: [source-maps, webpack, vite, chunks, javascript, reverse-engineering, deobfuscation]
---

# Source Maps and Chunked Bundles — Practical Use

**Date:** 2026-05-06 | **Updated:** 2026-05-06
**Tags:** `source-maps` `webpack` `vite` `chunks` `javascript` `reverse-engineering` `deobfuscation`

---

## Table of Contents

1. [Summary](#summary)
2. [What Source Maps Are](#1-what-source-maps-are)
3. [Source Map V3 Format](#2-source-map-v3-format)
4. [How Tools Discover Source Maps](#3-how-tools-discover-source-maps)
5. [Common Leak Patterns in Production](#4-common-leak-patterns-in-production)
6. [Probing a Target for Source Maps](#5-probing-a-target-for-source-maps)
7. [Webpack Chunk Model](#6-webpack-chunk-model)
8. [Reading the Webpack Manifest](#7-reading-the-webpack-manifest)
9. [Vite Chunk Model](#8-vite-chunk-model)
10. [Next.js Chunk Manifests](#9-nextjs-chunk-manifests)
11. [Practical Workflow — Fetch and Reconstruct](#10-practical-workflow--fetch-and-reconstruct)
12. [What You Actually Recover](#11-what-you-actually-recover)
13. [Operational Use Cases](#12-operational-use-cases)
14. [Defending Against This](#13-defending-against-this)
15. [Ethics and Legality](#14-ethics-and-legality)
16. [Related](#related)
17. [References](#references)

---

## Summary

A source map is a JSON sidecar file that maps a minified JavaScript line/column back to the original source — TypeScript, JSX, SCSS, original variable names, original file paths. Modern bundlers (webpack, Vite, Rollup, esbuild, swc) all emit them. Browsers load them from `//# sourceMappingURL=` comments at the bottom of bundles, or from the `SourceMap:` HTTP header. They exist to make production stack traces debuggable.

When source maps are accidentally published to production — and they often are — anyone fetching the bundle can reconstruct the original frontend repo: every TypeScript file, every component, every internal API call, every leaked configuration constant. For a TypeScript backend developer, the relevant practical insight is: when reverse-engineering a SPA's API surface, *check for source maps first*. The job goes from "decode minified gibberish" to "read the same TypeScript the original developer wrote."

This doc covers the V3 format essentials, where leaks happen, how to probe for them, the webpack/Vite/Next.js chunk models you need to walk a full app, and the tooling (`unwebpack-sourcemap`, `sourcemapper`, `source-map-explorer`, `webcrack`) that automates the recovery.

---

## 1. What Source Maps Are

A source map is a JSON file that lets a browser or debugger translate a position in a *generated* file (your minified, tree-shaken, transpiled bundle) back to a position in the *original* source. Without it, a stack trace points at column 47,213 of `main.a8f3c2.js`. With it, the stack trace points at line 84 of `src/components/checkout/PaymentForm.tsx`.

The format is officially the "Source Map Revision 3 Proposal" — historically a Google Doc maintained jointly by Mozilla and Google engineers, now being formalized through the TC39 / WHATWG source-map specification effort.

Key facts:

- One `.map` file per generated file.
- Pure JSON, no binary.
- The hardest field is `mappings`: a base64 VLQ-encoded sequence of integers describing the line/column correspondence between generated and source code.
- Optional but extremely common: `sourcesContent`, an inline copy of every original source file, embedded directly in the JSON. This is what turns a source map from a debugging aid into a full repo recovery vector.

For a backend dev, treat a `.map` file as: "the developer's `git ls-files` output for the frontend, with file contents attached, often shipped to your `curl`."

---

## 2. Source Map V3 Format

The top-level JSON object has these fields:

```json
{
  "version": 3,
  "file": "main.a8f3c2.js",
  "sourceRoot": "",
  "sources": [
    "webpack:///./src/index.tsx",
    "webpack:///./src/api/client.ts",
    "webpack:///./src/components/Header.tsx"
  ],
  "sourcesContent": [
    "import React from 'react';\nimport ReactDOM from 'react-dom/client';\n...",
    "export const apiBase = process.env.REACT_APP_API_BASE;\n...",
    "..."
  ],
  "names": ["React", "ReactDOM", "render", "apiBase", "fetch"],
  "mappings": "AAAA;AACA,SAASA,EAAEC,CAAC..."
}
```

Field by field:

- **`version`** — always `3` for current files. Earlier revisions exist but are obsolete.
- **`file`** — the generated file this map describes (informational).
- **`sourceRoot`** — optional prefix to prepend to each `sources[]` entry. Often empty or `""`.
- **`sources`** — array of original file paths. May be project-relative (`./src/foo.ts`), webpack-prefixed (`webpack:///./src/foo.ts`), or absolute paths from the dev's machine (`/Users/alice/proj/src/foo.ts` — yes, this leaks too).
- **`sourcesContent`** — array parallel to `sources[]` containing the original file contents inline. **This is the field that makes source maps a recovery vector**. When it's present, you do not need to fetch anything else; the original code is already in your hand.
- **`names`** — array of original symbol names referenced from `mappings`.
- **`mappings`** — the actual line/column correspondence, encoded as semicolon-separated groups (one per generated line) of comma-separated VLQ tuples.

### VLQ encoding (just enough to know what you're looking at)

VLQ stands for "variable-length quantity." It encodes a signed integer in a base64 string of arbitrary length. Each base64 character contributes 5 bits of the number plus 1 continuation bit; the first character also reserves 1 bit for the sign.

Each segment in `mappings` is a tuple of 1, 4, or 5 VLQ-encoded integers:

- 1 field: generated column
- 4 fields: + source file index, source line, source column
- 5 fields: + name index

All numbers are *deltas* relative to the previous segment, which keeps the encoding small.

You almost never decode VLQ by hand. The `source-map` npm package does it for you:

```bash
npm install source-map
```

```ts
import { SourceMapConsumer } from 'source-map';

const raw = await fetch('https://example.com/static/main.a8f3c2.js.map').then(r => r.json());
await SourceMapConsumer.with(raw, null, async (consumer) => {
  // consumer.sourceContentFor('webpack:///./src/api/client.ts') returns the raw source
  for (const source of consumer.sources) {
    console.log(source);
  }
});
```

For practical use, you rarely need `mappings` at all — you only need `sources` and `sourcesContent`.

---

## 3. How Tools Discover Source Maps

Three discovery mechanisms, in order of how often each is used in production:

### 3.1 `sourceMappingURL` comment

The bundler appends a comment as the last line of the generated file:

```js
//# sourceMappingURL=main.a8f3c2.js.map
```

The path is interpreted relative to the generated file's URL. If the bundle is at `https://example.com/static/js/main.a8f3c2.js`, then `main.a8f3c2.js.map` resolves to `https://example.com/static/js/main.a8f3c2.js.map`.

Inline data URIs are also valid:

```js
//# sourceMappingURL=data:application/json;base64,eyJ2ZXJzaW9uIjozLC...
```

Inline maps make the `.map` file unnecessary because the entire JSON is base64-encoded into the bundle itself. This is `devtool: 'inline-source-map'` in webpack and `build.sourcemap: 'inline'` in Vite.

### 3.2 `SourceMap` HTTP response header

Servers can attach the map URL via response header:

```
SourceMap: /static/js/main.a8f3c2.js.map
```

This is older spec language; some tooling still emits the deprecated header form:

```
X-SourceMap: /static/js/main.a8f3c2.js.map
```

DevTools and `curl -I` will both reveal these. They are useful when the server wants to deliver the map separately (e.g. behind auth) without modifying the bundle.

### 3.3 Manual override in DevTools

In Chrome DevTools → Sources panel → right-click any loaded JS file → "Add source map…" lets you paste a URL or local path. Useful when:

- The site removed the `sourceMappingURL` comment but the `.map` is still served.
- You have a local copy of the map and want to debug the live bundle.
- You're testing a hypothesis about which `.map` file pairs with which bundle.

---

## 4. Common Leak Patterns in Production

Every modern bundler ships source maps to production by accident at least once. The patterns:

### 4.1 Create React App default

CRA ships source maps to production unless you set:

```bash
GENERATE_SOURCEMAP=false npm run build
```

Many teams either do not know this flag exists or deliberately leave it on for Sentry. The result: a `build/static/js/main.<hash>.js.map` next to every chunk.

### 4.2 Vite "I just wanted to debug something" mistake

Vite's default is `build.sourcemap: false`. But during a debugging session a dev sets:

```ts
// vite.config.ts
export default defineConfig({
  build: {
    sourcemap: true, // or 'inline' — even worse for size
  },
});
```

…and forgets to revert it. The deploy goes out with maps included.

### 4.3 Next.js explicit opt-in

Next.js does not ship browser source maps by default. The opt-in is:

```js
// next.config.js
module.exports = {
  productionBrowserSourceMaps: true,
};
```

Teams enable it for Sentry and forget to pair it with anything that prevents public access.

### 4.4 Webpack `devtool` shipped to prod

The classic mistake: a single `webpack.config.js` with `devtool: 'source-map'` used for both dev and prod. The webpack docs explicitly warn against this. Variants like `eval-source-map` are dev-only by intent but still occasionally ship.

### 4.5 The Sentry / DataDog / Bugsnag "double-deploy" mistake

The recommended Sentry workflow is: build with source maps, *upload* the maps to Sentry, then *delete* the `.map` files from the public bundle so they are not served. Many teams do steps 1 and 2 but skip step 3. The maps end up both in Sentry *and* on the public CDN.

Sentry's docs explicitly call this out and recommend deleting maps from the deployed assets after upload.

### 4.6 Robots.txt and sitemap leaks

A sloppy `robots.txt` that reaches into `static/` or a sitemap generator that crawls all asset paths will list `.map` files. Search engines and the Internet Archive then index them, and removal from the origin does not always remove them from the archive.

### 4.7 Web archive persistence

Even after a site fixes its source-map exposure, copies often remain in:

- The Wayback Machine (`web.archive.org`)
- Google's web cache (deprecated but historically rich)
- Common Crawl

Reverse engineers routinely fetch maps from the Wayback Machine when the live site has scrubbed them.

---

## 5. Probing a Target for Source Maps

A short checklist for any SPA you want to study.

### 5.1 Inspect the bundle's last line

```bash
curl -s https://example.com/static/js/main.a8f3c2.js | tail -c 200
```

Look for `//# sourceMappingURL=...` on the final non-empty line. If present, that path resolves to the map.

### 5.2 Check response headers

```bash
curl -sI https://example.com/static/js/main.a8f3c2.js | grep -i 'sourcemap\|source-map'
```

Catches `SourceMap:` and `X-SourceMap:` headers.

### 5.3 Try `.map` directly

The most common naming convention is just `<bundle>.map`:

```bash
curl -sI https://example.com/static/js/main.a8f3c2.js.map
```

A `200 OK` and `Content-Type: application/json` confirms the map exists, even if the bundle's `sourceMappingURL` comment was stripped.

### 5.4 `unwebpack-sourcemap`

Pulls source files out of a `.map` and reconstructs a directory tree:

```bash
git clone https://github.com/rarecoil/unwebpack-sourcemap
cd unwebpack-sourcemap
pip install -r requirements.txt
python unwebpack_sourcemap.py --local main.a8f3c2.js.map ./recovered/
# or remote
python unwebpack_sourcemap.py --remote https://example.com/static/js/main.a8f3c2.js.map ./recovered/
```

Output is a directory tree mirroring `sources[]`, with each file containing the matching `sourcesContent[]` entry.

### 5.5 `sourcemapper`

A Go tool that does the same job, often easier to ship in a recon pipeline:

```bash
go install github.com/denandz/sourcemapper@latest
sourcemapper -url https://example.com/static/js/main.a8f3c2.js.map -output ./recovered
```

`sourcemapper` also accepts a JS URL directly and follows the `sourceMappingURL` comment for you.

### 5.6 Manual one-liner

If you just want to inspect what's inside without installing anything:

```bash
curl -s https://example.com/static/js/main.a8f3c2.js.map \
  | jq '{version, sourceCount: (.sources | length), hasContent: (.sourcesContent != null)}'
```

Then to dump a single original file:

```bash
curl -s https://example.com/static/js/main.a8f3c2.js.map \
  | jq -r '.sourcesContent[0]' > recovered-0.ts
```

---

## 6. Webpack Chunk Model

Modern SPAs do not ship one bundle. They ship dozens of chunks, and you cannot recover the full app from one map. Understanding the chunk model is what lets you walk the entire surface.

A typical webpack 5 production build emits these chunk classes:

### 6.1 Runtime chunk

A small file (a few KB) containing the webpack runtime: the `__webpack_require__` function, the chunk loader, and the manifest mapping `chunkId → URL`. With `optimization.runtimeChunk: 'single'`, this lives in a file named like `runtime.<hash>.js`. With `'multiple'`, there is one runtime per entry.

This file is the index. If you have it, you can enumerate every other chunk's URL.

### 6.2 Vendor chunks

Created by `splitChunks.cacheGroups.vendor` (the default config splits anything from `node_modules` into a `vendors~` chunk). Filename pattern: `vendors-<hash>.js` or `<groupName>.<hash>.js`. Often the largest single file in the build.

### 6.3 Route chunks / dynamic-import chunks

Every `import('./Foo')` in the code becomes its own chunk. React Router's `lazy()` produces these, as does Next.js's `next/dynamic` and Vue's `defineAsyncComponent`. Filename pattern under default config: `<chunkId>.<hash>.js` or, with `webpackChunkName` magic comments, `<name>.<hash>.js`.

For reverse engineering, **every dynamic-import chunk is a route or feature worth investigating**. The list of chunks is a surface map of the application's features.

### 6.4 Common chunks

When two routes share a non-vendor module, webpack pulls it into a separate "common" chunk. Naming follows whatever `splitChunks` config is in effect.

### 6.5 Filename hashing

Webpack uses `[contenthash]` (preferred, content-addressed) or `[chunkhash]` (per-chunk hash) in production. The hash means filenames change between deploys. A bookmark of yesterday's URL will 404 today.

### 6.6 The runtime's chunk loader

Inside the runtime chunk, a function looks something like:

```js
__webpack_require__.u = function (e) {
  return "static/js/" + e + "." + ({
    "143": "a3f8c19b",
    "267": "f0192bba",
    "412": "77d3ee2c",
    "568": "3b94a1ee"
  }[e]) + ".chunk.js";
};
```

That object literal is the manifest. It maps every chunk ID to the hash needed to construct its URL. Pulling it out gives you the full URL list of every chunk in the deployment. Once you have that list, point `sourcemapper` at each one.

---

## 7. Reading the Webpack Manifest

In webpack 5, the chunk manifest is *embedded* in the runtime chunk by default. To extract it:

```bash
# Fetch the runtime chunk
curl -s https://example.com/static/js/runtime.abc123.js > runtime.js

# Pretty-print and grep for the chunk-to-hash mapping
node -e 'const fs=require("fs");const s=fs.readFileSync("runtime.js","utf8");const m=s.match(/\{(?:"\d+":"[a-f0-9]+",?)+\}/g);console.log(m);'
```

The regex above is rough but works as a starting point. For real work, parse the AST with `acorn` or `@babel/parser` and walk for the object literal inside the `__webpack_require__.u` function.

Some builds also emit a separate file:

- `manifest.json` in the public folder (when using `webpack-manifest-plugin`)
- `asset-manifest.json` (Create React App's default — present at `/asset-manifest.json` on the deployed site)

CRA's `asset-manifest.json` is especially useful — it lists every emitted file and is publicly served by default:

```bash
curl -s https://example.com/asset-manifest.json | jq .
```

Output looks like:

```json
{
  "files": {
    "main.css": "/static/css/main.077fb8be.css",
    "main.js": "/static/js/main.a8f3c2.js",
    "static/js/453.85a98a4d.chunk.js": "/static/js/453.85a98a4d.chunk.js",
    "...": "..."
  },
  "entrypoints": [
    "static/js/runtime-main.182069b7.js",
    "static/js/main.a8f3c2.js"
  ]
}
```

That's the entire bundle list, no parsing required.

---

## 8. Vite Chunk Model

Vite produces a similar but Rollup-shaped chunk graph. Differences worth knowing:

- Default output dir: `dist/assets/`.
- Filenames: `<name>-<hash>.js` for entry/chunks, `<name>-<hash>.css` for styles.
- Dynamic imports each become their own chunk by default — Rollup emits one chunk per dynamic-import call site unless you configure `manualChunks`.
- No separate runtime chunk like webpack. The runtime is part of the entry chunk.

Vite emits `dist/.vite/manifest.json` when you set `build.manifest: true` in the config. That file maps original input files to the emitted output filenames:

```json
{
  "src/main.ts": {
    "file": "assets/main-a8f3c2.js",
    "isEntry": true,
    "imports": ["_vendor-c0a3.js"],
    "css": ["assets/main-3e91.css"]
  },
  "src/views/Settings.vue": {
    "file": "assets/Settings-77d3.js",
    "isDynamicEntry": true
  }
}
```

If a Vite app's deploy includes `manifest.json` in the public path, you've got the chunk graph for free.

For source maps specifically, Vite's `build.sourcemap` option accepts:

- `false` — no maps (default, safe)
- `true` — separate `.map` files emitted next to bundles
- `'inline'` — base64-embedded into bundles (huge file size, but the map cannot be deleted independently)
- `'hidden'` — emit `.map` files but do not append `sourceMappingURL` comments to bundles

`'hidden'` is a partial mitigation: tools that just `tail` the bundle won't find the map, but trying `<bundle>.map` directly still works.

---

## 9. Next.js Chunk Manifests

Next.js exposes its own JSON manifests in known locations on a deployed site:

- `_next/static/<buildId>/_buildManifest.js` — lists every chunk associated with each route.
- `_next/static/<buildId>/_ssgManifest.js` — lists all statically-generated routes.

The `<buildId>` itself is exposed in the page HTML (`__NEXT_DATA__` script tag, `buildId` field):

```bash
curl -s https://example.com/ | grep -oE '"buildId":"[^"]+"' | head -1
# "buildId":"AbC123xYz"
```

Then:

```bash
curl -s https://example.com/_next/static/AbC123xYz/_buildManifest.js
```

These files were never intended as private — they're served as part of the framework's runtime. But for an attacker or researcher, they enumerate every page chunk and every code-split route on the site.

If `productionBrowserSourceMaps: true` was set, every chunk listed in these manifests has a corresponding `.js.map` available.

---

## 10. Practical Workflow — Fetch and Reconstruct

A repeatable workflow for studying a target SPA:

### Step 1 — Identify all chunked JS files

Open Chrome DevTools → Network → filter by JS → reload the page → click around to trigger lazy routes. The Network panel logs every chunk URL. You can also pull the list from `asset-manifest.json` (CRA), `manifest.json` (Vite), or `_buildManifest.js` (Next.js) without clicking through the UI.

Right-click in the Network panel → "Save all as HAR" gives you a JSON file with every URL.

### Step 2 — Try `.map` for each

```bash
jq -r '.log.entries[].request.url' export.har \
  | grep '\.js$' \
  | sort -u \
  | while read url; do
      map="${url}.map"
      code=$(curl -s -o /dev/null -w "%{http_code}" "$map")
      echo "$code $map"
    done
```

Anything returning `200` is fair game.

### Step 3 — Check for `sourcesContent`

This is the make-or-break field. `sourcesContent` present → original code recovered. Absent → you only have file paths.

```bash
curl -s "$map" | jq 'has("sourcesContent")'
```

In practice, `sourcesContent` is present in 95%+ of public maps, because every major bundler inlines it by default.

### Step 4 — When `sourcesContent` is missing

Rare, but happens when the bundler is configured with `sourcesContent: false` (a webpack option) for size reasons. In that case, the map's `sources` array contains paths like `webpack:///./src/api/client.ts`. The `webpack:///` prefix is just webpack's internal scheme — it does not correspond to an HTTP URL.

Sometimes the dev's CDN still serves the original source files (rare, usually a dev server left exposed). More often you fall back to webcrack (next step).

### Step 5 — Visualize what's inside each chunk

```bash
npm install -g source-map-explorer
source-map-explorer https://example.com/static/js/main.a8f3c2.js
```

`source-map-explorer` opens a treemap of every original module by size, showing what makes up the chunk. Useful for:

- Spotting which packages a target uses.
- Identifying internal modules worth reading first (`src/api/`, `src/auth/`, `src/config/`).
- Comparing two deploys to see what changed.

### Step 6 — When source maps are missing entirely

Use `webcrack`:

```bash
npm install -g webcrack
webcrack main.a8f3c2.js -o ./recovered/
```

`webcrack` does heuristic deobfuscation:

- Splits a webpack bundle back into its constituent modules.
- Renames `__webpack_require__` calls to logical require statements.
- Inverts common obfuscator-io and javascript-obfuscator transforms.
- Handles control-flow flattening and string array obfuscation.

Output is JavaScript, not TypeScript — types are gone, file paths are guessed — but it's readable code. Pair it with manual inspection.

---

## 11. What You Actually Recover

When source maps are present with `sourcesContent`:

- **Original TypeScript / JSX / TSX**, with types intact.
- **Original CSS / SCSS / Tailwind class lists** (from CSS source maps, separate from JS).
- **Original file paths**, often revealing the developer's directory layout, occasionally absolute paths that include the developer's username (`/Users/alice/work/acme-frontend/...`).
- **Original variable and function names** (when the `names` array is populated and `sourcesContent` is present, you don't even need it — the code in `sourcesContent` already has the original names).
- **Comments**, including TODOs, FIXMEs, and occasionally jokes about colleagues.

What you do not recover:

- Server-side code (this is browser-bundle-only).
- Assets that were not bundled (PNG/SVG hosted separately).
- Environment variables that were not inlined at build time. CRA's `REACT_APP_*` and Vite's `VITE_*` variables *are* inlined and become string literals in the recovered code. Variables read at runtime (e.g. via `/api/config`) are not in the bundle.

The practical experience for a TS/Node dev: `cd recovered/src && tree && code .`, and you're reading the same code the original team writes daily.

---

## 12. Operational Use Cases

For a backend developer studying an external SPA — your own dependency, a competitor's app, a target you have permission to audit:

### 12.1 Find API endpoints

Reading minified code for endpoint patterns is painful. Reading the original `src/api/client.ts` is fast. Look for:

- Axios `baseURL` or `fetch` URL templates.
- Hand-written endpoint constants (`const ENDPOINTS = { ... }`).
- React Query keys that often double as endpoint paths (`useQuery(['/api/users', id], ...)`).
- TRPC router definitions, if applicable.

### 12.2 Find embedded keys

Search the recovered tree for:

```bash
grep -REn "(api[_-]?key|token|secret|sentry|amplitude|segment|stripe[_-]?pk)" recovered/
```

Common findings:

- Public Stripe / Mixpanel / Segment / Amplitude keys (these are designed to be public, but tell you which services are integrated).
- Sentry DSNs (also designed to be public, but they reveal the org/project name).
- Algolia search-only API keys (designed public, but the index name can be useful).
- Sometimes mistakenly-included secret keys, GraphQL admin tokens, internal service URLs.

### 12.3 Identify backend route patterns

By reading the frontend, you reverse the *server's* route structure. If the frontend calls `GET /api/v2/orgs/:orgId/teams/:teamId/members?cursor=...`, you've learned both the path shape and the pagination style. This is faster than fuzzing.

### 12.4 Map the feature surface

`ls recovered/src/features/` and `ls recovered/src/routes/` give you the canonical list of product features. Every dynamic-import chunk corresponds to a route worth visiting in the live app.

### 12.5 Find feature flags and kill-switches

Look for LaunchDarkly, Unleash, ConfigCat, Statsig, or homegrown flag systems:

```bash
grep -REn "useFlag|isEnabled|getFlag|launchdarkly" recovered/src/
```

The list of flag names tells you what's in flight.

---

## 13. Defending Against This

If you operate the site and want to prevent source-map exposure:

### 13.1 Don't ship `.map` files to public hosting

For Sentry / DataDog / Bugsnag workflows:

- Build with source maps.
- Upload to the error-tracking provider.
- Delete the `.map` files from the deploy artifact before rolling out.

Sentry's CLI supports this in the upload command: upload, then remove. Sentry's docs explicitly warn that publishing maps publicly defeats the value of uploading to Sentry in the first place.

### 13.2 If you must ship them, restrict access

Options:

- Serve `.map` files only to authenticated users (e.g. internal employees signed into a SSO proxy).
- Allow-list internal IPs.
- Put them behind basic auth on a separate subdomain.
- Use webpack's `devtool: 'hidden-source-map'` (or Vite's `'hidden'`) to omit the `sourceMappingURL` comment, then serve maps under unguessable paths. (Note: this is security-by-obscurity — paths in the manifest may still leak.)

### 13.3 Build pipeline guardrails

- A CI step that fails the build if any `.map` file ends up in the production artifact directory.
- A post-deploy probe that fetches `<known-bundle>.map` and alerts if it returns 200.

### 13.4 Rotate exposed secrets

If a leak happens, rotate any embedded secret credentials (even "public" ones if there's any chance of misuse) and audit access logs for the `.map` URLs.

---

## 14. Ethics and Legality

### 14.1 Reading public files is reading public files

Fetching a `.map` file from a public URL is no different from fetching the HTML or JS that references it. The HTTP server serves it without authentication; that's a deliberate (if accidental) operator choice. In US CFAA terms, this is not unauthorized access — there is no access control to circumvent.

The Wayback Machine and Google's index treat `.map` files the same as any other public asset. Crawling a public URL is not unauthorized access.

### 14.2 But: what you do with the recovered material matters

- **Reading the code to understand a public API** — generally fine.
- **Republishing the recovered source code as your own** — copyright infringement.
- **Using it to build a substantially similar competing product** — depends on jurisdiction; copyright applies even to source recovered through legitimate means.
- **Using a recovered API key to access a service you're not authorized to use** — clearly unauthorized access, regardless of how the key was obtained. CFAA and similar laws apply.
- **Using a recovered key to access *your own* account on a service** — fine; you're already authorized.

### 14.3 Bug-bounty context

Most bug-bounty programs *explicitly* allow source-map analysis as part of reconnaissance. HackerOne, Bugcrowd, and Intigriti programs typically scope this in. Reporting "your `.map` files are publicly served and contain X" is a common low-to-medium-severity finding.

### 14.4 Practical posture

If you are:

- Studying a dependency to understand integration points: fine.
- Auditing an app you have permission (written or implied via bug-bounty scope) to test: fine.
- Curious about a competitor's frontend architecture: fine to read; do not republish.
- Trying to access a service using credentials you found: stop.

When in doubt, treat source maps the same way you'd treat any other production artifact: just because the door is unlocked does not mean you should rummage through every drawer.

---

## Related

- [Chrome DevTools for Scraping](11-chrome-devtools-for-scraping.md)
- [API Reverse-Engineering](13-api-reverse-engineering.md)
- [Headless Browsers](../extraction/07-headless-browsers.md)
- [TypeScript learning path](../../typescript/INDEX.md)

---

## References

### Specs

- Source Map Revision 3 Proposal — original Mozilla/Google Google Doc, archived: <https://sourcemaps.info/spec.html>
- TC39 Source Map specification work: <https://github.com/tc39/source-map>
- Source Map specification repository: <https://github.com/source-map/source-map-spec>

### Bundler documentation

- Webpack `devtool` option: <https://webpack.js.org/configuration/devtool/>
- Webpack code splitting: <https://webpack.js.org/guides/code-splitting/>
- Webpack `splitChunks` config: <https://webpack.js.org/plugins/split-chunks-plugin/>
- Vite `build.sourcemap` option: <https://vitejs.dev/config/build-options.html#build-sourcemap>
- Vite manifest: <https://vitejs.dev/guide/backend-integration.html>
- Next.js `productionBrowserSourceMaps`: <https://nextjs.org/docs/app/api-reference/next-config-js/productionBrowserSourceMaps>
- Create React App advanced configuration (including `GENERATE_SOURCEMAP`): <https://create-react-app.dev/docs/advanced-configuration/>

### Recovery tooling

- `unwebpack-sourcemap`: <https://github.com/rarecoil/unwebpack-sourcemap>
- `sourcemapper`: <https://github.com/denandz/sourcemapper>
- `source-map-explorer` (npm): <https://www.npmjs.com/package/source-map-explorer>
- `source-map` (npm, the canonical V3 parser): <https://www.npmjs.com/package/source-map>
- `webcrack`: <https://github.com/j4k0xb/webcrack>

### Operational guidance

- Sentry source maps upload guide: <https://docs.sentry.io/platforms/javascript/sourcemaps/>
- Sentry CLI source maps reference: <https://docs.sentry.io/cli/sourcemaps/>
- DataDog browser source maps: <https://docs.datadoghq.com/real_user_monitoring/guide/upload-javascript-source-maps/>
