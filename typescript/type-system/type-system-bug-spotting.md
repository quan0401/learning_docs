---
title: "TypeScript Type System — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, typescript, type-system, compiler, generics, narrowing]
---

# TypeScript Type System — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `typescript` `type-system` `compiler` `generics` `narrowing`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary

Active-recall practice for the TypeScript type system. Twenty-three broken snippets covering structural typing, control-flow narrowing, conditional and mapped types, `infer`, `satisfies` vs `as`, `any` vs `unknown`, type predicates, declaration merging, branded types, function parameter bivariance, and distributive conditional types. Run through Easy in one sitting, Subtle in a second pass, and treat Senior trap as a code-review interview drill. Many bugs only surface under specific strict flags (`strict`, `strictNullChecks`, `strictFunctionTypes`, `noUncheckedIndexedAccess`); each bug calls out the flag when it matters. Every citation is a real TypeScript handbook page, release note, GitHub issue, or recognized-author article.

## How to use this doc
- Try to spot the bug before opening the `<details>` hint.
- Hints are one line; full root cause + corrected code lives in §4 keyed by bug number.
- Assume `"strict": true` unless a bug specifies otherwise.
- Don't skip Easy unless you've already nailed those traps in code review.

## 1. Easy (warm-up traps)

### Bug 1 — `any` swallows the error
```typescript
function getUserName(user: any): string {
  return user.naem; // typo, no error
}
```
<details><summary>Hint</summary>
What does <code>any</code> opt out of?
</details>

### Bug 2 — `as` cast launders a wrong shape
```typescript
type User = { id: string; email: string };
const raw = JSON.parse('{"id":"u1"}');
const u = raw as User;
console.log(u.email.toLowerCase()); // runtime: Cannot read properties of undefined
```
<details><summary>Hint</summary>
A type assertion is a promise to the compiler, not a check.
</details>

### Bug 3 — excess property check only fires on fresh literals
```typescript
type Options = { timeout: number };
function configure(o: Options) {}

configure({ timeout: 30, retires: 3 }); // error: 'retires' not in Options

const opts = { timeout: 30, retires: 3 };
configure(opts); // no error — why?
```
<details><summary>Hint</summary>
Excess property checks only target object literals at the call site.
</details>

### Bug 4 — `Array.find` result used without a null check
```typescript
const users: User[] = await loadUsers();
const admin = users.find(u => u.role === "admin");
console.log(admin.email); // strictNullChecks should bark — but does the code below silence it?
```
<details><summary>Hint</summary>
What is the return type of <code>Array.prototype.find</code>?
</details>

### Bug 5 — `Promise<T>` assigned where `T` was expected
```typescript
async function fetchUser(): Promise<User> { /* ... */ return {} as User; }

function render(user: User) { /* ... */ }

const user = fetchUser(); // forgot await
render(user); // expected User, got Promise<User>
```
<details><summary>Hint</summary>
Forgetting <code>await</code> hands the renderer the wrong shape entirely.
</details>

### Bug 6 — `void` return type accepts any return
```typescript
type Logger = (msg: string) => void;

const items = ["a", "b"];
const log: Logger = (msg) => items.push(msg); // returns number — accepted!
```
<details><summary>Hint</summary>
A function-typed parameter declared <code>void</code> is not the same as a return-statement <code>void</code>.
</details>

### Bug 7 — index signature widens to `any` without `noUncheckedIndexedAccess`
```typescript
const config: Record<string, string> = { host: "localhost" };
const port = config["port"]; // typed string — but it's actually undefined at runtime
console.log(port.toUpperCase()); // boom
```
<details><summary>Hint</summary>
Without one tsconfig flag, every index access claims the value is present.
</details>

## 2. Subtle (review-passers)

### Bug 8 — narrowing lost across a function boundary
```typescript
type State = { kind: "loading" } | { kind: "ready"; data: string };

function isReady(s: State): boolean {
  return s.kind === "ready";
}

function render(s: State) {
  if (isReady(s)) {
    console.log(s.data); // error: Property 'data' does not exist on 'State'
  }
}
```
<details><summary>Hint</summary>
A plain <code>boolean</code> return tells the control-flow analyzer nothing.
</details>

