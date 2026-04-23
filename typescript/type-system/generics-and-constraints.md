---
title: "Generics & Constraints in Practice"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, type-system, generics]
---

# Generics & Constraints in Practice

| | |
|---|---|
| **Problem** | Type a function that works with any entity that has an `id` |
| **Audience** | Intermediate TS/Node developer with deep Java knowledge |
| **Prereqs** | [Structural Typing & Type Compatibility](structural-typing.md), basic TS experience |
| **Bridges to** | [Java Type System for TS Developers](../../java/java-fundamentals/type-system-for-ts-devs.md) |

---

## 1. Generics Basics Refresher

You already know Java generics. Here is the mental model shift for TypeScript.

### Generic Functions

```typescript
// Java instinct: <T> T identity(T value)
// TS equivalent:
function identity<T>(value: T): T {
  return value;
}

const n = identity(42);        // T inferred as number (literal 42 widens)
const s = identity("hello");   // T inferred as string
```

No type erasure at the compiler level — the compiler sees `T` throughout the entire type-checking pass and resolves it fully. But at **runtime**, generics disappear just like Java. The emitted JS is plain `function identity(value) { return value; }`. The difference: TS resolves all generic relationships **before** erasure, so the compiler catches misuse that Java's erased runtime cannot.

### Generic Interfaces and Classes

```typescript
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<T>;
}

class InMemoryRepository<T extends { id: string }> implements Repository<T> {
  private store = new Map<string, T>();

  async findById(id: string): Promise<T | null> {
    return this.store.get(id) ?? null;
  }

  async save(entity: T): Promise<T> {
    this.store.set(entity.id, entity);
    return entity;
  }
}
```

### The Key Difference From Java

In Java, you write `<T extends Comparable<T>>` because the compiler needs a **nominal** bound to know what methods `T` has. In TS, constraints are **structural** — you describe the shape you need:

```typescript
// Java: <T extends Comparable<T>> int compare(T a, T b)
// TS: no interface needed — just describe the shape
function compare<T extends { compareTo(other: T): number }>(a: T, b: T): number {
  return a.compareTo(b);
}
```

This is structural typing applied to generics. The constraint `{ compareTo(other: T): number }` is an inline structural bound, not a reference to a named interface. See [Structural Typing & Type Compatibility](structural-typing.md) for the full assignability rules that govern this.

---

## 2. Constraints With `extends`

### The Motivating Problem

You want a function that works with any entity that has an `id`. Not a specific `User` or `Order` — anything with that shape.

```typescript
// Without generics — loses the specific type
function getId(entity: { id: string }): string {
  return entity.id;
}

const user = { id: "u1", name: "Alice", email: "alice@example.com" };
const result = getId(user); // string — but we lost the User type entirely
```

```typescript
// With generics — preserves the specific type
function getEntity<T extends { id: string }>(entity: T): T {
  console.log(`Processing entity ${entity.id}`);
  return entity;
}

const result = getEntity(user); // type is { id: string; name: string; email: string }
```

The `extends { id: string }` constraint guarantees `entity.id` is accessible inside the function body. The generic `T` preserves the caller's full type through the return.

### A Generic Repository Function

```typescript
interface HasId {
  readonly id: string;
}

interface HasTimestamp {
  readonly createdAt: Date;
  readonly updatedAt: Date;
}

// Single constraint
function findById<T extends HasId>(
  items: readonly T[],
  id: string
): T | undefined {
  return items.find((item) => item.id === id);
}

// Multiple constraints with intersection
function findRecent<T extends HasId & HasTimestamp>(
  items: readonly T[],
  since: Date
): T[] {
  return items.filter((item) => item.updatedAt >= since);
}
```

This pattern is the foundation of the generic `Entity<ID>` base in [DDD Tactical Patterns in TypeScript](../patterns/ddd-tactical-patterns.md).

### Constraining to Specific Types

```typescript
// T can only be string or number
function formatId<T extends string | number>(id: T): string {
  return String(id);
}

formatId("abc");  // OK
formatId(42);     // OK
formatId(true);   // Error: boolean is not assignable to string | number
```

### `keyof` Constraints

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: "u1", name: "Alice", age: 30 };
const name = getProperty(user, "name");  // type: string
const age = getProperty(user, "age");    // type: number
getProperty(user, "email");              // Error: "email" is not assignable to "id" | "name" | "age"
```

---

## 3. Generic Inference

TypeScript infers type arguments from usage. You rarely need to write `<string>` explicitly.

### Where Inference Works Well

```typescript
function merge<T, U>(a: T, b: U): T & U {
  return { ...a, ...b };
}

