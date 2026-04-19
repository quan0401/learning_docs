---
title: "Declaration Files & Type Acquisition"
date: 2026-04-19
updated: 2026-04-19
tags: [typescript, declaration-files, d.ts, types, definitelytyped, module-augmentation]
---

# Declaration Files & Type Acquisition

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `typescript` `declaration-files` `d.ts` `types` `definitelytyped` `module-augmentation`

## Table of Contents

- [Summary](#summary)
- [What `.d.ts` Files Are](#what-dts-files-are)
- [`@types/*` Packages and DefinitelyTyped](#types-packages-and-definitelytyped)
- [Writing Your Own `.d.ts` for an Untyped Module](#writing-your-own-dts-for-an-untyped-module)
- [Declaration Merging](#declaration-merging)
- [Module Augmentation](#module-augmentation)
- [`declare global`](#declare-global)
- [Triple-Slash Directives](#triple-slash-directives)
- [Related](#related)
- [References](#references)

---

## Summary

You install a library, import it, and TypeScript screams: *"Could not find a declaration file for module 'some-lib'."* This doc covers everything between that error and a fully typed codebase: how `.d.ts` files work, where community types come from, how to write your own declarations, and how to extend third-party types when they are incomplete.

**Java parallel:** A `.d.ts` file is conceptually similar to a Java interface packaged in a separate JAR. The `.d.ts` describes the contract (method signatures, types) without providing the implementation — just as a Java interface defines `void process(Order order)` without a method body. DefinitelyTyped is roughly analogous to Maven Central hosting interface-only JARs for libraries that did not originally ship with typed APIs.

---

## What `.d.ts` Files Are

A declaration file (`.d.ts`) contains **only type information** — no runtime code. It tells the compiler the shape of values, functions, and classes exported by a JavaScript module.

### The Three-File Relationship

When you compile a TypeScript file with `--declaration`, the compiler emits up to three artifacts:

```text
src/utils.ts          # Source: types + implementation
dist/utils.js         # Runtime: JavaScript only
dist/utils.d.ts       # Types: declarations only
dist/utils.d.ts.map   # Source map: links .d.ts positions back to .ts
```

The `.d.ts.map` file enables "Go to Definition" to jump to the original `.ts` source instead of landing on the declaration file. Libraries that ship `.d.ts.map` files give consumers a dramatically better IDE experience.

### Generating Declarations

```json
// tsconfig.json
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "declarationDir": "./dist/types",
    "emitDeclarationOnly": false
  }
}
```

Given this source:

```typescript
// src/hash.ts
export function sha256(input: string): string {
  // ... implementation
}

export interface HashOptions {
  encoding: "hex" | "base64";
  rounds: number;
}
```

`tsc` produces:

```typescript
// dist/types/hash.d.ts
export declare function sha256(input: string): string;

export interface HashOptions {
  encoding: "hex" | "base64";
  rounds: number;
}
```

Notice that `declare` appears on the function but not on the interface. Interfaces are already purely declarative — they have no runtime representation. The `declare` keyword means "this symbol exists at runtime but I am only describing its type here."

### When Libraries Ship Their Own Types

Modern libraries written in TypeScript typically set `"types"` (or `"typings"`) in their `package.json`:

```json
{
  "name": "my-lib",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts"
}
```

When you `import { something } from "my-lib"`, the compiler reads `dist/index.d.ts` for type information and the bundler/runtime reads `dist/index.js` for execution. Two files, one import.

---

## `@types/*` Packages and DefinitelyTyped

Many popular libraries (Express, Lodash, pg) are written in JavaScript and ship no `.d.ts` files. The community maintains type declarations in the [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) monorepo, published to npm under the `@types` scope.

```bash
npm install express
npm install -D @types/express    # types only, devDependency
```

### How the Compiler Finds `@types`

By default, TypeScript automatically includes every package under `node_modules/@types/` in the compilation. This is controlled by two tsconfig options:

```json
{
  "compilerOptions": {
    // Only include these @types packages (instead of everything):
    "types": ["node", "express"],

    // Where to look for type packages (defaults to node_modules/@types):
    "typeRoots": ["./node_modules/@types", "./custom-types"]
  }
}
```

**`types`** — An allowlist. If specified, *only* these `@types` packages are included globally. Packages not listed here are invisible unless you explicitly `import` from them. Useful when stray `@types` packages pollute the global scope (e.g., `@types/jest` leaking `describe` and `it` into non-test files).

**`typeRoots`** — Directories to scan for type packages. Each subdirectory is treated as a package (the directory name is the package name). Rarely overridden unless you have a custom types folder.

### `typeAcquisition`

This setting applies **only to JavaScript projects** (no `tsconfig.json`) using the VS Code language service or `ts-server`. It controls automatic `@types` downloading:

```json
// jsconfig.json
{
  "typeAcquisition": {
    "enable": true,
    "include": ["express"],
    "exclude": ["jquery"]
  }
}
```

In a TypeScript project with a `tsconfig.json`, this option does nothing. You manage types through `npm install` and the `types`/`typeRoots` options.

### `skipLibCheck`

```json
{
  "compilerOptions": {
    "skipLibCheck": true
  }
}
```

When `true`, the compiler skips type-checking all `.d.ts` files (both your own and `node_modules`). This exists because:

1. **Conflicting type versions.** Two `@types` packages might depend on different versions of `@types/node`, creating contradictory global declarations.
2. **Performance.** Checking thousands of `.d.ts` files in `node_modules` adds build time with little practical benefit.
3. **Loose third-party types.** Some `@types` packages contain errors that block your build even though they do not affect your code.

Most production projects enable `skipLibCheck`. The tradeoff: errors inside `.d.ts` files (including your own) are silently ignored.

---

## Writing Your Own `.d.ts` for an Untyped Module

When no `@types` package exists and the library ships no types, you write your own.

### The Minimal "Make It Stop" Declaration

Create a file that declares the module with an `any` export:

```typescript
// src/types/some-lib.d.ts
declare module "some-lib";
```

This tells TypeScript: "this module exists; everything it exports is `any`." The error goes away, but you get no type safety. It is the equivalent of `@SuppressWarnings("unchecked")` in Java — a conscious opt-out.

### A Practical Middle-Ground Declaration

Type only the parts you actually use:

```typescript
// src/types/csv-parser.d.ts
declare module "csv-parser" {
  import { Transform } from "stream";

  interface CsvParserOptions {
    separator?: string;
    headers?: string[] | boolean;
    skipLines?: number;
  }

  function csvParser(options?: CsvParserOptions): Transform;
  export = csvParser;
}
```

This covers the main factory function and the options your code depends on. You do not need to type every obscure option — type what you use.

### A Comprehensive Declaration

For internal libraries or heavily used dependencies, type the full surface:

```typescript
// src/types/internal-metrics.d.ts
declare module "@company/metrics" {
  export interface Counter {
    inc(labels?: Record<string, string>): void;
    inc(value: number, labels?: Record<string, string>): void;
  }

  export interface Histogram {
    observe(value: number, labels?: Record<string, string>): void;
    startTimer(labels?: Record<string, string>): () => number;
  }

  export interface MetricsRegistry {
    createCounter(name: string, help: string): Counter;
    createHistogram(name: string, help: string, buckets?: number[]): Histogram;
  }

  export function createRegistry(prefix: string): MetricsRegistry;
}
```

### Where to Put Declaration Files

The compiler finds `.d.ts` files through several mechanisms:

1. **Included in tsconfig `include` glob.** If your `include` is `["src/**/*"]` and the file is at `src/types/some-lib.d.ts`, it is automatically picked up.

2. **Referenced via `typeRoots`.** Place files in a directory listed in `typeRoots`, structured as `typeRoots/package-name/index.d.ts`.

3. **Referenced via `paths`.** Map a module name to a file explicitly:

```json
{
  "compilerOptions": {
    "paths": {
      "some-lib": ["./src/types/some-lib.d.ts"]
    }
  }
}
```

The simplest approach: create `src/types/` and make sure it falls within your `include` glob.

---

## Declaration Merging

TypeScript merges multiple declarations of the same name into a single definition. This is fundamental to extending third-party types.

### Interface Merging

Two interface declarations with the same name merge their members:

```typescript
interface User {
  id: string;
  name: string;
}

interface User {
  email: string;
  role: "admin" | "member";
}

// The compiler sees a single User with all four fields:
const user: User = {
  id: "1",
  name: "Quan",
  email: "quan@example.com",
  role: "admin",
};
```

This works across files as long as both declarations are in the same scope. It is the mechanism behind every type extension pattern in TypeScript.

**Java parallel:** Java interfaces cannot be "reopened" — once declared, their contract is fixed. TypeScript's merging is closer to Kotlin extension functions or Java's default methods added in a later interface version, but more powerful because it happens at the type level with no runtime cost.

### Namespace Merging

Namespaces with the same name also merge:

```typescript
namespace Validation {
  export function isEmail(value: string): boolean {
    return /^[^@]+@[^@]+$/.test(value);
  }
}

namespace Validation {
  export function isUUID(value: string): boolean {
    return /^[0-9a-f]{8}-/.test(value);
  }
}

// Both functions are available:
Validation.isEmail("a@b.com");
Validation.isUUID("550e8400-...");
```

### The Express.Request Augmentation Pattern

This is the pattern every Express developer needs. Express ships types via `@types/express`, which re-exports interfaces from `@types/express-serve-static-core`. To add properties to `Request` (e.g., after auth middleware attaches a `user`), you augment that core module:

```typescript
// src/types/express.d.ts
import { User } from "../models/user";

declare module "express-serve-static-core" {
  interface Request {
    user?: User;
    requestId: string;
  }
}
```

Now every `req` in every route handler knows about `user` and `requestId`:

```typescript
// src/routes/profile.ts
import { Router } from "express";

const router = Router();

router.get("/profile", (req, res) => {
  // req.user is typed as User | undefined
  if (!req.user) {
    return res.status(401).json({ error: "Not authenticated" });
  }
  res.json({ name: req.user.name, email: req.user.email });
});
```

**Why `express-serve-static-core` and not `express`?** The `Request` interface you interact with is defined in `@types/express-serve-static-core`. The `@types/express` package re-exports it. If you augment `"express"` directly, you are adding to the *module* exports, not to the `Request` interface. Augmenting the core package is what merges into the actual `Request` type.

---

## Module Augmentation

Module augmentation uses `declare module` **inside a module file** (a file with at least one `import` or `export`) to extend a third-party module's types.

### Syntax

```typescript
// src/types/prisma-extensions.ts
import { PrismaClient } from "@prisma/client";

// This import/export makes the file a module — required for augmentation
declare module "@prisma/client" {
  interface PrismaClient {
    $softDelete<T>(model: string, id: string): Promise<T>;
  }
}
```

### Ambient vs Module Augmentation

This distinction trips people up:

**Ambient declaration** — a `declare module "x"` in a `.d.ts` file (or any file with no imports/exports). This **defines** the module's types from scratch. If the module already has types, this replaces them.

```typescript
// ambient.d.ts — no imports, no exports
// This DEFINES what "lodash" exports (or overwrites existing types):
declare module "lodash" {
  export function chunk<T>(array: T[], size: number): T[][];
}
```

**Module augmentation** — a `declare module "x"` inside a file that is already a module (has `import` or `export`). This **extends** the existing types via declaration merging.

```typescript
// augment.ts — has an import, so it IS a module
import "lodash";

declare module "lodash" {
  // ADDS to existing lodash types, does not replace them:
  interface LoDashStatic {
    customHelper(input: string): string;
  }
}
```

### The Critical Rule

If your augmentation file has no `import` or `export` at the top level, TypeScript treats `declare module` as an ambient declaration that **overwrites** the module's types rather than extending them. The fix is simple — add at least one import:

```typescript
// WRONG: no import/export — this replaces express types entirely
declare module "express-serve-static-core" {
  interface Request {
    user?: User;
  }
}

// CORRECT: the import makes this a module — augmentation merges
import { User } from "../models/user";

declare module "express-serve-static-core" {
  interface Request {
    user?: User;
  }
}
```

If you have nothing to import, an empty export works:

```typescript
export {};

declare module "express-serve-static-core" {
  interface Request {
    requestId: string;
  }
}
```

---

## `declare global`

When you need to add types to the **global scope** from inside a module file, use `declare global`. This is the counterpart to module augmentation — instead of extending a specific module, you extend the global namespace.

### Typing Environment Variables

The most common use case in Node/Express projects. Without this, `process.env.DATABASE_URL` is `string | undefined` and every access needs a guard:

```typescript
// src/types/env.d.ts
export {};

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "production" | "test";
      PORT: string;
      DATABASE_URL: string;
      JWT_SECRET: string;
      REDIS_URL: string;
    }
  }
}
```

Now `process.env.DATABASE_URL` is typed as `string` (not `string | undefined`), and accessing `process.env.TYPO` produces a type error.

**Important caveat:** This is a type-level lie. The compiler trusts your declaration, but if the variable is missing at runtime, you get `undefined`. Pair this with runtime validation at startup:

```typescript
// src/config.ts
import "dotenv/config";

const required = ["DATABASE_URL", "JWT_SECRET", "REDIS_URL"] as const;

for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
}

export const config = {
  databaseUrl: process.env.DATABASE_URL,
  jwtSecret: process.env.JWT_SECRET,
  port: parseInt(process.env.PORT || "3000", 10),
} as const;
```

### Adding Properties to `globalThis`

For libraries that attach themselves to the global object (common in testing or polyfill setups):

```typescript
// src/types/global.d.ts
export {};

declare global {
  var prisma: import("@prisma/client").PrismaClient | undefined;

  function sleep(ms: number): Promise<void>;
}
```

The `var` keyword is required here — `let` and `const` are not allowed in `declare global` because they are block-scoped and cannot describe global properties.

### `declare global` vs Top-Level `declare`

In a non-module file (no `import`/`export`), top-level declarations are already global:

```typescript
// global-ambient.d.ts — no import/export, so everything is global
declare function sleep(ms: number): Promise<void>;
```

In a module file, top-level declarations are module-scoped. You need `declare global` to escape into the global scope:

```typescript
// in-a-module.ts
import { something } from "./somewhere";

// This is module-scoped — only visible if you import it:
declare function localHelper(): void;

// This is global — visible everywhere:
declare global {
  function sleep(ms: number): Promise<void>;
}
```

---

## Triple-Slash Directives

Triple-slash directives are special comments that instruct the compiler to include additional files or type packages. They must appear at the very top of a file, before any code.

### `/// <reference types="..." />`

Includes a type package by name, equivalent to adding it to the `types` array in tsconfig:

```typescript
/// <reference types="node" />
/// <reference types="jest" />

// Now Node.js globals (Buffer, process) and Jest globals (describe, it) are available
```

**When you need this:** Rarely in modern projects. The primary use case is in `.d.ts` files that ship with a library and need to declare a dependency on another type package without using `import` (which would make the file a module).

Example from a library's declaration file:

```typescript
// dist/index.d.ts (shipped with the library)
/// <reference types="node" />

export declare function readConfig(path: string): Promise<Buffer>;
```

In application code, prefer `import` over triple-slash directives. Imports are explicit, follow module resolution, and work with tree-shaking.

### `/// <reference path="..." />`

Includes a specific file by relative path:

```typescript
/// <reference path="./legacy-types.d.ts" />
```

**When you need this:** Almost never in modern projects. This was essential before `tsconfig.json` existed — it was the only way to tell the compiler about other files. Today, `include` globs and `import` statements handle file discovery.

You might encounter it in:
- Very old TypeScript codebases being migrated
- Auto-generated declaration files that reference companion files
- Global script files (non-module) that need to reference other global scripts

### `/// <reference lib="..." />`

Includes a built-in library definition:

```typescript
/// <reference lib="es2022" />
/// <reference lib="dom" />
```

Equivalent to adding entries to the `lib` array in tsconfig. Useful in declaration files that need specific built-in types without assuming the consumer's `lib` configuration.

### When Imports Suffice (Most of the Time)

| Scenario | Use Import | Use Triple-Slash |
|----------|-----------|-------------------|
| Application code needing a type | Yes | No |
| `.d.ts` file declaring dependency on `@types/node` | No | Yes |
| Legacy global script referencing another script | No | Yes (path) |
| Library `.d.ts` needing built-in lib types | No | Yes (lib) |

Rule of thumb: if you are writing `.ts` files in an application, use `import`. If you are authoring `.d.ts` files for a library, triple-slash directives declare type dependencies without turning the file into a module.

---

## Related

- [tsconfig Demystified](tsconfig-demystified.md) — `strict`, `module`, `moduleResolution`, and the settings that control how declarations are resolved
- [Module Resolution Deep Dive](module-resolution.md) — how Node and TypeScript resolve `import` paths, `exports` field, CJS/ESM interop

---

## References

- [TypeScript Handbook — Declaration Files](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html)
- [TypeScript Handbook — Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)
- [TypeScript Handbook — Module Augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation)
- [DefinitelyTyped Repository](https://github.com/DefinitelyTyped/DefinitelyTyped)
- [tsconfig Reference — `types`](https://www.typescriptlang.org/tsconfig#types)
- [tsconfig Reference — `typeRoots`](https://www.typescriptlang.org/tsconfig#typeRoots)
- [tsconfig Reference — `skipLibCheck`](https://www.typescriptlang.org/tsconfig#skipLibCheck)
- [`@types/express-serve-static-core`](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/express-serve-static-core)
