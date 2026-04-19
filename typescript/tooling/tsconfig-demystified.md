---
title: "tsconfig Demystified"
date: 2026-04-19
updated: 2026-04-19
tags: [typescript, tsconfig, compiler, module-resolution, strict, project-references, tooling]
---

# tsconfig Demystified

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `typescript` `tsconfig` `compiler` `module-resolution` `strict` `project-references` `tooling`

## Table of Contents

- [Summary](#summary)
- [The `strict` Family](#the-strict-family)
  - [`strictNullChecks`](#strictnullchecks)
  - [`noImplicitAny`](#noimplicitany)
  - [`strictFunctionTypes`](#strictfunctiontypes)
  - [`strictBindCallApply`](#strictbindcallapply)
  - [`strictPropertyInitialization`](#strictpropertyinitialization)
  - [`noImplicitThis`](#noimplicitthis)
  - [`alwaysStrict`](#alwaysstrict)
  - [`useUnknownInCatchVariables`](#useunknownincatchvariables)
- [`module` and `moduleResolution`](#module-and-moduleresolution)
  - [The Critical Distinction](#the-critical-distinction)
  - [`moduleResolution` Options](#moduleresolution-options)
  - [Why `moduleResolution: "node"` Is Legacy](#why-moduleresolution-node-is-legacy)
  - [Decision Matrix](#decision-matrix)
- [`target`](#target)
  - [What `target` Actually Controls](#what-target-actually-controls)
  - [What `target` Does NOT Do](#what-target-does-not-do)
- [Project References](#project-references)
  - [When You Need Them](#when-you-need-them)
  - [Practical Two-Project Setup](#practical-two-project-setup)
- [Path Aliases](#path-aliases)
  - [The tsconfig Side](#the-tsconfig-side)
  - [The Runtime Problem](#the-runtime-problem)
  - [Node Subpath Imports (Recommended)](#node-subpath-imports-recommended)
- [Other Practical Flags](#other-practical-flags)
  - [`esModuleInterop`](#esmoduleinterop)
  - [`resolveJsonModule`](#resolvejsonmodule)
  - [`skipLibCheck`](#skiplibcheck)
  - [`declaration` and `declarationMap`](#declaration-and-declarationmap)
  - [`sourceMap`](#sourcemap)
  - [`outDir` and `rootDir`](#outdir-and-rootdir)
  - [`incremental`](#incremental)
- [Starter Configs](#starter-configs)
- [Related](#related)
- [References](#references)

---

## Summary

`tsconfig.json` is where your TypeScript project's behavior is defined: what syntax the compiler emits, how it resolves imports, how strict the type checker is, and where output goes. Most backend TS developers copy-paste a config from a blog post and never touch it until something breaks. Then one flag change cascades into dozens of errors and the reaction is to revert and move on.

This doc explains every flag you are likely to encounter in a Node.js backend project. The goal is that after reading it, you can look at any `tsconfig.json` and know *why* each line is there, what breaks if you remove it, and what you gain by adding it.

**Java parallel:** Think of `tsconfig.json` as a combination of `javac` compiler flags, `maven-compiler-plugin` configuration, and Gradle's `sourceCompatibility`/`targetCompatibility` --- except it also controls module resolution, which Java handles via the classpath/module-path.

---

## The `strict` Family

The `"strict": true` flag is a meta-flag that enables a collection of individual checks. As of TypeScript 5.x, it enables all of the following. Each can be toggled independently.

```jsonc
// These two configs are equivalent:
{ "compilerOptions": { "strict": true } }

{
  "compilerOptions": {
    "strictNullChecks": true,
    "noImplicitAny": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "useUnknownInCatchVariables": true
  }
}
```

> **Recommendation:** Always use `"strict": true`. If you need to disable one flag during migration, set `"strict": true` and then override the specific flag: `"strictPropertyInitialization": false`. This way you get every *future* strict flag that TypeScript adds automatically.

### `strictNullChecks`

**What it does:** Makes `null` and `undefined` their own types instead of assignable to everything.

**Java parallel:** This is like going from a world where every reference can be `null` (Java without `@NonNull` annotations) to a world where nullability is tracked in the type system (Kotlin's `String` vs `String?`).

```typescript
// strictNullChecks: false — compiles fine, crashes at runtime
function getLength(s: string) {
  return s.length;
}
getLength(null); // No error. Runtime: "Cannot read properties of null"

// strictNullChecks: true — compile error
function getLength(s: string) {
  return s.length;
}
getLength(null); // Error: Argument of type 'null' is not assignable to parameter of type 'string'
```

**What breaks when you enable it:** Every place where you pass `null`/`undefined` to a function expecting a concrete type. Every place where you access a property without a null guard. Typically the largest wave of errors when turning on `strict`.

**Fix pattern:**

```typescript
function getLength(s: string | null): number {
  if (s === null) return 0; // narrow first
  return s.length;          // safe
}
```

### `noImplicitAny`

**What it does:** Errors when TypeScript cannot infer a type and would silently assign `any`.

```typescript
// noImplicitAny: false — silently becomes 'any'
function add(a, b) {  // a: any, b: any — no error
  return a + b;
}

// noImplicitAny: true — compile error
function add(a, b) {  // Error: Parameter 'a' implicitly has an 'any' type
  return a + b;
}
```

**What breaks when you enable it:** Every untyped callback parameter, every `req`/`res` in Express handlers without explicit types, every destructured parameter without annotation.

**Fix pattern:**

```typescript
import { Request, Response } from "express";

function add(a: number, b: number): number {
  return a + b;
}

app.get("/users", (req: Request, res: Response) => {
  // ...
});
```

### `strictFunctionTypes`

**What it does:** Enforces *contravariant* parameter checking for function types. Without it, TypeScript checks function parameters *bivariantly*, which is unsound.

```typescript
type Handler = (event: MouseEvent) => void;

// strictFunctionTypes: false — compiles (UNSOUND)
const handler: Handler = (event: Event) => {
  console.log(event.clientX); // Runtime error: clientX doesn't exist on Event
};

// strictFunctionTypes: true — compile error
// Type '(event: Event) => void' is not assignable to type 'Handler'.
//   Types of parameters 'event' and 'event' are incompatible.
```

**Java parallel:** Java generics already enforce this via bounded wildcards. `Consumer<Event>` is not assignable to `Consumer<MouseEvent>` because `Consumer` is contravariant in its type parameter.

**What breaks when you enable it:** Libraries that use broad event handlers, middleware chains that widen parameter types.

### `strictBindCallApply`

**What it does:** Type-checks `bind`, `call`, and `apply` using the actual function signature instead of allowing `any`.

```typescript
function greet(name: string, age: number) {
  return `${name} is ${age}`;
}

// strictBindCallApply: false
greet.call(null, "Quan", "not a number"); // No error — args are 'any'

// strictBindCallApply: true
greet.call(null, "Quan", "not a number");
// Error: Argument of type 'string' is not assignable to parameter of type 'number'
```

### `strictPropertyInitialization`

**What it does:** Requires class properties to be initialized in the constructor or declared with a definite assignment assertion (`!`).

```typescript
// strictPropertyInitialization: true
class UserService {
  private db: Database; // Error: Property 'db' has no initializer and is not definitely assigned

  constructor() {
    // forgot to assign this.db
  }
}
```

**Java parallel:** This is exactly what Java's `final` field enforcement does — the compiler ensures every `final` field is assigned by the end of the constructor.

**Fix pattern — three options:**

```typescript
// 1. Initialize in constructor
class UserService {
  private db: Database;
  constructor(db: Database) { this.db = db; }
}

// 2. Initialize at declaration
class UserService {
  private db: Database = new Database();
}

// 3. Definite assignment assertion (use sparingly — e.g., DI frameworks)
class UserService {
  private db!: Database; // "trust me, it will be assigned"
}
```

### `noImplicitThis`

**What it does:** Errors when `this` has an implicit `any` type.

```typescript
// noImplicitThis: true
class Timer {
  seconds = 0;
  start() {
    setInterval(function () {
      this.seconds++; // Error: 'this' implicitly has type 'any'
    }, 1000);
  }
}
```

**Fix:** Use arrow functions (which capture `this` lexically) or add an explicit `this` parameter.

```typescript
start() {
  setInterval(() => {
    this.seconds++; // OK — arrow captures outer 'this'
  }, 1000);
}
```

### `alwaysStrict`

**What it does:** Emits `"use strict";` at the top of every output file and parses source in strict mode.

This is almost always a no-op for modern projects because ES modules are strict by default. It matters only if you emit CommonJS files that are not ES modules.

### `useUnknownInCatchVariables`

**What it does:** Types `catch` clause variables as `unknown` instead of `any`.

```typescript
// useUnknownInCatchVariables: false
try { /* ... */ } catch (err) {
  console.log(err.message); // err is 'any' — no error, but unsafe
}

// useUnknownInCatchVariables: true
try { /* ... */ } catch (err) {
  console.log(err.message); // Error: 'err' is of type 'unknown'
}
```

**Fix pattern:**

```typescript
try {
  await fetch("/api");
} catch (err) {
  if (err instanceof Error) {
    console.log(err.message); // narrowed to Error
  } else {
    console.log("Unknown error:", err);
  }
}
```

**Java parallel:** Java's `catch (Exception e)` always gives you a typed exception. TypeScript has no such guarantee because anything can be thrown — even a string or number. `unknown` forces you to narrow, which is the correct behavior.

---

## `module` and `moduleResolution`

These two flags are the most confusing pair in `tsconfig.json`. They control different things but interact tightly.

### The Critical Distinction

| Flag | Controls | Analogy |
|------|----------|---------|
| `module` | **What module syntax TypeScript emits** — `require()` or `import/export` | Like choosing whether `javac` outputs classfiles for Java 8 or Java 21 module system |
| `moduleResolution` | **How TypeScript finds the file** behind an import specifier | Like configuring the classpath vs module-path resolution strategy |

```jsonc
{
  "compilerOptions": {
    "module": "node16",             // Emit: ESM or CJS depending on package.json "type"
    "moduleResolution": "node16"    // Resolve: follow Node 16+ rules (exports field, etc.)
  }
}
```

**What breaks if you mix them wrong:**

```jsonc
{
  // WRONG: bundler resolution with Node-style emit
  "module": "commonjs",
  "moduleResolution": "bundler"  // bundler resolution allows bare specifiers
                                  // that Node's CJS loader doesn't understand
}
```

This compiles but fails at runtime because TypeScript resolves imports using bundler rules (which skip file extensions) while Node's `require()` needs exact file paths.

### `moduleResolution` Options

#### `"node16"` / `"nodenext"`

Use when: **You run code directly on Node.js** (Express, Fastify, CLI tools, Prisma scripts).

- Follows Node's actual resolution algorithm
- Requires `.js` extensions in relative imports (even for `.ts` source files)
- Respects `package.json` `"exports"` field
- Distinguishes ESM vs CJS based on `package.json` `"type"` field

```typescript
// With moduleResolution: "node16", you must write:
import { helper } from "./utils.js";  // Yes, .js — even though the source is .ts

// NOT this:
import { helper } from "./utils";     // Error: relative import without extension
```

#### `"bundler"`

Use when: **A bundler (Vite, esbuild, webpack, tsup) processes your code before it runs.**

- Allows extensionless imports
- Respects `package.json` `"exports"`
- Does NOT require `.js` extensions
- Assumes a bundler will handle resolution at build time

```typescript
// With moduleResolution: "bundler":
import { helper } from "./utils";  // OK — bundler resolves this
```

#### `"esnext"` (module, not resolution)

`"module": "esnext"` emits the latest ES module syntax. Pair it with `"moduleResolution": "bundler"` for bundled projects.

### Why `moduleResolution: "node"` Is Legacy

The `"node"` (without the `16`) strategy is the old Node.js CJS-era resolution:

- Does NOT support `package.json` `"exports"` field
- Does NOT distinguish ESM from CJS
- Looks for `index.js` implicitly
- Cannot correctly resolve modern packages that only export via `"exports"`

Many packages (including recent versions of `uuid`, `chalk`, and `jose`) only define an `"exports"` field. With `"moduleResolution": "node"`, TypeScript cannot find their types, and you get:

```
error TS2307: Cannot find module 'jose' or its corresponding type declarations.
```

The fix is to switch to `"node16"` or `"nodenext"`.

### Decision Matrix

| Project type | `module` | `moduleResolution` |
|---|---|---|
| Node.js backend (ESM) | `"node16"` or `"nodenext"` | `"node16"` or `"nodenext"` |
| Node.js backend (CJS) | `"commonjs"` | `"node16"` or `"nodenext"` |
| Vite / esbuild / tsup | `"esnext"` | `"bundler"` |
| Library (dual CJS/ESM) | `"node16"` | `"node16"` |

---

## `target`

### What `target` Actually Controls

`target` tells TypeScript which JavaScript version to *emit*. It controls **syntax downleveling** — converting newer syntax to older equivalents.

```jsonc
{ "compilerOptions": { "target": "es2020" } }
```

**Example: optional chaining (`?.`)**

Optional chaining was introduced in ES2020.

```typescript
// Source
const city = user?.address?.city;
```

```javascript
// target: "es2020" — kept as-is
const city = user?.address?.city;

// target: "es2019" — compiled away
const city = (_b = (_a = user) === null || _a === void 0
  ? void 0 : _a.address) === null || _b === void 0
  ? void 0 : _b.city;
```

**Example: `async`/`await`**

```javascript
// target: "es2017"+ — kept as-is (async/await is ES2017)
async function fetchUser() { return await db.query(); }

// target: "es2015" — downleveled to generator + __awaiter helper
function fetchUser() {
  return __awaiter(this, void 0, void 0, function* () {
    return yield db.query();
  });
}
```

### What `target` Does NOT Do

`target` does **NOT** polyfill missing runtime APIs. If you set `target: "es2022"` but run on Node 14:

```typescript
const items = [1, 2, 3];
items.at(-1); // TypeScript thinks this is fine (Array.prototype.at is ES2022)
              // Node 14 runtime: TypeError: items.at is not a function
```

TypeScript emits the `.at()` call as-is because it is valid ES2022 *syntax*. But Node 14 does not have the `Array.prototype.at` method. You need a polyfill or a newer Node version.

**Java parallel:** This is like `javac --release 11` — it controls what bytecode features are used, but it does not add missing runtime classes. If you use a Java 17 API but compile with `--release 11`, `javac` errors. TypeScript is *less safe* here — it does not error, it just trusts `target` to match your runtime.

**Practical advice for Node.js backends:**

| Node version | Safe `target` |
|---|---|
| Node 18 | `"es2022"` |
| Node 20 | `"es2023"` |
| Node 22 | `"es2024"` |

Match `target` to your minimum Node version. Check [node.green](https://node.green) for exact feature support.

---

## Project References

### When You Need Them

Project references let you split a large TypeScript project into smaller, independently compiled pieces. The compiler builds them in dependency order and caches results.

**You need project references when:**
- Monorepo with shared libraries (e.g., `packages/shared`, `packages/api`, `packages/worker`)
- A `common` package used by multiple services
- Build times are slow and you want incremental compilation across packages

**Java parallel:** This is conceptually similar to a multi-module Maven/Gradle project where each module has its own `build.gradle` and modules declare dependencies on each other.

### Practical Two-Project Setup

```
monorepo/
├── packages/
│   ├── shared/
│   │   ├── src/
│   │   │   └── types.ts
│   │   └── tsconfig.json
│   └── api/
│       ├── src/
│       │   └── server.ts
│       └── tsconfig.json
└── tsconfig.json          ← root "solution" config
```

**`packages/shared/tsconfig.json`:**

```jsonc
{
  "compilerOptions": {
    "composite": true,          // REQUIRED for referenced projects
    "declaration": true,        // REQUIRED — other projects need .d.ts files
    "declarationMap": true,     // Enables "Go to Definition" to reach source
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src"]
}
```

**`packages/api/tsconfig.json`:**

```jsonc
{
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../shared" }   // Declares dependency on shared
  ],
  "include": ["src"]
}
```

**Root `tsconfig.json`:**

```jsonc
{
  "files": [],                 // Root config does not compile anything itself
  "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/api" }
  ]
}
```

**Build command:**

```bash
tsc --build              # Builds in dependency order: shared → api
tsc --build --watch      # Incremental watch mode across all projects
```

**Key flags:**

| Flag | Purpose |
|---|---|
| `composite` | Marks a project as a buildable unit. Requires `declaration: true` |
| `references` | Declares which other projects this one depends on |
| `tsBuildInfo` | Generated automatically — stores incremental build state |

**What breaks if you skip `composite`:**

```
error TS6306: Referenced project 'packages/shared' must have setting "composite": true.
```

---

## Path Aliases

### The tsconfig Side

Path aliases let you replace deep relative imports with short prefixes:

```typescript
// Without aliases
import { User } from "../../../../shared/types/user";

// With aliases
import { User } from "@shared/types/user";
```

**Configuration:**

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",              // Required anchor for paths resolution
    "paths": {
      "@shared/*": ["packages/shared/src/*"],
      "@api/*": ["packages/api/src/*"]
    }
  }
}
```

### The Runtime Problem

Here is the critical gotcha: **`paths` only affects TypeScript's type checking and editor experience. It does NOT rewrite imports in the emitted JavaScript.**

Your compiled JS still contains:

```javascript
// dist/api/server.js
const user_1 = require("@shared/types/user"); // Node has no idea what "@shared" means
```

Node throws: `Error: Cannot find module '@shared/types/user'`

**You must also configure a runtime resolver.** Pick one:

| Approach | When to use |
|---|---|
| `tsconfig-paths` | Quick fix for `ts-node` or development |
| Bundler aliases (esbuild, webpack) | Production bundled builds |
| Node subpath imports | Zero-dependency, Node-native solution |

### Node Subpath Imports (Recommended)

Since Node 12.19+, you can define import aliases directly in `package.json` using the `"imports"` field. This works at runtime without any extra tooling.

**`package.json`:**

```jsonc
{
  "imports": {
    "#shared/*": "./packages/shared/src/*",
    "#api/*": "./packages/api/src/*"
  }
}
```

**Usage:**

```typescript
import { User } from "#shared/types/user"; // Works in both TS and Node runtime
```

> Note: Node requires subpath imports to start with `#`. This is by spec — it prevents collisions with npm package names.

**tsconfig side** (still needed for type checking):

```jsonc
{
  "compilerOptions": {
    "paths": {
      "#shared/*": ["packages/shared/src/*"],
      "#api/*": ["packages/api/src/*"]
    }
  }
}
```

Both `package.json` `"imports"` and `tsconfig.json` `"paths"` must agree. If they diverge, TypeScript will resolve to one file but Node will load a different one (or crash).

---

## Other Practical Flags

### `esModuleInterop`

**What it does:** Adds helper functions to handle the mismatch between CommonJS `module.exports` and ES module `import` syntax.

Without it:

```typescript
// express exports via module.exports = createApplication
import express from "express";
// Error: Module has no default export

// You'd have to write:
import * as express from "express";
```

With `esModuleInterop: true`:

```typescript
import express from "express"; // Works — helper creates synthetic default
```

**Recommendation:** Always enable. Nearly every Node.js project needs this.

### `resolveJsonModule`

**What it does:** Allows importing `.json` files with type safety.

```typescript
// resolveJsonModule: true
import config from "./config.json";
// config is typed as { port: number; host: string; ... } based on the actual JSON
```

Without it: `Cannot find module './config.json'`.

### `skipLibCheck`

**What it does:** Skips type-checking `.d.ts` files (declaration files from `node_modules`).

**Why you want it:** Third-party declaration files sometimes conflict with each other or with your TypeScript version. `skipLibCheck` avoids cascading errors from packages you do not control.

**The trade-off:** You lose type-checking of library declarations. In practice, this is fine — library authors test their own types. The alternative is spending hours debugging type errors in code you did not write.

**Recommendation:** Enable for application code. Disable for library authoring where you need to verify your own `.d.ts` output.

### `declaration` and `declarationMap`

**`declaration: true`** — generates `.d.ts` files alongside your JavaScript output. Required when you publish a library or use project references.

**`declarationMap: true`** — generates `.d.ts.map` files that let "Go to Definition" in your editor jump to the original `.ts` source instead of the `.d.ts` file.

```jsonc
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true  // Enables source-level navigation in editors
  }
}
```

**When to use:** Library packages, shared monorepo packages, any project consumed as a dependency.

### `sourceMap`

**What it does:** Generates `.js.map` files that map compiled JavaScript back to your TypeScript source. This is what makes debugger breakpoints, stack traces, and error reporting point to your `.ts` files instead of compiled `.js`.

```jsonc
{ "compilerOptions": { "sourceMap": true } }
```

**When to disable:** Production Docker images where you want smaller builds and do not need source-level debugging. Even then, many teams keep source maps for error tracking (Sentry, Datadog).

### `outDir` and `rootDir`

**`outDir`** — where compiled files go:

```jsonc
{ "compilerOptions": { "outDir": "./dist" } }
// src/server.ts → dist/server.js
```

**`rootDir`** — the root of your source tree. Controls the directory structure inside `outDir`:

```jsonc
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
// src/server.ts → dist/server.js  (not dist/src/server.js)
```

**What breaks if you omit `rootDir`:** TypeScript infers it from the common ancestor of all input files. If you accidentally include a file outside `src/`, the directory structure in `dist/` changes unpredictably:

```
// Without rootDir, if tsconfig accidentally includes ./scripts/seed.ts:
dist/
├── src/
│   └── server.js      ← suddenly nested under src/
└── scripts/
    └── seed.js
```

**Recommendation:** Always set `rootDir` explicitly.

### `incremental`

**What it does:** Caches compilation results in a `.tsBuildInfo` file so subsequent builds only recompile changed files.

```jsonc
{ "compilerOptions": { "incremental": true } }
```

On a mid-size project (500+ files), this can cut rebuild times from 10+ seconds to under 1 second. The `.tsBuildInfo` file should be in `.gitignore`.

---

## Starter Configs

**Node.js backend (Express/Fastify, ESM, Node 20+):**

```jsonc
{
  "compilerOptions": {
    // Type checking
    "strict": true,
    "noUncheckedIndexedAccess": true,     // arr[0] is T | undefined, not T
    "noFallthroughCasesInSwitch": true,

    // Module system
    "module": "node16",
    "moduleResolution": "node16",
    "target": "es2023",

    // Emit
    "outDir": "./dist",
    "rootDir": "./src",
    "sourceMap": true,
    "declaration": true,

    // Interop
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,

    // Performance
    "incremental": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**Library package (consumed by other TS projects):**

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "module": "node16",
    "moduleResolution": "node16",
    "target": "es2022",

    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": true,         // If used with project references

    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

---

## Related

- [Module Resolution Deep Dive](module-resolution.md) — Node's import resolution algorithm, `.js` extensions in ESM, dual CJS/ESM publishing
- [Declaration Files & Type Acquisition](declaration-files.md) — `.d.ts` mechanics, `@types/*`, module augmentation
- [Build Tools and JVM](../../java/java-fundamentals/build-tools-and-jvm.md) — Java's equivalent of compiler and build configuration

---

## References

- [TypeScript: TSConfig Reference](https://www.typescriptlang.org/tsconfig) — official docs for every compiler option
- [TypeScript: Module Resolution](https://www.typescriptlang.org/docs/handbook/modules/theory.html) — deep theory on how TS resolves imports
- [TypeScript: Project References](https://www.typescriptlang.org/docs/handbook/project-references.html) — official guide to `composite` and `references`
- [Node.js: Subpath Imports](https://nodejs.org/api/packages.html#subpath-imports) — the `"imports"` field spec
- [node.green](https://node.green) — Node.js ECMAScript compatibility table
- [TypeScript 5.0 Release Notes](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/) — `bundler` moduleResolution introduced
- [Are the types wrong?](https://arethetypeswrong.github.io/) — tool to check if a package's types resolve correctly under different `moduleResolution` settings