### Bug 9 — lying type predicate
```typescript
type Email = string & { __brand: "Email" };

function isEmail(s: string): s is Email {
  return true; // forgot the actual check
}

function send(to: Email) { /* ... */ }
const input: string = "not-an-email";
if (isEmail(input)) send(input); // compiles, ships bad data
```
<details><summary>Hint</summary>
The compiler trusts the predicate's body completely.
</details>

### Bug 10 — discriminated union without exhaustiveness check
```typescript
type Event = { kind: "click" } | { kind: "scroll" } | { kind: "key" };

function describe(e: Event): string {
  switch (e.kind) {
    case "click": return "click";
    case "scroll": return "scroll";
  } // adding a 4th variant later compiles silently
}
```
<details><summary>Hint</summary>
What return type does the compiler infer when one branch is missing?
</details>

### Bug 11 — method-shorthand bivariance vs property arrow
```typescript
// strictFunctionTypes
interface Handler {
  on(evt: MouseEvent): void;          // method shorthand: bivariant
}
interface SafeHandler {
  on: (evt: MouseEvent) => void;      // property: contravariant under strictFunctionTypes
}

const h1: Handler = { on(e: Event) {} };       // accepted — bivariant!
const h2: SafeHandler = { on: (e: Event) => {} }; // error
```
<details><summary>Hint</summary>
Method shorthand opts out of <code>strictFunctionTypes</code>.
</details>

### Bug 12 — distributive conditional unintentionally distributing
```typescript
type Boxed<T> = T extends any ? { value: T } : never;
type X = Boxed<string | number>; // { value: string } | { value: number }, not { value: string | number }
```
<details><summary>Hint</summary>
A naked type parameter on the left of <code>extends</code> distributes over unions.
</details>

### Bug 13 — `keyof` over indexed access loses literal keys
```typescript
type User = { id: string; name: string; age: number };
type StringKeys = { [K in keyof User]: User[K] extends string ? K : never }[keyof User];
// works, but:

const keys = Object.keys({} as User); // typed string[], not ("id"|"name"|"age")[]
keys.forEach(k => console.log(k.toUpperCase()));
```
<details><summary>Hint</summary>
<code>Object.keys</code> is typed deliberately loose — the design rationale is in a TS issue.
</details>

### Bug 14 — mapped type `as` clause filters more than expected
```typescript
type PublicShape<T> = {
  [K in keyof T as K extends `_${string}` ? never : K]: T[K];
};
type X = PublicShape<{ id: string; _internal: number; tag_label: string }>;
// tag_label kept (good) — but what about a key that is a number?
type Y = PublicShape<{ 0: "a"; _x: 1 }>; // numeric key passes through
```
<details><summary>Hint</summary>
<code>K extends `_${string}`</code> only matches strings — numeric and symbol keys slip past.
</details>

### Bug 15 — `Readonly<T>` is shallow
```typescript
type Config = Readonly<{ db: { host: string } }>;
const c: Config = { db: { host: "localhost" } };
c.db = { host: "x" }; // error
c.db.host = "x";      // allowed — Readonly is not deep
```
<details><summary>Hint</summary>
Built-in <code>Readonly</code> only freezes the top level.
</details>

### Bug 16 — `satisfies` doesn't narrow array literals
```typescript
const routes = ["GET", "POST"] satisfies string[];
type R = typeof routes; // string[], not readonly ["GET", "POST"]

const routes2 = ["GET", "POST"] as const;
type R2 = typeof routes2; // readonly ["GET", "POST"]
```
<details><summary>Hint</summary>
<code>satisfies</code> validates against a type but doesn't request literal preservation.
</details>