// Both T and U inferred from the arguments
const merged = merge({ id: "u1" }, { name: "Alice" });
// type: { id: string } & { name: string }
```

```typescript
function map<T, U>(items: T[], fn: (item: T) => U): U[] {
  return items.map(fn);
}

// T inferred from items, U inferred from fn's return
const lengths = map(["hello", "world"], (s) => s.length);
// type: number[]
```

### Where Inference Fails

**Problem 1: Return type cannot drive inference backwards.**

```typescript
function createEmpty<T>(): T[] {
  return [];
}

// T has nothing to infer from — defaults to unknown
const items = createEmpty();         // unknown[]
const items2 = createEmpty<string>(); // string[] — explicit annotation needed
```

**Problem 2: Complex nested generics lose precision.**

```typescript
function wrap<T>(value: T): { data: T } {
  return { data: value };
}

function processWrapped<T>(wrapped: { data: T }): T {
  return wrapped.data;
}

// This works — inference chains through
const result = processWrapped(wrap(42)); // number

// But deeply nested chains or conditional returns may need help
```

**Problem 3: Object literals widen types.**

```typescript
function register<T extends { role: string }>(config: T): T {
  return config;
}

const admin = register({ role: "admin", level: 5 });
// admin.role is string, not "admin" — the literal widened
```

### The `satisfies` Trick

`satisfies` checks a type constraint **without widening**:

```typescript
const adminConfig = {
  role: "admin" as const,
  level: 5,
} satisfies { role: string };

// adminConfig.role is "admin" (literal type preserved)
// adminConfig.level is number (not lost)
```

### The `const` Type Parameter (TS 5.0+)

```typescript
function register<const T extends { role: string }>(config: T): T {
  return config;
}

const admin = register({ role: "admin", level: 5 });
// admin.role is "admin" — the const modifier preserves literals
```

---

## 4. Default Type Parameters

### Basic Defaults

```typescript
type ApiResponse<TData = unknown, TError = string> = {
  readonly success: boolean;
  readonly data: TData | null;
  readonly error: TError | null;
  readonly timestamp: Date;
};

// Use with defaults
const response: ApiResponse = {
  success: true,
  data: null,
  error: null,
  timestamp: new Date(),
};

// Override just the first parameter
const userResponse: ApiResponse<User> = {
  success: true,
  data: { id: "u1", name: "Alice" },
  error: null,
  timestamp: new Date(),
};

// Override both
const detailedResponse: ApiResponse<User, ApiError> = {
  success: false,
  data: null,
  error: { code: 404, message: "Not found" },
  timestamp: new Date(),
};
```

### Configurable Utility Types

```typescript
type PaginatedResponse<
  TItem,
  TMeta = { total: number; page: number; limit: number }
> = {
  readonly items: readonly TItem[];
  readonly meta: TMeta;
};

// Default meta is usually fine
type UserPage = PaginatedResponse<User>;

// Custom meta when you need cursor pagination
type UserCursorPage = PaginatedResponse<User, { cursor: string; hasMore: boolean }>;
```

### Defaults With Constraints

```typescript
// Default must satisfy the constraint
type Store<T extends Record<string, unknown> = Record<string, string>> = {
  get<K extends keyof T>(key: K): T[K];
  set<K extends keyof T>(key: K, value: T[K]): void;
};
```

---

## 5. Function Overloads vs Generics

### When to Use Overloads

Overloads are right when the **return type changes based on specific input values**, not just input types:

```typescript
// Overloads: return type depends on the format argument's VALUE
function parseDate(input: string, format: "iso"): Date;
function parseDate(input: string, format: "unix"): number;
function parseDate(input: string, format: "iso" | "unix"): Date | number {
  if (format === "iso") return new Date(input);
  return Date.parse(input);
}

const d = parseDate("2026-04-23", "iso");   // Date
const n = parseDate("2026-04-23", "unix");  // number
```

### When to Use Generics

Generics are right when the function is **truly polymorphic** — it works the same way regardless of type:

```typescript
// Generic: same logic for any T
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

const n = first([1, 2, 3]);        // number | undefined
const s = first(["a", "b", "c"]);  // string | undefined
```

### The Same Function Both Ways

**Scenario:** A `createElement` that returns different element types.

```typescript
// Overload approach — explicit mapping
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "span"): HTMLSpanElement;
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

// Generic approach — uses a type map
type TagMap = HTMLElementTagNameMap;

