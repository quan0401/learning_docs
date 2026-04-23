---
title: "Recursive & Advanced Types"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, type-system, recursive-types, branded-types, advanced]
---

# Recursive & Advanced Types

**Date:** 2026-04-23 | **Updated:** 2026-04-23
**Tags:** `typescript` `type-system` `recursive-types` `branded-types` `advanced`

## Table of Contents

- [The Problem](#the-problem)
- [Recursive Types](#recursive-types)
- [Deep Utility Types](#deep-utility-types)
- [Variadic Tuple Types](#variadic-tuple-types)
- [`const` Assertions and Type Derivation](#const-assertions-and-type-derivation)
- [Branded / Opaque Types Deep Dive](#branded--opaque-types-deep-dive)
- [Template Literal Types (Advanced)](#template-literal-types-advanced)
- [Type-Level Programming Patterns](#type-level-programming-patterns)
- [Performance and Limits](#performance-and-limits)
- [Real-World Examples](#real-world-examples)
- [Related](#related)
- [References](#references)

---

## The Problem

You need to type a deeply nested JSON schema validator. The schema describes objects
containing objects containing arrays of objects, recursively. A flat type alias
cannot express this. You need types that reference themselves, types that parse
strings at the type level, and types that accumulate information through chained
method calls — all without a single `any`.

```typescript
// Goal: a type that accepts arbitrary valid JSON
type Json = /* ??? */;

// Goal: make every property readonly at every nesting depth
type DeepReadonly<T> = /* ??? */;

// Goal: extract route params from "/users/:id/posts/:postId"
type Params<Route extends string> = /* ??? */;
```

This document ties together
[conditional types](conditional-types-and-infer.md),
[mapped types](mapped-types.md), and
[generics](generics-and-constraints.md) into patterns that solve
real problems at arbitrary depth.

---

## Recursive Types

A recursive type alias references itself in its own definition. TypeScript
resolves these lazily — the compiler defers evaluation until the type is
actually used, which prevents infinite expansion.

### The `Json` type

```typescript
type Json =
  | string
  | number
  | boolean
  | null
  | Json[]
  | { [key: string]: Json };
```

This is valid because each branch of the union is either a terminal (primitive)
or a container that wraps `Json` again. The compiler does not try to expand `Json`
eagerly; it checks structural compatibility at each use site.

### How deferred resolution works

Before TypeScript 3.7, recursive type aliases were illegal — you needed an
intermediate `interface` to break the cycle:

```typescript
// Pre-3.7 workaround
interface JsonArray extends Array<Json> {}
interface JsonObject {
  [key: string]: Json;
}
type Json = string | number | boolean | null | JsonArray | JsonObject;
```

From 3.7 onward, the compiler detects that the self-reference sits inside
a structurally deferred position (inside an array type, object type, or tuple)
and defers resolution automatically. Self-references in *immediately resolved*
positions still error:

```typescript
// Error: Type alias 'Bad' circularly references itself
type Bad = Bad | string;
```

The rule: the recursive reference must appear inside a generic type argument,
an array/tuple element, or an object property — any position where the compiler
can defer without needing the full type immediately.

### Java comparison

Java generics achieve a similar pattern with `extends`:

```java
// Java recursive bound
interface Comparable<T extends Comparable<T>> { ... }
```

TypeScript recursive types are more flexible because structural typing means the
compiler never needs to resolve the full infinite tree — it only checks as deep
as the actual value requires.

---

## Deep Utility Types

Recursive types shine when you need to transform every property at every depth.
These combine recursive type aliases with
[mapped types](mapped-types.md).

### `DeepReadonly<T>`

```typescript
type DeepReadonly<T> =
  T extends ReadonlyArray<infer U>
    ? ReadonlyArray<DeepReadonly<U>>
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T;
```

Walk through the branches:
1. If `T` is an array, make it `ReadonlyArray` and recurse into the element type.
2. If `T` is an object, map every property to `readonly` and recurse into the value type.
3. Otherwise (primitives), return `T` unchanged.

### `DeepPartial<T>`

```typescript
type DeepPartial<T> =
  T extends object
    ? { [K in keyof T]?: DeepPartial<T[K]> }
    : T;
```

### `DeepRequired<T>`

```typescript
type DeepRequired<T> =
  T extends object
    ? { [K in keyof T]-?: DeepRequired<T[K]> }
    : T;
```

The `-?` modifier removes optionality — see
[mapped types](mapped-types.md) for details on modifier removal.

### Practical use: patch operations

```typescript
interface Config {
  database: {
    host: string;
    port: number;
    pool: { min: number; max: number };
  };
  logging: {
    level: "debug" | "info" | "warn" | "error";
    pretty: boolean;
  };
}

function applyPatch(base: Config, patch: DeepPartial<Config>): Config {
  // merge logic here
  return { ...base, ...patch } as Config;
}

// Only override the pool max — fully type-safe
applyPatch(defaultConfig, { database: { pool: { max: 50 } } });
```

---

## Variadic Tuple Types

Introduced in TypeScript 4.0, variadic tuple types let you spread tuple types
inside other tuples using `[...T]` syntax. This is the type-level equivalent of
the spread operator.

### `Concat`

```typescript
type Concat<A extends readonly unknown[], B extends readonly unknown[]> =
  [...A, ...B];

type Result = Concat<[1, 2], [3, 4]>; // [1, 2, 3, 4]
```

### Labeled tuple elements

TypeScript 4.0 also added labels for documentation and IDE hints:

```typescript
type Range = [start: number, end: number];
type LogEntry = [timestamp: Date, level: string, ...messages: string[]];
```

### Type-safe `zip`

```typescript
type Zip<A extends readonly unknown[], B extends readonly unknown[]> =
  A extends [infer HeadA, ...infer TailA]
    ? B extends [infer HeadB, ...infer TailB]
      ? [[HeadA, HeadB], ...Zip<TailA, TailB>]
      : []
    : [];

type Zipped = Zip<[string, number], [boolean, Date]>;
// [[string, boolean], [number, Date]]
```

This uses recursive [conditional types with `infer`](conditional-types-and-infer.md)
to peel off one element at a time from each tuple.

### Type-safe pipeline

```typescript
type PipeArgs<Fns extends readonly Function[], Acc = unknown> =
  Fns extends [infer First extends (...args: any) => any, ...infer Rest extends Function[]]
    ? [(...args: [Acc]) => ReturnType<First>, ...PipeArgs<Rest, ReturnType<First>>]
    : [];

// Simplified pipe that chains return types
function pipe<A, B>(f: (a: A) => B): (a: A) => B;
function pipe<A, B, C>(f: (a: A) => B, g: (b: B) => C): (a: A) => C;
function pipe<A, B, C, D>(
  f: (a: A) => B,
  g: (b: B) => C,
  h: (c: C) => D,
): (a: A) => D;
function pipe(...fns: Function[]) {
  return (arg: unknown) => fns.reduce((acc, fn) => fn(acc), arg);
}

// Type-safe — compiler verifies the chain
const transform = pipe(
  (s: string) => s.length,     // string => number
  (n: number) => n > 5,        // number => boolean
  (b: boolean) => b.toString() // boolean => string
);
```

---

## `const` Assertions and Type Derivation

The `as const` assertion tells the compiler to infer the narrowest possible type:
everything becomes `readonly`, and literal values stay as literal types instead of
widening to `string`, `number`, etc.

### Array to union

```typescript
const ROLES = ["admin", "editor", "viewer"] as const;
// Type: readonly ["admin", "editor", "viewer"]

type Role = (typeof ROLES)[number];
// "admin" | "editor" | "viewer"
```

Without `as const`, `ROLES` would be `string[]` and the derived union would be
`string` — useless for type safety.

### Config objects with literal types

```typescript
const HTTP_METHODS = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
  DELETE: "DELETE",
} as const;

type HttpMethod = (typeof HTTP_METHODS)[keyof typeof HTTP_METHODS];
// "GET" | "POST" | "PUT" | "DELETE"
```

### Deriving types from runtime values

This is one of the most powerful patterns in TypeScript: define the value once,
derive the type from it. No duplication, no drift.

```typescript
const EVENTS = {
  USER_CREATED: { topic: "user", action: "created" },
  USER_DELETED: { topic: "user", action: "deleted" },
  ORDER_PLACED: { topic: "order", action: "placed" },
} as const;

type EventName = keyof typeof EVENTS;
// "USER_CREATED" | "USER_DELETED" | "ORDER_PLACED"

type EventPayload<K extends EventName> = (typeof EVENTS)[K];
// Preserves literal types: { readonly topic: "user"; readonly action: "created" }
```

### Interaction with `satisfies`

Use `satisfies` to validate shape while preserving literal inference:

```typescript
const ROUTES = {
  home: "/",
  about: "/about",
  user: "/users/:id",
} as const satisfies Record<string, string>;
// Type preserves literal values but validates the shape
```

---

## Branded / Opaque Types Deep Dive

[Structural typing](structural-typing.md) means a `UserId` that is just a
`string` alias is freely assignable to a `PostId` that is also a `string`. Branded
types add a phantom property that makes them nominally distinct. This section goes
deeper than the basics covered in the structural typing doc.

### Strategy 1: Intersection brand

The simplest approach:

```typescript
type UserId = string & { readonly __brand: "UserId" };
type PostId = string & { readonly __brand: "PostId" };

function getUser(id: UserId): User { /* ... */ }

const userId = "abc" as UserId;
const postId = "abc" as PostId;

getUser(userId); // OK
getUser(postId); // Error: '__brand' types are incompatible
```

**Downside:** `__brand` is visible in autocomplete and shows up in `keyof`.

### Strategy 2: Unique symbol brand

Hides the brand from external access:

```typescript
declare const userIdBrand: unique symbol;
declare const postIdBrand: unique symbol;

type UserId = string & { readonly [userIdBrand]: typeof userIdBrand };
type PostId = string & { readonly [postIdBrand]: typeof postIdBrand };
```

Since `unique symbol` types are only equal to themselves, no code outside this
module can accidentally create a compatible brand. The symbol is `declare`-only,
so it generates zero runtime JavaScript.

### Strategy 3: Runtime-validated factory functions

Brands without validation are just `as` casts with extra steps. Combine brands
with validation for real safety:

```typescript
type UserId = string & { readonly __brand: "UserId" };

function createUserId(raw: string): UserId {
  if (!/^usr_[a-zA-Z0-9]{20}$/.test(raw)) {
    throw new Error(`Invalid user ID format: ${raw}`);
  }
  return raw as UserId;
}

// The only way to get a UserId is through the validated factory
const id = createUserId("usr_abc123def456ghi789jk"); // UserId
```

This pairs naturally with
[DDD tactical patterns](../patterns/ddd-tactical-patterns.md) where entity IDs
and value objects need both type-level distinction and runtime validation.

### Strategy 4: Composing brands

Brands compose via intersection:

```typescript
type Positive = number & { readonly __positive: true };
type Integer = number & { readonly __integer: true };
type PositiveInt = Positive & Integer;

function createPositiveInt(n: number): PositiveInt {
  if (!Number.isInteger(n) || n <= 0) {
    throw new Error(`Expected positive integer, got: ${n}`);
  }
  return n as PositiveInt;
}
```

### Building a multi-entity ID system

```typescript
// Generic brand factory
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type ProductId = Brand<string, "ProductId">;

// Generic validated constructor
function createId<B extends string>(
  brand: B,
  pattern: RegExp,
) {
  return (raw: string): Brand<string, B> => {
    if (!pattern.test(raw)) {
      throw new Error(`Invalid ${brand}: ${raw}`);
    }
    return raw as Brand<string, B>;
  };
}

const userId = createId("UserId", /^usr_[a-z0-9]{16}$/);
const orderId = createId("OrderId", /^ord_[a-z0-9]{16}$/);

const u = userId("usr_abc123def456gh");   // UserId
const o = orderId("ord_xyz789mno012pq");  // OrderId
```

---

## Template Literal Types (Advanced)

Basic template literal types are covered in
[mapped types](mapped-types.md). Here we push into recursive
template literal parsing.

### Recursive string splitting

```typescript
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Tail}`
    ? [Head, ...Split<Tail, D>]
    : [S];

type Parts = Split<"a.b.c", ".">;
// ["a", "b", "c"]
```

### Extracting route parameters

Parse a URL pattern like `"/users/:id/posts/:postId"` into a typed parameter
object at compile time:

```typescript
type ExtractParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & ExtractParams<Rest>
    : Path extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = ExtractParams<"/users/:id/posts/:postId">;
// { id: string } & { postId: string }
```

Walk through the recursion:
1. Match `"/users/:id/posts/:postId"` — `Param = "id"`, `Rest = "posts/:postId"`.
2. Recurse on `"posts/:postId"` — `Param = "postId"`, no more `/` after it.
3. Return `{ id: string } & { postId: string }`.

### Type-safe route builder

```typescript
type Route<Path extends string> = {
  path: Path;
  buildUrl: (params: ExtractParams<Path>) => string;
};

function defineRoute<const P extends string>(path: P): Route<P> {
  return {
    path,
    buildUrl: (params) => {
      let url: string = path;
      for (const [key, value] of Object.entries(params as Record<string, string>)) {
        url = url.replace(`:${key}`, value);
      }
      return url;
    },
  };
}

const userPostRoute = defineRoute("/users/:userId/posts/:postId");

// Fully typed — compiler demands both userId and postId
userPostRoute.buildUrl({ userId: "42", postId: "99" });
```

---

## Type-Level Programming Patterns

TypeScript's type system is Turing-complete (within recursion limits). These
patterns push that boundary for practical purposes.

### Type-level arithmetic with tuples

Tuples serve as the natural number encoding at the type level:

```typescript
type BuildTuple<N extends number, T extends unknown[] = []> =
  T["length"] extends N ? T : BuildTuple<N, [...T, unknown]>;

type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"];

type Sum = Add<3, 4>; // 7

type Subtract<A extends number, B extends number> =
  BuildTuple<A> extends [...BuildTuple<B>, ...infer Rest]
    ? Rest["length"]
    : never;

type Diff = Subtract<10, 3>; // 7
```

This works because `["length"]` on a tuple returns a numeric literal type.

### Type-level string manipulation

```typescript
type Join<T extends readonly string[], D extends string> =
  T extends readonly [infer Head extends string]
    ? Head
    : T extends readonly [infer Head extends string, ...infer Tail extends string[]]
      ? `${Head}${D}${Join<Tail, D>}`
      : "";

type Joined = Join<["users", "posts", "comments"], "/">;
// "users/posts/comments"

type Replace<S extends string, From extends string, To extends string> =
  S extends `${infer Before}${From}${infer After}`
    ? Replace<`${Before}${To}${After}`, From, To>
    : S;

type Cleaned = Replace<"hello--world--foo", "--", "-">;
// "hello-world-foo"
```

### Builder pattern with accumulated type information

```typescript
interface QueryBuilder<
  Table extends string = never,
  Selected extends string = never,
  Filtered extends string = never,
> {
  from<T extends string>(table: T): QueryBuilder<T, Selected, Filtered>;
  select<C extends string>(...cols: C[]): QueryBuilder<Table, Selected | C, Filtered>;
  where<C extends string>(col: C, op: string, val: unknown): QueryBuilder<Table, Selected, Filtered | C>;
  build(): { table: Table; selected: Selected; filtered: Filtered };
}

// Usage — the type accumulates through the chain
declare const qb: QueryBuilder;
const query = qb
  .from("users")              // QueryBuilder<"users", never, never>
  .select("name", "email")    // QueryBuilder<"users", "name" | "email", never>
  .where("age", ">", 18)     // QueryBuilder<"users", "name" | "email", "age">
  .build();

// query.table = "users", query.selected = "name" | "email"
```

### Type-safe SQL query builder sketch

```typescript
type Tables = {
  users: { id: number; name: string; email: string };
  orders: { id: number; userId: number; total: number };
};

type SelectResult<T extends keyof Tables, C extends keyof Tables[T]> =
  Pick<Tables[T], C>;

declare function select<
  T extends keyof Tables,
  C extends keyof Tables[T],
>(table: T, ...columns: C[]): SelectResult<T, C>[];

// Returns Pick<Tables["users"], "name" | "email">[]
const users = select("users", "name", "email");
//    ^? { name: string; email: string }[]
```

---

## Performance and Limits

### Recursive depth limits

TypeScript imposes a recursion depth limit of roughly 50 levels (the exact number
varies by TS version and the specific type structure). When exceeded:

```
Type instantiation is excessively deep and possibly infinite. ts(2589)
```

This is not configurable. `noErrorTruncation` in `tsconfig.json` only controls
error *message* display length — it does not raise the recursion limit.

### Tail-call optimization for types

TypeScript 4.5 added tail-recursive evaluation for conditional types. The
compiler detects when a type's recursive position is in the "tail" (the final
expression) and avoids building up a deep call stack:

```typescript
// Non-tail — accumulates intermediate types on the stack
type LengthBad<T extends unknown[], Acc extends unknown[] = []> =
  T extends [infer _, ...infer Rest]
    ? LengthBad<Rest, [...Acc, unknown]>  // not tail position
    : Acc["length"];

// Tail-recursive — TS optimizes this
type Length<T extends unknown[], N extends number = 0> =
  T extends [infer _, ...infer Rest]
    ? Length<Rest>  // tail position
    : T["length"];
```

Tail-recursive types can handle hundreds of elements instead of hitting the
~50 depth limit.

### Debugging slow types with `--generateTrace`

```bash
tsc --generateTrace ./trace-output
```

This produces Chrome trace files you can load in `chrome://tracing`. Look for:
- Types that take disproportionate time to check.
- Excessive instantiation counts — a sign of combinatorial explosion.
- Specific files or type aliases that dominate the trace.

### When type-level programming goes too far

**Simplify if:**
- Error messages become incomprehensible to your team.
- IDE autocompletion slows noticeably (>500ms delay).
- A `// @ts-expect-error` starts looking tempting.
- The type definition is longer than the runtime code it protects.

**The rule:** if your type error message would require a conference talk to
explain, the type is too clever. Extract named intermediate types, add comments
explaining the recursion, or simplify the constraint.

---

## Real-World Examples

### Deeply nested form validation types

```typescript
type ValidationRule<T> =
  T extends object
    ? { [K in keyof T]?: ValidationRule<T[K]> | ((value: T[K]) => string | null) }
    : (value: T) => string | null;

interface Address {
  street: string;
  city: string;
  geo: { lat: number; lng: number };
}

interface UserForm {
  name: string;
  address: Address;
}

const rules: ValidationRule<UserForm> = {
  name: (v) => (v.length < 2 ? "Too short" : null),
  address: {
    city: (v) => (v === "" ? "Required" : null),
    geo: {
      lat: (v) => (v < -90 || v > 90 ? "Invalid latitude" : null),
    },
  },
};
```

The `ValidationRule` type recurses into nested objects and offers either a
validator function or a nested rules object at each level.

### JSON Schema to TypeScript type (simplified)

```typescript
type JsonSchemaToType<S> =
  S extends { type: "string" } ? string :
  S extends { type: "number" } ? number :
  S extends { type: "boolean" } ? boolean :
  S extends { type: "null" } ? null :
  S extends { type: "array"; items: infer I } ? JsonSchemaToType<I>[] :
  S extends { type: "object"; properties: infer P }
    ? { [K in keyof P]: JsonSchemaToType<P[K]> }
    : unknown;

const userSchema = {
  type: "object",
  properties: {
    name: { type: "string" },
    age: { type: "number" },
    tags: { type: "array", items: { type: "string" } },
  },
} as const;

type User = JsonSchemaToType<typeof userSchema>;
// { name: string; age: number; tags: string[] }
```

This combines `as const`, recursive conditional types, and `infer` — every
major pattern from this document.

### Type-safe event emitter with namespace support

```typescript
type EventMap = {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "order:created": { orderId: string; total: number };
  "order:shipped": { orderId: string; trackingId: string };
};

type EventsInNamespace<
  Map,
  NS extends string,
> = {
  [K in keyof Map as K extends `${NS}:${infer Event}` ? Event : never]: Map[K];
};

type UserEvents = EventsInNamespace<EventMap, "user">;
// { login: { userId: string; timestamp: number }; logout: { userId: string } }

class TypedEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<string, Set<Function>>();

  on<K extends keyof Events & string>(
    event: K,
    handler: (payload: Events[K]) => void,
  ): this {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(handler);
    return this;
  }

  emit<K extends keyof Events & string>(event: K, payload: Events[K]): void {
    this.listeners.get(event)?.forEach((fn) => fn(payload));
  }
}

const emitter = new TypedEmitter<EventMap>();
emitter.on("user:login", (payload) => {
  // payload is { userId: string; timestamp: number }
  console.log(payload.userId);
});
```

### Express/Fastify route parameter extraction

```typescript
// Reusing ExtractParams from earlier
type ExtractParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & ExtractParams<Rest>
    : Path extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

interface TypedRequest<Path extends string> {
  params: ExtractParams<Path>;
  query: Record<string, string | undefined>;
  body: unknown;
}

type RouteHandler<Path extends string> = (
  req: TypedRequest<Path>,
  res: { json: (data: unknown) => void },
) => void;

function get<const P extends string>(
  path: P,
  handler: RouteHandler<P>,
): void {
  // register with framework
}

get("/users/:userId/posts/:postId", (req, res) => {
  // req.params is { userId: string } & { postId: string }
  const { userId, postId } = req.params;
  res.json({ userId, postId });
});
```

This is essentially what libraries like Fastify and Hono do under the hood to
provide type-safe route handlers. The
[error handling architecture](../patterns/error-handling-architecture.md) doc
shows how `Result` type chaining uses similar recursive patterns for
composing typed outcomes.

---

## Related

Sibling type-system docs:

- [Generics & Constraints](generics-and-constraints.md) — generic recursive types and constraint patterns
- [Conditional Types & `infer`](conditional-types-and-infer.md) — recursive types depend heavily on conditional type machinery
- [Mapped Types & Key Remapping](mapped-types.md) — recursive mapped types for deep transformations
- [Structural Typing](structural-typing.md) — branded types introduction and structural vs nominal comparison

Related pattern docs:

- [DDD Tactical Patterns](../patterns/ddd-tactical-patterns.md) — branded types for entity IDs and value objects
- [Error Handling Architecture](../patterns/error-handling-architecture.md) — `Result` type chaining with recursive patterns

---

## References

- [TypeScript Handbook — Recursive Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#recursive-conditional-types)
- [TypeScript Handbook — Variadic Tuple Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-0.html#variadic-tuple-types)
- [TypeScript Handbook — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [TypeScript Handbook — `const` assertions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#const-assertions)
- [TypeScript 4.5 — Tail-Recursion Elimination on Conditional Types](https://devblogs.microsoft.com/typescript/announcing-typescript-4-5/#tail-recursion-elimination-on-conditional-types)
- [TypeScript Performance Wiki — Compiler Performance](https://github.com/microsoft/TypeScript/wiki/Performance)
- [Matt Pocock — Branded Types](https://www.totaltypescript.com/branded-types)
- [Type Challenges — Community type-level exercises](https://github.com/type-challenges/type-challenges)