### Bug 17 — generic constraint over `keyof T` widens to `string | number | symbol`
```typescript
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  // ...
  return {} as Pick<T, K>;
}

type User = { id: string; age: number };
const r = pick({} as User, ["id"]); // K is "id" — fine

declare function badPick<T>(obj: T, keys: (keyof T)[]): Partial<T>; // K not captured
const r2 = badPick({} as User, ["id", "ag" as keyof User]); // typo escapes
```
<details><summary>Hint</summary>
Without a captured <code>K extends keyof T</code>, individual literals collapse to the union.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — branded type forgotten on persistence boundary
```typescript
type UserId = string & { readonly __brand: "UserId" };

function load(id: UserId): Promise<User> { /* ... */ return {} as any; }

// In a controller:
app.get("/users/:id", (req, res) => {
  load(req.params.id as UserId); // cast launders an unvalidated string
});
```
<details><summary>Hint</summary>
Branded types only protect inside the type system — the boundary needs a parser.
</details>

### Bug 19 — declaration merging clobbers a library type
```typescript
// types/express.d.ts
import "express";
declare module "express" {
  interface Request {
    user: { id: string }; // not optional — assumed set by middleware
  }
}

// Anywhere now:
app.get("/me", (req, res) => res.json(req.user.id)); // no error, but middleware may not have run
```
<details><summary>Hint</summary>
Augmenting a global as non-optional makes every consumer skip the null check.
</details>

### Bug 20 — recursive conditional type hits depth limit
```typescript
// Deep flatten of nested arrays
type Flatten<T> = T extends readonly (infer U)[] ? Flatten<U> : T;

type Bad = Flatten<number[][][][][][][][][][][][][][][][][][][][][][][][][]>;
// "Type instantiation is excessively deep and possibly infinite."
```
<details><summary>Hint</summary>
Pre-4.7 conditional recursion was capped at ~50; even tail-recursive forms have a hard limit.
</details>

### Bug 21 — `enum` reverse mapping leaks runtime keys
```typescript
enum Role { Admin = 1, User = 2 }
const allRoles = Object.values(Role); // [ "Admin", "User", 1, 2 ] — not what you wanted
```
<details><summary>Hint</summary>
Numeric enums emit a bidirectional object at runtime.
</details>

### Bug 22 — class `private` is compile-time only
```typescript
class Account {
  private balance = 0;
  add(n: number) { this.balance += n; }
}

const a = new Account();
// (a as any).balance = 1_000_000;        // bypass at runtime
// JSON.stringify(a) // exposes "balance" anyway
```
<details><summary>Hint</summary>
TypeScript <code>private</code> erases at compile time; ECMAScript <code>#name</code> does not.
</details>

### Bug 23 — type-only import accidentally elided at runtime
```typescript
import { Logger } from "./logger"; // Logger is both a class AND used only as a type sometimes
import type { Config } from "./config";

// With "isolatedModules" + "verbatimModuleSyntax", a misclassified import
// can either stay (importing a runtime side effect you didn't want)
// or be elided (and `new Logger()` at runtime throws ReferenceError).
const log = new Logger(); // works only if the import was kept
```
<details><summary>Hint</summary>
<code>verbatimModuleSyntax</code> means TypeScript no longer guesses — <code>import type</code> is erased, plain <code>import</code> is kept.
</details>

## 4. Solutions