function createEl<K extends keyof TagMap>(tag: K): TagMap[K] {
  return document.createElement(tag);
}
```

The generic version is more extensible — `HTMLElementTagNameMap` already contains every tag. Overloads are better when you have a small, fixed set of cases with genuinely different logic.

**Rule of thumb:** If you are writing more than 3 overload signatures, you probably want a generic with a type map instead.

---

## 6. Generic Utility Types

### Build Your Own

```typescript
// Nullable<T> — T or null
type Nullable<T> = T | null;

// DeepPartial<T> — recursively make all properties optional
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// KeysOfType<T, V> — extract keys whose values match type V
type KeysOfType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

// Usage
interface User {
  id: string;
  name: string;
  age: number;
  active: boolean;
}

type StringKeys = KeysOfType<User, string>; // "id" | "name"
type NumberKeys = KeysOfType<User, number>; // "age"
```

### How the Built-ins Work

```typescript
// Partial<T> — all properties become optional
type Partial<T> = { [P in keyof T]?: T[P] };

// Required<T> — all properties become required
type Required<T> = { [P in keyof T]-?: T[P] };

// Pick<T, K> — select specific properties
type Pick<T, K extends keyof T> = { [P in K]: T[P] };

// Omit<T, K> — remove specific properties
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

// Record<K, V> — object type with keys K and values V
type Record<K extends keyof any, T> = { [P in K]: T };

// Readonly<T> — all properties become readonly
type Readonly<T> = { readonly [P in keyof T]: T[P] };
```

These are all **mapped types**. The `[P in keyof T]` syntax iterates over every key and transforms it. This bridges directly to the next doc on mapped types and key remapping.

---

## 7. Advanced Patterns

### Generic Factories

```typescript
interface Entity {
  readonly id: string;
}

type EntityFactory<T extends Entity> = (props: Omit<T, "id">) => T;

function createFactory<T extends Entity>(
  generateId: () => string
): EntityFactory<T> {
  return (props) => ({ ...props, id: generateId() } as T);
}

interface User extends Entity {
  readonly name: string;
  readonly email: string;
}

const createUser = createFactory<User>(() => crypto.randomUUID());
const user = createUser({ name: "Alice", email: "alice@example.com" });
// user: User with a generated id
```

### Builder Pattern With Generics

Each `.with()` call **narrows the type**, so the compiler knows which fields have been set:

```typescript
type RequiredFields = "host" | "port" | "database";

class ConnectionBuilder<Set extends string = never> {
  private config: Record<string, unknown> = {};

  with<K extends RequiredFields>(
    key: K,
    value: K extends "port" ? number : string
  ): ConnectionBuilder<Set | K> {
    return Object.assign(
      new ConnectionBuilder<Set | K>(),
      { config: { ...this.config, [key]: value } }
    );
  }

  // build() is only callable when all required fields are set
  build(this: ConnectionBuilder<RequiredFields>): ConnectionConfig {
    return this.config as ConnectionConfig;
  }
}

interface ConnectionConfig {
  host: string;
  port: number;
  database: string;
}

const config = new ConnectionBuilder()
  .with("host", "localhost")
  .with("port", 5432)
  .with("database", "myapp")
  .build(); // OK — all fields set

const incomplete = new ConnectionBuilder()
  .with("host", "localhost")
  // .build(); // Error — missing "port" and "database"
```

### Generic Type Guards

```typescript
function isNonNull<T>(value: T | null | undefined): value is T {
  return value != null;
}

function hasProperty<T extends object, K extends string>(
  obj: T,
  key: K
): obj is T & Record<K, unknown> {
  return key in obj;
}

const items: (string | null)[] = ["a", null, "b", null, "c"];
const filtered = items.filter(isNonNull); // string[]

const data: unknown = { id: "u1", name: "Alice" };
if (hasProperty(data as object, "id")) {
  console.log(data.id); // OK — narrowed to { id: unknown }
}
```

### Higher-Kinded Type Simulation

TypeScript does not have native HKTs (like Haskell's `* -> *`). The community workaround uses a **Kind registry pattern**:

```typescript
// Step 1: Define a URI-indexed registry
interface URItoKind<A> {
  Array: Array<A>;
  Promise: Promise<A>;
  Nullable: A | null;
}

type URIS = keyof URItoKind<unknown>;
type Kind<F extends URIS, A> = URItoKind<A>[F];

// Step 2: Write generic functions over any "container"
interface Functor<F extends URIS> {
  map<A, B>(fa: Kind<F, A>, f: (a: A) => B): Kind<F, B>;
}

