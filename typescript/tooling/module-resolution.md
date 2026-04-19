---
title: "Module Resolution Deep Dive"
date: 2026-04-19
updated: 2026-04-19
tags: [typescript, node, esm, cjs, module-resolution, exports, imports, publishing]
---

# Module Resolution Deep Dive

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `typescript` `node` `esm` `cjs` `module-resolution` `exports` `imports` `publishing`

## Table of Contents

- [Summary](#summary)
- [How Node Resolves Imports](#how-node-resolves-imports)
  - [CommonJS Resolution Algorithm](#commonjs-resolution-algorithm)
  - [ESM Resolution Algorithm](#esm-resolution-algorithm)
  - [Key Differences: CJS vs ESM Resolution](#key-differences-cjs-vs-esm-resolution)
- [`.js` Extensions in ESM](#js-extensions-in-esm)
  - [Why You Write `.js` When the Source Is `.ts`](#why-you-write-js-when-the-source-is-ts)
  - [`moduleResolution: "node16"` / `"nodenext"`](#moduleresolution-node16--nodenext)
- [`package.json` `type` Field](#packagejson-type-field)
  - [`.mjs` and `.cjs` Overrides](#mjs-and-cjs-overrides)
- [`package.json` `exports` Field](#packagejson-exports-field)
  - [Conditional Exports](#conditional-exports)
  - [Subpath Exports](#subpath-exports)
  - [Subpath Patterns](#subpath-patterns)
  - [Real-World `exports` Map](#real-world-exports-map)
- [Dual CJS/ESM Publishing](#dual-cjsesm-publishing)
  - [The Practical Setup](#the-practical-setup)
  - [`types` Placement](#types-placement)
- [`imports` Field (Node Subpath Imports)](#imports-field-node-subpath-imports)
  - [Comparison to tsconfig `paths`](#comparison-to-tsconfig-paths)
- [Common Resolution Failures](#common-resolution-failures)
  - [`Cannot find module`](#cannot-find-module)
  - [`ERR_MODULE_NOT_FOUND`](#err_module_not_found)
  - [`ERR_REQUIRE_ESM`](#err_require_esm)
  - [Diagnosis Cheat Sheet](#diagnosis-cheat-sheet)
- [Java Parallel: Classpath and Module Path](#java-parallel-classpath-and-module-path)
- [Related](#related)
- [References](#references)

---

## Summary

"Why can't I import this package?" is the most common question in the Node/TypeScript ecosystem. The answer is always the same: **the resolution algorithm could not find a file at the path you specified, under the rules of the module system you are using.** CJS and ESM have different rules, TypeScript adds its own layer on top, and `package.json` fields like `exports`, `type`, and `imports` modify the algorithm further.

This doc walks through every piece of the resolution pipeline so that the next time an import fails, you can diagnose it from first principles instead of randomly toggling tsconfig flags.

If you come from Java: think of this as the equivalent of understanding classpath resolution, module-info.java, and the difference between automatic modules and named modules. The concepts parallel more closely than you'd expect.

---

## How Node Resolves Imports

### CommonJS Resolution Algorithm

When Node encounters `require("foo")`, it follows this algorithm:

```text
1. Is "foo" a core module? (fs, path, http, etc.)
   -> Yes: return the built-in module. Done.

2. Does "foo" start with ./ or ../ or / ?
   -> Yes: resolve as a FILE, then as a DIRECTORY.

3. Otherwise: resolve as a PACKAGE from node_modules.

FILE resolution (for require("./foo")):
   a. Try exact path:       ./foo
   b. Try with extensions:  ./foo.js, ./foo.json, ./foo.node
   c. Try as directory:     ./foo/package.json "main" field
   d. Try directory index:  ./foo/index.js, ./foo/index.json, ./foo/index.node

PACKAGE resolution (for require("foo")):
   a. Walk up from the current file's directory:
      ./node_modules/foo
      ../node_modules/foo
      ../../node_modules/foo
      ... (up to filesystem root)
   b. For each node_modules/foo found:
      - Check package.json "exports" field first (if it exists, it controls everything)
      - Check package.json "main" field
      - Fall back to index.js
```

Key CJS behaviors:
- **Extension probing**: `require("./utils")` tries `./utils.js`, `./utils.json`, `./utils.node` automatically.
- **Directory index**: `require("./lib")` falls back to `./lib/index.js`.
- **Synchronous**: `require()` blocks the event loop while it searches.

### ESM Resolution Algorithm

When Node encounters `import foo from "foo"` (in an ESM context), the algorithm changes significantly:

```text
1. Is "foo" a core module? (with or without "node:" prefix)
   -> Yes: return the built-in module. Done.

2. Does "foo" start with ./ or ../ or / ?
   -> Yes: resolve as a FILE — MANDATORY full specifier.
      - NO extension probing. "./utils" will NOT find "./utils.js".
      - NO directory index. "./lib" will NOT find "./lib/index.js".
      - You MUST write: import "./utils.js" or import "./lib/index.js"

3. Does "foo" start with # ?
   -> Yes: resolve via the "imports" field in the nearest package.json.

4. Otherwise: resolve as a PACKAGE from node_modules.
   a. Walk up from the current file's directory (same as CJS).
   b. For each node_modules/foo found:
      - Check package.json "exports" field first (takes full control).
      - If no "exports": check "main" field.
      - If no "main": try index.js (but this is legacy fallback).
```

### Key Differences: CJS vs ESM Resolution

| Behavior | CJS (`require`) | ESM (`import`) |
|---|---|---|
| Extension probing | Yes (`.js`, `.json`, `.node`) | **No** — full specifier required |
| Directory `index.js` | Yes | **No** — must be explicit |
| `__dirname`, `__filename` | Available | **Not available** (use `import.meta.url`) |
| JSON import | `require("./data.json")` works | Needs `assert { type: "json" }` or `with { type: "json" }` |
| Top-level `await` | Not allowed | Allowed |
| `exports` field | Respected (Node 12.11+) | Respected |
| Dynamic import | `require()` is always available | `import()` returns a Promise |

The ESM strictness is intentional. No extension probing means no ambiguity, which enables faster resolution and better static analysis.

---

## `.js` Extensions in ESM

### Why You Write `.js` When the Source Is `.ts`

This is the single most confusing thing for TypeScript developers new to ESM. Your source file is `utils.ts`, but your import statement says:

```typescript
// src/routes/user.ts
import { hashPassword } from "../utils/crypto.js"; // <-- .js, not .ts
```

Why? Because **TypeScript does not rewrite import specifiers during compilation.** When `tsc` compiles `user.ts` to `user.js`, the import path is emitted verbatim into the output:

```javascript
// dist/routes/user.js (compiled output)
import { hashPassword } from "../utils/crypto.js"; // identical to source
```

At runtime, Node is executing the `.js` files in `dist/`. It looks for `../utils/crypto.js` and finds it. If you had written `.ts` in the import, Node would look for a `.ts` file that doesn't exist in `dist/`.

The mental model:

```text
Source:   src/utils/crypto.ts   --->   tsc   --->   dist/utils/crypto.js
Import:       "../utils/crypto.js"      (passed through unchanged)
Runtime:  Node loads dist/utils/crypto.js  <--- matches the import path
```

TypeScript is smart enough to know that `import "../utils/crypto.js"` in a `.ts` file means "the `.ts` (or `.tsx`) file that will compile to this `.js` path." It resolves types from the source file during type-checking, but emits the `.js` path for runtime.

### `moduleResolution: "node16"` / `"nodenext"`

The `moduleResolution` setting in `tsconfig.json` controls which resolution algorithm TypeScript simulates during type-checking:

```json
{
  "compilerOptions": {
    "module": "node16",
    "moduleResolution": "node16"
  }
}
```

With `"node16"` or `"nodenext"`:
- TypeScript **enforces** the ESM resolution rules for files that Node would treat as ESM.
- You **must** include the `.js` extension on relative imports.
- You **cannot** import a directory (no implicit `index.js`).
- TypeScript checks the `exports` field in `package.json` when resolving bare specifiers.

With the older `"bundler"` or `"node"`:
- Extension probing is allowed (import `"./utils"` without `.js`).
- Directory index resolution works.
- The `exports` field may or may not be checked depending on the setting.

The practical advice: if you are targeting Node.js directly (no bundler), use `"node16"` or `"nodenext"`. If you use a bundler (Vite, Webpack, esbuild), `"bundler"` is appropriate because the bundler does its own resolution.

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

---

## `package.json` `type` Field

The `type` field determines the default module system for `.js` files in the package:

```json
{
  "name": "my-service",
  "type": "module"
}
```

| `type` value | `.js` files treated as | Default if omitted |
|---|---|---|
| `"module"` | ESM | - |
| `"commonjs"` | CJS | Yes (omitting = `"commonjs"`) |

This is the reason `require()` stops working when you add `"type": "module"` to your `package.json`. Every `.js` file in that package is now ESM. Node expects `import`/`export` syntax.

### `.mjs` and `.cjs` Overrides

File extensions override the `type` field, regardless of what it says:

| Extension | Always treated as | Ignores `type`? |
|---|---|---|
| `.mjs` | ESM | Yes |
| `.cjs` | CJS | Yes |
| `.js` | Depends on `type` field | No |

The TypeScript equivalents:

| Extension | Always treated as | Compiles to |
|---|---|---|
| `.mts` | ESM | `.mjs` |
| `.cts` | CJS | `.cjs` |
| `.ts` | Depends on `type` field | `.js` |

This means you can have a single CJS file in an otherwise ESM package:

```text
my-package/
├── package.json         ("type": "module")
├── src/
│   ├── index.ts         -> ESM (because type: module)
│   ├── server.ts        -> ESM
│   └── legacy-hook.cts  -> CJS (always, regardless of type field)
```

---

## `package.json` `exports` Field

The `exports` field is the modern replacement for `main`. When present, it **takes full control** over what consumers can import from your package. Anything not listed in `exports` is invisible.

### Conditional Exports

The `exports` field supports conditions that let Node (and TypeScript) pick the right entry point based on how the package is being consumed:

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs",
      "types": "./dist/types/index.d.ts",
      "default": "./dist/esm/index.js"
    }
  }
}
```

The conditions are evaluated **top to bottom, first match wins**:

| Condition | When it matches |
|---|---|
| `"types"` | TypeScript type resolution (must come first) |
| `"import"` | ESM `import` or dynamic `import()` |
| `"require"` | CJS `require()` |
| `"node"` | Any Node.js environment |
| `"browser"` | Bundler targeting browsers |
| `"default"` | Fallback for anything else |

**Important:** `"types"` must be listed **before** `"import"` and `"require"`. TypeScript resolves conditions in declaration order, and it stops at the first match. If `"import"` comes first, TypeScript picks the `.js` file and never sees the `.d.ts`.

### Subpath Exports

Expose specific entry points for deep imports:

```json
{
  "name": "my-lib",
  "exports": {
    ".": "./dist/index.js",
    "./utils": "./dist/utils.js",
    "./middleware": "./dist/middleware/index.js",
    "./package.json": "./package.json"
  }
}
```

Consumers can now write:

```typescript
import { createApp } from "my-lib";
import { retry } from "my-lib/utils";
import { cors } from "my-lib/middleware";
```

Anything **not** listed in `exports` is blocked:

```typescript
import { internal } from "my-lib/dist/internal.js";
// ERR_PACKAGE_PATH_NOT_EXPORTED
```

This is the encapsulation mechanism. Think of it like Java's `module-info.java` where `exports com.mylib.api` makes a package visible while internal packages remain hidden.

### Subpath Patterns

For libraries with many files (like icon sets or locale data), use glob-like patterns:

```json
{
  "name": "my-icons",
  "exports": {
    ".": "./dist/index.js",
    "./icons/*": "./dist/icons/*.js",
    "./locales/*": "./dist/locales/*.js"
  }
}
```

```typescript
import { ArrowIcon } from "my-icons/icons/arrow";
import en from "my-icons/locales/en";
```

The `*` maps one-to-one: `./icons/arrow` matches `./dist/icons/arrow.js`.

### Real-World `exports` Map

Here is a complete `exports` field for a library that supports CJS, ESM, and TypeScript consumers with separate build outputs:

```json
{
  "name": "@acme/sdk",
  "version": "2.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs",
      "default": "./dist/esm/index.js"
    },
    "./client": {
      "types": "./dist/types/client.d.ts",
      "import": "./dist/esm/client.js",
      "require": "./dist/cjs/client.cjs",
      "default": "./dist/esm/client.js"
    },
    "./middleware": {
      "types": "./dist/types/middleware.d.ts",
      "import": "./dist/esm/middleware.js",
      "require": "./dist/cjs/middleware.cjs",
      "default": "./dist/esm/middleware.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "package.json"],
  "main": "./dist/cjs/index.cjs",
  "module": "./dist/esm/index.js",
  "types": "./dist/types/index.d.ts"
}
```

The `main`, `module`, and top-level `types` are fallbacks for older tools that don't understand `exports`.

---

## Dual CJS/ESM Publishing

### The Practical Setup

Publishing a library that works for both `require()` and `import` consumers requires separate build outputs. Here is a minimal but production-ready setup:

**Directory structure:**

```text
my-lib/
├── src/
│   ├── index.ts
│   └── utils.ts
├── dist/
│   ├── esm/
│   │   ├── index.js       (ES modules)
│   │   └── utils.js
│   ├── cjs/
│   │   ├── index.cjs      (CommonJS)
│   │   └── utils.cjs
│   └── types/
│       ├── index.d.ts      (declarations)
│       └── utils.d.ts
├── tsconfig.json
├── tsconfig.esm.json
├── tsconfig.cjs.json
└── package.json
```

**tsconfig.esm.json:**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "dist/esm",
    "declaration": false
  }
}
```

**tsconfig.cjs.json:**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node",
    "outDir": "dist/cjs",
    "declaration": false
  }
}
```

**Base tsconfig.json:**

```json
{
  "compilerOptions": {
    "target": "es2022",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "rootDir": "src",
    "declaration": true,
    "declarationDir": "dist/types",
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src"]
}
```

**Build script in package.json:**

```json
{
  "scripts": {
    "build": "npm run build:esm && npm run build:cjs && npm run build:types",
    "build:esm": "tsc -p tsconfig.esm.json",
    "build:cjs": "tsc -p tsconfig.cjs.json",
    "build:types": "tsc -p tsconfig.json --emitDeclarationOnly"
  }
}
```

An increasingly popular alternative is to use `tsup`, `unbuild`, or `pkgroll` which handle dual output from a single config:

```bash
# tsup handles CJS + ESM + .d.ts in one command
npx tsup src/index.ts --format cjs,esm --dts
```

### `types` Placement

TypeScript 5.0+ supports separate type conditions per export condition. This allows CJS and ESM consumers to get different `.d.ts` files if needed:

```json
{
  "exports": {
    ".": {
      "import": {
        "types": "./dist/types/index.d.mts",
        "default": "./dist/esm/index.js"
      },
      "require": {
        "types": "./dist/types/index.d.cts",
        "default": "./dist/cjs/index.cjs"
      }
    }
  }
}
```

In most libraries, a single `.d.ts` works for both. The nested format is only needed when the CJS and ESM APIs differ (e.g., CJS exports a default, ESM exports named).

---

## `imports` Field (Node Subpath Imports)

The `imports` field in `package.json` defines **private** import aliases within your own package. All keys must start with `#`:

```json
{
  "name": "my-service",
  "type": "module",
  "imports": {
    "#db": "./src/infrastructure/database/client.js",
    "#db/*": "./src/infrastructure/database/*.js",
    "#config": "./src/config/index.js",
    "#utils/*": "./src/shared/utils/*.js",
    "#domain/*": "./src/domain/*.js"
  }
}
```

Now instead of fragile relative paths:

```typescript
// Before: fragile relative paths that break when you move files
import { db } from "../../../infrastructure/database/client.js";
import { validateEmail } from "../../../shared/utils/validation.js";

// After: stable aliases resolved by Node itself
import { db } from "#db";
import { validateEmail } from "#utils/validation.js";
```

The `imports` field supports the same conditional syntax as `exports`:

```json
{
  "imports": {
    "#logger": {
      "development": "./src/logging/dev-logger.js",
      "production": "./src/logging/prod-logger.js",
      "default": "./src/logging/prod-logger.js"
    }
  }
}
```

```bash
# Node picks the condition based on --conditions flag
node --conditions=development src/index.js
```

### Comparison to tsconfig `paths`

| Feature | `imports` (package.json) | `paths` (tsconfig.json) |
|---|---|---|
| Runtime resolution | Yes (Node resolves natively) | **No** (erased at compile time) |
| Needs bundler/loader | No | Yes (ts-node, tsx, webpack, etc.) |
| Prefix | Must start with `#` | Any prefix (e.g., `@/`) |
| Conditional per env | Yes (import/require/dev/prod) | No |
| Scoped to package | Yes | Yes (within tsconfig scope) |
| ESM compatible | Yes | Only if the tool supports it |

The critical difference: `paths` is a **compile-time alias only**. TypeScript resolves it for type-checking but emits the alias verbatim. Node does not understand `@/utils/foo` at runtime. You need an additional tool (bundler, `tsc-alias`, `tsconfig-paths`) to rewrite the paths in the output.

`imports` with the `#` prefix works at runtime with zero additional tooling. TypeScript (with `moduleResolution: "node16"` or `"nodenext"`) reads the `imports` field from `package.json` and resolves types correctly during compilation.

**Recommendation:** For Node.js backend services (Express, Fastify, NestJS), prefer `#imports` over tsconfig `paths`. You get native runtime resolution, no extra build step, and proper ESM support.

```json
// tsconfig.json — no "paths" needed when using #imports
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "rootDir": "src",
    "outDir": "dist"
  }
}
```

---

## Common Resolution Failures

### `Cannot find module`

**When:** TypeScript compilation (red squiggly in IDE).

**Typical causes:**

1. **Missing `.js` extension in ESM mode.**

```typescript
// tsconfig: moduleResolution "node16"

import { foo } from "./utils";       // ERROR: Cannot find module './utils'
import { foo } from "./utils.js";    // OK
```

2. **Package has no types.** The JS package works at runtime but TS can't find type declarations.

```bash
# Fix: install the community types
npm install -D @types/express

# Or: create a declaration file
# src/types/untyped-lib.d.ts
declare module "untyped-lib" {
  export function doStuff(): void;
}
```

3. **`exports` field blocks deep imports.** The package exports `"."` but not `"./internal"`.

```typescript
import { something } from "some-lib/internal";
// Cannot find module 'some-lib/internal'
// The package "some-lib" does not export "./internal"
```

### `ERR_MODULE_NOT_FOUND`

**When:** Node.js runtime in ESM mode.

**Typical causes:**

1. **Missing file extension.** ESM requires the full specifier.

```bash
# Error:
# Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/dist/utils'
#   imported from /app/dist/index.js
# Did you mean to import "../utils.js"?

# The fix: add .js extension to all relative imports in source
```

2. **Importing a directory.** ESM does not resolve `index.js` from a directory path.

```bash
# Error:
# Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/dist/routes'
# (because /app/dist/routes is a directory, not a file)

# Fix: import the index file explicitly
import { router } from "./routes/index.js";
```

3. **File literally does not exist.** Check that `tsc` output landed where you think it did. Verify `outDir` in tsconfig and check the `dist/` directory.

### `ERR_REQUIRE_ESM`

**When:** A CJS file tries to `require()` an ESM-only package.

```bash
# Error:
# Error [ERR_REQUIRE_ESM]: require() of ES Module /app/node_modules/got/dist/source/index.js
#   from /app/src/fetcher.js not supported.
# Instead change the require of index.js in /app/src/fetcher.js to a dynamic import()
#   which is available in all CommonJS modules.
```

This happens because many popular packages (got, node-fetch v3, chalk v5, execa v6+, nanoid v4+) shipped ESM-only breaking changes.

**Fixes (in order of preference):**

1. **Migrate your project to ESM.** Add `"type": "module"` to your `package.json` and update imports.

2. **Use dynamic `import()` in your CJS code:**

```javascript
// CommonJS file
async function fetchData(url) {
  const { default: got } = await import("got");  // dynamic import works in CJS
  return got(url).json();
}
```

3. **Pin the last CJS-compatible version:**

```bash
npm install got@11  # last CJS version before ESM-only v12
```

### Diagnosis Cheat Sheet

When an import fails, walk through this checklist:

```text
1. What module system is the CURRENT file running as?
   -> Check "type" in nearest package.json, or look at file extension (.mjs/.cjs)

2. What module system does the TARGET expose?
   -> Check target's package.json "exports" and "type"

3. Is the specifier correct for the module system?
   -> ESM: full path with extension required
   -> CJS: extension probing and index resolution available

4. Does the target's "exports" field allow this subpath?
   -> If "exports" exists, only listed paths are importable

5. Are TypeScript types available?
   -> Check "types" condition in exports, or @types/* package, or bundled .d.ts

6. Is moduleResolution in tsconfig matching reality?
   -> "node16"/"nodenext" for direct Node execution
   -> "bundler" for Vite/Webpack/esbuild pipelines
```

---

## Java Parallel: Classpath and Module Path

If you are coming from the Java learning path, the parallels are:

| Node/TS concept | Java equivalent |
|---|---|
| `node_modules` lookup (walk up dirs) | Classpath search order |
| `package.json` `exports` | `module-info.java` `exports` directive |
| Subpath exports (encapsulation) | `exports com.mylib.api` (hides internal packages) |
| `"type": "module"` | Adding `module-info.java` to a JAR |
| `.mjs` / `.cjs` overrides | Classpath JAR (automatic module) vs module-path JAR |
| `imports` field (`#` aliases) | `requires` directives in `module-info.java` |
| `ERR_REQUIRE_ESM` | `IllegalAccessError` when accessing unexported package |
| Dual CJS/ESM publish | Multi-release JARs (`META-INF/versions/`) |

Both ecosystems went through the same evolution: a loose resolution system (classpath / `require`) replaced by a stricter module system (JPMS / ESM) that adds encapsulation but breaks backward compatibility during the transition period.

---

## Related

- [tsconfig Demystified](tsconfig-demystified.md) -- `module`, `moduleResolution`, and related compiler options
- [Declaration Files & Type Acquisition](declaration-files.md) -- how `.d.ts` files and `@types/*` fit into resolution

---

## References

- [Node.js Modules: ECMAScript modules](https://nodejs.org/api/esm.html) -- official ESM resolution algorithm
- [Node.js Modules: CommonJS modules](https://nodejs.org/api/modules.html) -- `require()` resolution algorithm
- [Node.js Modules: Packages](https://nodejs.org/api/packages.html) -- `exports`, `imports`, `type` field documentation
- [TypeScript: Module Resolution](https://www.typescriptlang.org/docs/handbook/modules/reference.html) -- `moduleResolution` settings and behavior
- [TypeScript: Modules - Theory](https://www.typescriptlang.org/docs/handbook/modules/theory.html) -- the "why" behind TS module design decisions
- [Are the types wrong?](https://arethetypeswrong.github.io/) -- tool to check if a package's types resolve correctly for CJS and ESM consumers
- [publint](https://publint.dev/) -- lint `package.json` `exports` for correctness