### Bug 1 — `any` swallows the error
**Root cause:** `any` opts every property access out of type checking. The typo `naem` is silently accepted; runtime returns `undefined`. The TypeScript Handbook explicitly warns: "When you don't specify a type, and TypeScript can't infer it from context, the compiler will typically default to `any`."
**Fix:** prefer `unknown` and narrow before use, or model the parameter:
```typescript
function getUserName(user: { name: string }): string {
  return user.name;
}
// or, for truly dynamic input:
function getUserNameUnknown(user: unknown): string {
  if (typeof user === "object" && user && "name" in user && typeof user.name === "string") {
    return user.name;
  }
  throw new Error("invalid user");
}
```
**Reference:** [TS Handbook — The any type](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any) and [Effective TypeScript, Item 5: "Limit Use of the `any` Type"](https://effectivetypescript.com/).

### Bug 2 — `as` cast launders a wrong shape
**Root cause:** A type assertion (`as User`) is *not* a runtime check — it tells the compiler "trust me." The TS Handbook on type assertions states: "TypeScript only allows type assertions which convert to a more specific or less specific version of a type." Going from `any`/`unknown` to a specific shape compiles, but the data may not match.
**Fix:** validate at the boundary with a parser (Zod, io-ts, valibot) or hand-rolled type predicate:
```typescript
import { z } from "zod";
const UserSchema = z.object({ id: z.string(), email: z.string().email() });
const u = UserSchema.parse(JSON.parse(input)); // throws if malformed
```
**Reference:** [TS Handbook — Type Assertions](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions); Matt Pocock, ["You should not use `as` in TypeScript"](https://www.totaltypescript.com/the-dangers-of-using-as-in-typescript).

### Bug 3 — excess property check only fires on fresh literals
**Root cause:** Excess property checking is a special-case rule for *fresh object literals* assigned to a typed target. Once the literal is assigned to a variable, the variable's inferred type widens to include the extra property, and structural compatibility silently accepts it. Documented in the TS Handbook section "Excess Property Checks."
**Fix:** type the variable explicitly, or use `satisfies`:
```typescript
const opts: Options = { timeout: 30, retires: 3 }; // error here
// or
const opts = { timeout: 30, retires: 3 } satisfies Options; // error here
```
**Reference:** [TS Handbook — Object Types: Excess Property Checks](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks).

### Bug 4 — `Array.find` result used without a null check
**Root cause:** `Array.prototype.find` is typed `(...) => T | undefined`. Under `strictNullChecks` accessing `.email` is an error; under non-strict it silently lets you crash at runtime. Even strict users sometimes write `admin!.email` (non-null assertion) and trade type safety for a runtime `TypeError`.
**Fix:**
```typescript
const admin = users.find(u => u.role === "admin");
if (!admin) throw new Error("no admin");
console.log(admin.email);
```
**Reference:** [TS Handbook — `strictNullChecks`](https://www.typescriptlang.org/tsconfig#strictNullChecks); MDN [`Array.prototype.find`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/find).

### Bug 5 — `Promise<T>` assigned where `T` was expected
**Root cause:** Forgetting `await` returns the Promise itself. TypeScript usually catches this — *unless* the consumer is typed `any`, accepts a wide union, or relies on duck typing. In `render(user)`, if `User` happens to allow extra properties (or `render` is typed `(user: any)`), TS emits no error and you render `[object Promise]`.
**Fix:** enable `@typescript-eslint/no-floating-promises` and `no-misused-promises`; always `await`:
```typescript
const user = await fetchUser();
render(user);
```
**Reference:** [TS Handbook — Async Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-1-7.html#async-await-support-in-es6-targets-node-v4); [`@typescript-eslint/no-floating-promises`](https://typescript-eslint.io/rules/no-floating-promises/).

### Bug 6 — `void` return type accepts any return
**Root cause:** A function-type parameter declared to return `void` does *not* mean "must return undefined" — it means "the caller will ignore the return value." This is intentional and documented in the TS Handbook ("Return type void"): it allows passing `arr.push` (which returns `number`) to `forEach` (which expects `(x: T) => void`). The trap is that on your own callbacks, you may rely on the returned value not existing.
**Fix:** if you need to *forbid* a return, type it `undefined` explicitly:
```typescript
type Logger = (msg: string) => undefined; // now `items.push` is rejected
```
**Reference:** [TS Handbook — Return type void](https://www.typescriptlang.org/docs/handbook/2/functions.html#return-type-void).

### Bug 7 — index signature widens to `any` without `noUncheckedIndexedAccess`
**Root cause:** Default behavior of `Record<string, string>` types `config["port"]` as `string` even when the key is missing. This is unsound — the value really is `undefined`. The fix is the `noUncheckedIndexedAccess` compiler option, which types every index access `T | undefined`.
**Fix:** in tsconfig:
```json
{ "compilerOptions": { "noUncheckedIndexedAccess": true } }
```
Then narrow:
```typescript
const port = config["port"];
if (port !== undefined) console.log(port.toUpperCase());
```
**Reference:** [tsconfig — `noUncheckedIndexedAccess`](https://www.typescriptlang.org/tsconfig#noUncheckedIndexedAccess); [TypeScript 4.1 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html).

### Bug 8 — narrowing lost across a function boundary
**Root cause:** A function returning `boolean` doesn't tell the control-flow analyzer anything about the argument's type. To narrow across a function call you need a *type predicate* — `s is { kind: "ready"; data: string }` — or an assertion function.
**Fix:**
```typescript
function isReady(s: State): s is Extract<State, { kind: "ready" }> {
  return s.kind === "ready";
}
```
**Reference:** [TS Handbook — Using type predicates](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates).

### Bug 9 — lying type predicate
**Root cause:** Type predicates (`x is T`) are unchecked by the compiler — TypeScript trusts the predicate body to do the runtime check. A predicate that returns `true` unconditionally compiles fine and is the canonical example of an unsound escape hatch. Documented in the same Handbook page; Matt Pocock and Stefan Baumgartner have both written warnings.
**Fix:** check the structure you're claiming:
```typescript
function isEmail(s: string): s is Email {
  return /^[^@]+@[^@]+\.[^@]+$/.test(s);
}
```
Better: use a parser library that returns a branded type only on success.
**Reference:** [TS Handbook — Using type predicates](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates); [fettblog.eu — TypeScript: assertion functions](https://fettblog.eu/typescript-assertion-signatures/).

### Bug 10 — discriminated union without exhaustiveness check
**Root cause:** Without an exhaustiveness check, adding a new variant to the union compiles silently and the function returns `undefined` for it. The `never` type makes the missing branch fail at compile time.
**Fix:**
```typescript
function describe(e: Event): string {
  switch (e.kind) {
    case "click": return "click";
    case "scroll": return "scroll";
    case "key": return "key";
    default: {
      const _exhaustive: never = e;
      throw new Error(`unhandled: ${(_exhaustive as Event).kind}`);
    }
  }
}
```
**Reference:** [TS Handbook — Exhaustiveness checking with `never`](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking).

### Bug 11 — method-shorthand bivariance vs property arrow
**Root cause:** Under `strictFunctionTypes`, function-type *properties* are checked contravariantly. But `strictFunctionTypes` deliberately exempts *method shorthand* on interfaces and class declarations — they remain bivariant for backward compatibility with the DOM lib. So `Handler` (using `on(evt: MouseEvent): void`) accepts `(e: Event) => void`, while `SafeHandler` (using a property arrow) does not. This is one of the most common "TS feels inconsistent" complaints; Anders Hejlsberg discussed it on the original `strictFunctionTypes` PR.
**Fix:** prefer property arrows when you want strict variance:
```typescript
interface SafeHandler {
  on: (evt: MouseEvent) => void;
}
```
**Reference:** [TS 2.6 release notes — `strictFunctionTypes`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-6.html#strict-function-types); [microsoft/TypeScript#18654](https://github.com/microsoft/TypeScript/pull/18654).

### Bug 12 — distributive conditional unintentionally distributing
**Root cause:** A conditional type `T extends U ? X : Y` where `T` is a *naked* type parameter distributes over unions in `T`. So `Boxed<string | number>` becomes `Boxed<string> | Boxed<number>`. To prevent this, wrap both sides in tuples.
**Fix:**
```typescript
type Boxed<T> = [T] extends [any] ? { value: T } : never;
type X = Boxed<string | number>; // { value: string | number }
```
**Reference:** [TS Handbook — Distributive Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types).

### Bug 13 — `keyof` over indexed access loses literal keys
**Root cause:** `Object.keys(o)` is typed `string[]` regardless of `o`'s type. The TypeScript team's stance, on issue [#12253](https://github.com/microsoft/TypeScript/issues/12253), is that returning `(keyof T)[]` would be unsound — the runtime object can have *more* keys than the type advertises (structural typing means subtypes carry extras), and TypeScript prefers a sound default over a convenient lie.
**Fix:** assert when you know the shape is closed (sealed), and prefer `for...in` with a guard, or use `Object.entries` with an explicit cast:
```typescript
(Object.keys(user) as (keyof User)[]).forEach(k => /* ... */);
```
Better: use a `satisfies`-validated `as const` map and iterate the const keys.
**Reference:** [microsoft/TypeScript#12253 — Object.keys returns string[]](https://github.com/microsoft/TypeScript/issues/12253).

### Bug 14 — mapped type `as` clause filters more than expected
**Root cause:** Key remapping via `as` was added in TS 4.1. Filtering by `K extends `_${string}` ? never : K` only matches *string* keys; numeric and symbol keys evaluate `K extends `_${string}`` to `false` and are kept. If your goal is "drop underscore-prefixed string keys, keep everything else," that's correct. If your goal is "string keys only," combine with `K extends string`.
**Fix:**
```typescript
type PublicShape<T> = {
  [K in keyof T as K extends string
    ? K extends `_${string}` ? never : K
    : never]: T[K];
};
```
**Reference:** [TS 4.1 release notes — Key Remapping in Mapped Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html#key-remapping-in-mapped-types).

### Bug 15 — `Readonly<T>` is shallow
**Root cause:** Built-in `Readonly<T>` is a one-level mapped type: `{ readonly [K in keyof T]: T[K] }`. Nested objects are unaffected. The TS Handbook explicitly notes this. `as const` is similarly shallow only on dynamic data — it's *deep* for object literals at the literal site, but the moment you assign a runtime value through it, only the outer shape carries `readonly`.
**Fix:** use a recursive helper or `as const`:
```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
const c: DeepReadonly<{ db: { host: string } }> = { db: { host: "localhost" } };
// c.db.host = "x"; // error
```
**Reference:** [TS Handbook — `Readonly<Type>`](https://www.typescriptlang.org/docs/handbook/utility-types.html#readonlytype).

### Bug 16 — `satisfies` doesn't narrow array literals
**Root cause:** `satisfies T` validates the expression *against* `T` while preserving the expression's inferred type — but the inferred type of `["GET", "POST"]` is `string[]` (widened), not `readonly ["GET", "POST"]`. To preserve literal narrowing, combine with `as const`, which Daniel Rosenwasser discussed in the 4.9 release notes and Matt Pocock has covered extensively.
**Fix:**
```typescript
const routes = ["GET", "POST"] as const satisfies readonly string[];
type R = typeof routes; // readonly ["GET", "POST"]
```
**Reference:** [TS 4.9 release notes — The `satisfies` Operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator); Matt Pocock, ["The `satisfies` operator"](https://www.totaltypescript.com/clarifying-the-satisfies-operator).

### Bug 17 — generic constraint over `keyof T` widens to `string | number | symbol`
**Root cause:** When the type parameter is *not* captured per-argument, `keyof T` collapses to a union and you lose per-element literal inference. A captured `K extends keyof T` is what makes `pick(user, ["id"])` narrow to `K = "id"`. Effective TypeScript, Item 14 ("Use Type Operations and Generics to Avoid Repeating Yourself") covers this pattern.
**Fix:**
```typescript
function pick<T, K extends keyof T>(obj: T, keys: readonly K[]): Pick<T, K> {
  const out = {} as Pick<T, K>;
  for (const k of keys) out[k] = obj[k];
  return out;
}
```
**Reference:** [TS Handbook — Generic Constraints](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-constraints); Effective TypeScript (Vanderkam), Item 14.

### Bug 18 — branded type forgotten on persistence boundary
**Root cause:** Branded types (`string & { __brand: "UserId" }`) are *purely* a compile-time construct — at runtime there is no brand. The compiler can't tell the difference between a validated `UserId` and `req.params.id as UserId`. The whole point of brands is to force every entry point through a *constructor function* that performs validation; the cast bypasses that.
**Fix:** export only a constructor:
```typescript
function userIdFromString(s: string): UserId {
  if (!/^u_[a-zA-Z0-9]{16}$/.test(s)) throw new Error("invalid user id");
  return s as UserId;
}
app.get("/users/:id", (req, res) => load(userIdFromString(req.params.id)));
```
**Reference:** [TypeScript Deep Dive — Nominal Typing](https://basarat.gitbook.io/typescript/main-1/nominaltyping); Matt Pocock, ["The `Brand` pattern"](https://www.totaltypescript.com/workshops/advanced-typescript-patterns).

### Bug 19 — declaration merging clobbers a library type
**Root cause:** Module augmentation (`declare module "express"`) merges your additions into the library's interface. Declaring `req.user` as non-optional means the type system assumes it's always set — but middleware ordering bugs, route-specific auth, or missing middleware all leave it `undefined` at runtime. The TS Handbook on declaration merging is explicit: merging is *additive*, not *checked*.
**Fix:** make augmentations optional and narrow:
```typescript
declare module "express" {
  interface Request { user?: { id: string } }
}
app.get("/me", (req, res) => {
  if (!req.user) return res.status(401).end();
  res.json(req.user.id);
});
```
**Reference:** [TS Handbook — Declaration Merging: Module Augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation).

### Bug 20 — recursive conditional type hits depth limit
**Root cause:** TypeScript caps conditional-type recursion to prevent infinite instantiation. Pre-4.5 the limit was ~50; TS 4.5 added *tail-recursion elimination on conditional types*, raising the practical limit. Even so, deeply nested instantiations error out with "Type instantiation is excessively deep."
**Fix:** rewrite to tail-recursive form (the recursive call is the entire branch result), and bound the recursion explicitly:
```typescript
type Flatten<T, Depth extends number = 10> =
  Depth extends 0 ? T :
  T extends readonly (infer U)[] ? Flatten<U, Prev<Depth>> : T;
```
**Reference:** [TS 4.5 release notes — Tail-Recursion Elimination on Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-5.html#tailrec-conditional-types); [microsoft/TypeScript#45711](https://github.com/microsoft/TypeScript/pull/45711).

### Bug 21 — `enum` reverse mapping leaks runtime keys
**Root cause:** Numeric enums emit a bidirectional object: `{ Admin: 1, User: 2, "1": "Admin", "2": "User" }`. So `Object.values(Role)` gives you the names *and* the numbers. String enums don't have this problem. The TS Handbook's enum page documents it; Anders Hejlsberg has acknowledged that enums "predate the rest of the type system" and recommends string-literal unions or `as const` objects for new code.
**Fix:** use a const object + literal union:
```typescript
const Role = { Admin: "admin", User: "user" } as const;
type Role = typeof Role[keyof typeof Role]; // "admin" | "user"
const all = Object.values(Role); // ["admin", "user"]
```
**Reference:** [TS Handbook — Enums (Reverse Mappings)](https://www.typescriptlang.org/docs/handbook/enums.html#reverse-mappings); [microsoft/TypeScript#1206 — string enums proposal context](https://github.com/microsoft/TypeScript/issues/1206).

### Bug 22 — class `private` is compile-time only
**Root cause:** TypeScript's `private` keyword is checked at compile time and erased at emit; the field is a normal property at runtime, fully accessible via `(obj as any).field` or `Object.entries`. ECMAScript private fields (`#name`) are enforced by the runtime — accessing them from outside the class throws.
**Fix:**
```typescript
class Account {
  #balance = 0;
  add(n: number) { this.#balance += n; }
}
const a = new Account();
// (a as any).balance; // undefined
// JSON.stringify(a); // "{}" — # fields are not own enumerable properties
```
**Reference:** [TS 3.8 release notes — ECMAScript Private Fields](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#ecmascript-private-fields); MDN [Private class features](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields).

### Bug 23 — type-only import accidentally elided at runtime
**Root cause:** With `verbatimModuleSyntax: true` (TS 5.0+, replaces `importsNotUsedAsValues` and `preserveValueImports`), TypeScript no longer guesses whether an import is type-only — `import` is kept verbatim, `import type` is erased verbatim. If you write `import type { Logger }` and then `new Logger()`, the import is erased and the runtime throws `ReferenceError: Logger is not defined`. Conversely, `import { Logger }` for a type-only use still emits the require/ESM import, which can pull in side effects you didn't want.
**Fix:** match the import form to the usage:
```typescript
import { Logger } from "./logger";   // value: kept at runtime
import type { Config } from "./config"; // type-only: erased
const log = new Logger();
const cfg: Config = { /* ... */ };
```
**Reference:** [TS 5.0 release notes — `--verbatimModuleSyntax`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#--verbatimmodulesyntax); [tsconfig — `verbatimModuleSyntax`](https://www.typescriptlang.org/tsconfig#verbatimModuleSyntax).

## Related
- [Structural Typing](./structural-typing.md)
- [Type Narrowing](./type-narrowing.md)
- [Conditional Types and `infer`](./conditional-types-and-infer.md)
- [Mapped Types](./mapped-types.md)
- [Generics and Constraints](./generics-and-constraints.md)
- [tsconfig Demystified](../tooling/tsconfig-demystified.md)

## References
- TypeScript Handbook — Narrowing: https://www.typescriptlang.org/docs/handbook/2/narrowing.html
- TypeScript Handbook — Conditional Types: https://www.typescriptlang.org/docs/handbook/2/conditional-types.html
- TypeScript Handbook — Object Types (Excess Property Checks): https://www.typescriptlang.org/docs/handbook/2/objects.html
- TypeScript Handbook — Functions (`void` return): https://www.typescriptlang.org/docs/handbook/2/functions.html
- TypeScript Handbook — Generics: https://www.typescriptlang.org/docs/handbook/2/generics.html
- TypeScript Handbook — Utility Types: https://www.typescriptlang.org/docs/handbook/utility-types.html
- TypeScript Handbook — Enums: https://www.typescriptlang.org/docs/handbook/enums.html
- TypeScript Handbook — Declaration Merging: https://www.typescriptlang.org/docs/handbook/declaration-merging.html
- TypeScript 2.6 release notes — `strictFunctionTypes`: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-6.html
- TypeScript 3.8 release notes — Private Fields, `import type`: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html
- TypeScript 4.1 release notes — Key Remapping, Template Literal Types: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html
- TypeScript 4.5 release notes — Tail-Recursion on Conditional Types: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-5.html
- TypeScript 4.9 release notes — `satisfies`: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html
- TypeScript 5.0 release notes — `verbatimModuleSyntax`: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html
- tsconfig reference — `noUncheckedIndexedAccess`: https://www.typescriptlang.org/tsconfig#noUncheckedIndexedAccess
- tsconfig reference — `strictNullChecks`: https://www.typescriptlang.org/tsconfig#strictNullChecks
- tsconfig reference — `verbatimModuleSyntax`: https://www.typescriptlang.org/tsconfig#verbatimModuleSyntax
- microsoft/TypeScript#12253 — `Object.keys` typing rationale: https://github.com/microsoft/TypeScript/issues/12253
- microsoft/TypeScript#18654 — strictFunctionTypes / method bivariance: https://github.com/microsoft/TypeScript/pull/18654
- microsoft/TypeScript#45711 — Tail-recursive conditional types: https://github.com/microsoft/TypeScript/pull/45711
- TypeScript Deep Dive (Basarat) — Nominal Typing: https://basarat.gitbook.io/typescript/main-1/nominaltyping
- Matt Pocock — Dangers of `as` in TypeScript: https://www.totaltypescript.com/the-dangers-of-using-as-in-typescript
- Matt Pocock — `satisfies` operator: https://www.totaltypescript.com/clarifying-the-satisfies-operator
- Stefan Baumgartner (fettblog.eu) — Assertion signatures: https://fettblog.eu/typescript-assertion-signatures/
- Effective TypeScript (Dan Vanderkam): https://effectivetypescript.com/
- `@typescript-eslint/no-floating-promises`: https://typescript-eslint.io/rules/no-floating-promises/