// Step 3: Implement for specific containers
const arrayFunctor: Functor<"Array"> = {
  map: (fa, f) => fa.map(f),
};

const result = arrayFunctor.map([1, 2, 3], (n) => n * 2); // number[]
```

This is the pattern used by libraries like `fp-ts`. It is a workaround, not first-class syntax — use it when you genuinely need abstraction over type constructors, not as a default.

---

## 8. TS Generics vs Java Generics

Coming from Java, these are the critical differences. See [Java Type System for TypeScript Developers](../../java/java-fundamentals/type-system-for-ts-devs.md) for the full Java side.

| Aspect | TypeScript | Java |
|--------|-----------|------|
| **Erasure** | Erased at emit to JS, but fully resolved at compile time | Erased to bounds at compile time; runtime has no type info |
| **Constraints** | Structural: `<T extends { id: string }>` | Nominal: `<T extends Identifiable>` |
| **Wildcards** | Not needed — union and intersection types cover the same ground | `<? extends T>`, `<? super T>` for use-site variance |
| **Variance** | Declaration-site with `in`/`out` (TS 4.7+), otherwise structural | Use-site via wildcards; declaration-site for some interfaces |
| **Reification** | Not available at runtime — `instanceof T` is impossible | Not available at runtime (type erasure) |
| **Primitive support** | Generics work with all types including primitives | `List<int>` impossible, must use `List<Integer>` (boxing) |
| **Default params** | `<T = string>` supported | Not supported |
| **Const type params** | `<const T>` preserves literal types | No equivalent |
| **Type inference** | Aggressive — infers from arguments, return context, constraints | Limited — diamond operator `<>`, some method inference |

### Variance: `in`/`out` vs Wildcards

```typescript
// TS 4.7+ declaration-site variance
interface Producer<out T> {  // covariant — only produces T
  get(): T;
}

interface Consumer<in T> {   // contravariant — only consumes T
  accept(value: T): void;
}

// Java equivalent:
// interface Producer<T> { T get(); }
//   used as: Producer<? extends Animal>
// interface Consumer<T> { void accept(T value); }
//   used as: Consumer<? super Animal>
```

In practice, TS infers variance structurally even without `in`/`out`. The annotations are an optimization hint for the compiler and a documentation signal for humans.

### What Java Has That TS Doesn't

- **Runtime type checks on generics** — neither language has this (both erase), but Java's reflection can recover some bound information. TS has zero runtime trace.
- **Bounded wildcards for flexible APIs** — TS uses union/intersection types and conditional types instead, which are often more powerful but have a steeper learning curve.

### What TS Has That Java Doesn't

- **Conditional types**: `type IsString<T> = T extends string ? true : false`
- **Mapped types**: `{ [K in keyof T]: Wrapper<T[K]> }`
- **Template literal types**: `` type Route = `/${string}` ``
- **`infer` keyword**: Extract types from inside other types
- **Default type parameters**: `<T = string>`

These capabilities make TS generics a type-level programming language. Java generics are a constraint system. Different tools, different power profiles.

---

## Practical Patterns From This Repo

The Prisma Client extension API ([Prisma Deep Dive](../data-access/prisma-deep-dive.md)) is one of the most advanced real-world uses of TS generics you will encounter. The `$extends` method uses deeply nested generics, conditional types, and mapped types to type-safely extend the client with custom methods — a case where understanding constraints, inference, and utility types all come together.

---

## Related

- [Structural Typing & Type Compatibility](structural-typing.md) — the assignability system that underlies generic constraints
- [Java Type System for TypeScript Developers](../../java/java-fundamentals/type-system-for-ts-devs.md) — Java generics, type erasure, and the nominal side of the comparison
- [DDD Tactical Patterns in TypeScript](../patterns/ddd-tactical-patterns.md) — `Entity<ID>` generic base class, branded types as type-safe IDs
- [Prisma Deep Dive](../data-access/prisma-deep-dive.md) — Prisma Client extension API as advanced generics in the wild
- Conditional Types & `infer` _(next in Tier 2)_ — extends the patterns from section 6
- Mapped Types & Key Remapping _(Tier 2)_ — extends the utility type implementations from section 6

## References

- [TypeScript Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TypeScript Handbook: Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [TypeScript 4.7 Release Notes: Variance Annotations](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#optional-variance-annotations-for-type-parameters)
- [TypeScript 5.0 Release Notes: const Type Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#const-type-parameters)
- [TypeScript Handbook: Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [fp-ts Kind Pattern](https://gcanti.github.io/fp-ts/guides/HKT.html) — higher-kinded type simulation in practice
