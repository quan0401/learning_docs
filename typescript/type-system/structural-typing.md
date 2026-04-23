---
title: "Structural Typing & Type Compatibility"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, type-system, structural-typing]
---

# Structural Typing & Type Compatibility

**Date:** 2026-04-23 | **Updated:** 2026-04-23
**Tags:** `typescript` `type-system` `structural-typing`

## Table of Contents

- [Summary](#summary)
- [Structural vs Nominal Typing](#structural-vs-nominal-typing)
- [Excess Property Checks](#excess-property-checks)
- [Freshness](#freshness)
- [Assignability Rules](#assignability-rules)
- [Covariance and Contravariance](#covariance-and-contravariance)
- [Branded Types for Nominal Typing in TS](#branded-types-for-nominal-typing-in-ts)
- [The `satisfies` Operator](#the-satisfies-operator)
- [Related](#related)
- [References](#references)

---

## Summary

You define two interfaces with identical fields. You pass one where the other is expected, and TypeScript says nothing. Coming from Java, where `class Dog` and `class Cat` are never interchangeable regardless of their fields, this feels wrong. It is not wrong — it is **structural typing**, and it is the single most important mental model shift when moving from Java to TypeScript.

This doc covers how structural typing works, where it breaks your expectations (excess property checks), how to exploit it (assignability rules, variance), and how to escape it when it is too loose (branded types, `satisfies`).

**Java parallel:** Java uses **nominal typing** — type compatibility depends on the declared class or interface name and the `implements`/`extends` hierarchy. TypeScript uses **structural typing** — type compatibility depends only on the shape (the set of properties and their types). See [Java Type System for TypeScript Developers](../../java/java-fundamentals/type-system-for-ts-devs.md) for the full nominal-side explanation.

---

## Structural vs Nominal Typing

### The Surprise

```typescript
interface Dog {
  name: string;
  bark(): void;
}

interface Cat {
  name: string;
  bark(): void; // cats don't bark, but the shape matches
}

function greet(dog: Dog): string {
  return `Hello, ${dog.name}`;
}

const myCat: Cat = { name: "Whiskers", bark: () => console.log("meow?") };
greet(myCat); // No error. Cat is assignable to Dog.
```

TypeScript does not care that `myCat` was declared as `Cat`. It checks: does `myCat` have a `name: string` and a `bark(): void`? Yes. Assignment allowed.

### Why Java Refuses This

In Java, even with identical fields and methods, two classes are distinct types unless one explicitly extends or implements the other:

```java
// Java — nominal typing
class Dog { String name; void bark() {} }
class Cat { String name; void bark() {} }

void greet(Dog dog) { /* ... */ }
greet(new Cat()); // COMPILE ERROR: Cat is not Dog
```

The types are structurally identical but nominally different. Java requires an explicit relationship (`Cat extends Dog` or both implement a shared interface) to allow the call.

### When Structural Typing Is Powerful

Structural typing enables patterns that would require boilerplate interface hierarchies in Java:

```typescript
// Any object with a .length property works
function logLength(item: { length: number }): void {
  console.log(item.length);
}

logLength("hello");         // string has .length
logLength([1, 2, 3]);       // array has .length
logLength({ length: 42 });  // plain object with .length
```

In Java, you would need `String`, `List`, and your custom class to all implement a `HasLength` interface. In TypeScript, the shape is the contract.

### The Mental Model

Think of TypeScript types as **requirements lists**, not identity badges. When you write `function greet(dog: Dog)`, you are saying: "I need an object that has at least the properties listed in `Dog`." You are *not* saying: "I need an object that was constructed by a `Dog` factory."

---

## Excess Property Checks

### The Inconsistency That Is Not

Here is the code that confuses every Java developer on day one:

```typescript
interface Dog {
  name: string;
}

// FAILS — Object literal has excess property 'age'
const rex: Dog = { name: "Rex", age: 5 };
//                              ~~~ Error: 'age' does not exist in type 'Dog'

// WORKS — variable assignment, no excess check
const obj = { name: "Rex", age: 5 };
const rex2: Dog = obj; // No error
```

Same data, different behavior. Why?

### The Rationale

When you write an **object literal** directly in an assignment, TypeScript assumes you are describing the *complete* shape of the object. An extra property is almost certainly a typo or a mistake — you typed `age` when `Dog` has no `age`. The compiler catches it.

When you assign an **existing variable**, TypeScript reasons differently. The variable `obj` has type `{ name: string; age: number }`. That type is a *superset* of `Dog`. Structural typing says: "Does `obj` have at least a `name: string`? Yes." The extra `age` property is harmless — the function receiving a `Dog` will never access `age`, so it does not matter that `age` exists.

### Where Excess Checks Apply

Excess property checks trigger in these specific contexts:

```typescript
interface Config {
  host: string;
  port: number;
}

// 1. Direct variable assignment
const cfg: Config = { host: "localhost", port: 3000, retries: 3 }; // Error

// 2. Function argument (object literal)
function connect(cfg: Config) { /* ... */ }
connect({ host: "localhost", port: 3000, retries: 3 }); // Error

// 3. Return statement (object literal)
function makeConfig(): Config {
  return { host: "localhost", port: 3000, retries: 3 }; // Error
}
```

### Bypassing Excess Checks (When You Mean To)

Sometimes extra properties are intentional. Three legitimate escapes:

```typescript
// 1. Intermediate variable
const raw = { host: "localhost", port: 3000, retries: 3 };
const cfg: Config = raw; // OK

// 2. Type assertion
const cfg2: Config = { host: "localhost", port: 3000, retries: 3 } as Config; // OK

// 3. Index signature on the interface
interface FlexibleConfig {
  host: string;
  port: number;
  [key: string]: unknown; // allows any extra properties
}
```

Prefer option 1 (intermediate variable) in production code — it is explicit and does not suppress other type checking the way `as` does.

---

## Freshness

### What "Fresh" Means

TypeScript internally tracks whether an object type was created from an **object literal expression** in the current statement. This is called **freshness**.

- An object literal is **fresh** at the point of creation.
- Once assigned to a variable, spread into another object, or passed through a function, it **loses freshness**.
- Only **fresh** object types undergo excess property checks.

### Freshness in Action

```typescript
interface Point {
  x: number;
  y: number;
}

// Fresh — excess check fires
const p1: Point = { x: 1, y: 2, z: 3 }; // Error: 'z' is excess

// Loses freshness via variable
const raw = { x: 1, y: 2, z: 3 };
const p2: Point = raw; // OK — raw is not fresh

// Loses freshness via spread
const p3: Point = { ...{ x: 1, y: 2, z: 3 } }; // OK — spread result is not fresh

// Loses freshness via function return
function makePoint() {
  return { x: 1, y: 2, z: 3 };
}
const p4: Point = makePoint(); // OK — return value is not fresh
```

### Why This Matters in Real Code

In Express middleware or configuration builders, you often construct objects step by step:

```typescript
interface MiddlewareOptions {
  path: string;
  method: "GET" | "POST";
}

// This pattern is common and works because the merged object is not fresh
const defaults = { path: "/", method: "GET" as const };
const overrides = { path: "/api", extra: true };
const opts: MiddlewareOptions = { ...defaults, ...overrides }; // OK
```

The `extra` property survives because the spread result is not fresh. If you need to catch accidental extras in this pattern, use `satisfies` (covered below).

---

## Assignability Rules

### The Core Rule

Type `A` is assignable to type `B` when `A` has **at least** all the properties of `B`, and each property's type in `A` is assignable to the corresponding property's type in `B`.

```typescript
interface Shape {
  color: string;
}

interface Circle {
  color: string;
  radius: number;
}

let shape: Shape;
let circle: Circle = { color: "red", radius: 10 };

shape = circle; // OK: Circle has everything Shape needs (and more)
// circle = shape; // Error: Shape is missing 'radius'
```

This is the **width subtyping** rule: wider types (more properties) are assignable to narrower types (fewer properties). It mirrors the Liskov Substitution Principle — anywhere you expect a `Shape`, a `Circle` can stand in because it fulfills the `Shape` contract.

### Function Assignability

Functions follow different rules because of how they are *consumed*:

```typescript
type Handler = (event: MouseEvent) => void;

// Can I assign a function that accepts a broader type?
const generalHandler = (event: Event) => { console.log(event.type); };
const specificHandler = (event: MouseEvent) => { console.log(event.clientX); };

let handler: Handler;
handler = generalHandler;  // OK — accepts Event (broader), safe to call with MouseEvent
// handler = specificHandler; // Already the exact type — OK
```

The caller will pass a `MouseEvent`. A function that accepts `Event` (a supertype) is safe — it will never access properties that do not exist. This is **parameter contravariance**.

### Return Type Assignability

```typescript
type Factory = () => Shape;

const circleFactory = (): Circle => ({ color: "red", radius: 10 });
const shapeFactory: Factory = circleFactory; // OK — Circle is assignable to Shape
```

The caller expects a `Shape` back. Getting a `Circle` (which has everything `Shape` has, plus more) is safe. This is **return type covariance**.

### Optional Properties and Assignability

```typescript
interface Required {
  name: string;
  age: number;
}

interface WithOptional {
  name: string;
  age?: number; // optional
}

let req: Required = { name: "Alice", age: 30 };
let opt: WithOptional;

opt = req; // OK — Required satisfies WithOptional (age is present, which satisfies age?)
// req = opt; // Error — opt.age might be undefined, Required needs it defined
```

---

## Covariance and Contravariance

### The Problem

You have a function that processes animals. Can you pass it where a function that processes dogs is expected? The answer depends on whether we are talking about **input** (parameters) or **output** (return types).

### Covariance (Output Position) — "Narrowing Is Safe"

A type is **covariant** in a position if you can substitute a more specific (narrower) type.

```typescript
interface Animal { name: string; }
interface Dog extends Animal { breed: string; }

type AnimalFactory = () => Animal;
type DogFactory = () => Dog;

// Dog is narrower than Animal
// () => Dog is assignable to () => Animal — covariant return
const makeDog: DogFactory = () => ({ name: "Rex", breed: "Lab" });
const makeAnimal: AnimalFactory = makeDog; // OK
```

The caller of `makeAnimal` expects an `Animal`. It gets a `Dog`, which has `name` (everything `Animal` guarantees) plus `breed`. No property access will fail.

### Contravariance (Input Position) — "Widening Is Safe"

A type is **contravariant** in a position if you can substitute a more general (wider) type.

```typescript
type AnimalHandler = (a: Animal) => void;
type DogHandler = (d: Dog) => void;

const handleAnimal: AnimalHandler = (a) => { console.log(a.name); };
const handleDog: DogHandler = (d) => { console.log(d.breed); };

let dogHandler: DogHandler;
dogHandler = handleAnimal; // OK — handleAnimal accepts Animal (wider), safe to call with Dog
// dogHandler = handleDog;   // Already exact — OK

let animalHandler: AnimalHandler;
// animalHandler = handleDog; // UNSAFE — handleDog accesses .breed, but caller might pass a Cat
```

With `strictFunctionTypes` enabled (default in `strict` mode), TypeScript enforces contravariant parameter checking. Without it, parameters are checked **bivariantly** (both directions allowed), which is unsound but pragmatic for certain callback patterns.

### The `in` and `out` Variance Annotations (TS 4.7+)

You can explicitly declare variance on generic type parameters:

```typescript
// out = covariant (only appears in output positions)
interface Producer<out T> {
  produce(): T;
}

// in = contravariant (only appears in input positions)
interface Consumer<in T> {
  consume(item: T): void;
}

// in out = invariant (appears in both positions)
interface Processor<in out T> {
  process(item: T): T;
}
```

This helps the compiler check your generic types faster and catch variance violations at the declaration site rather than at each use site.

### Java Parallel: Wildcards

Java's variance annotations map directly:

| Concept | TypeScript | Java |
|---------|-----------|------|
| Covariant (read/produce) | `out T` | `<? extends T>` |
| Contravariant (write/consume) | `in T` | `<? super T>` |
| Invariant (read + write) | `in out T` | `<T>` (no wildcard) |

Java enforces variance at the *use site* (each variable declaration chooses `extends` or `super`). TypeScript's `in`/`out` enforce at the *declaration site* (the generic type itself declares its variance). Declaration-site variance is less flexible but catches errors earlier.

---

## Branded Types for Nominal Typing in TS

### The Problem

Structural typing is too permissive when different domain concepts share the same underlying type:

```typescript
type UserId = string;
type OrderId = string;

function getUser(id: UserId): User { /* ... */ }

const orderId: OrderId = "order-123";
getUser(orderId); // No error! Both are just 'string'
```

This is a real bug in production systems. A developer passes an order ID to a user lookup, the database returns nothing (or worse, the wrong thing), and the error is silent.

### The Branded Type Pattern

Add a phantom property that exists only at the type level:

```typescript
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

// Factory functions — the only way to create branded values
function UserId(id: string): UserId {
  return id as UserId;
}

function OrderId(id: string): OrderId {
  return id as OrderId;
}

function getUser(id: UserId): User { /* ... */ }

const userId = UserId("user-456");
const orderId = OrderId("order-123");

getUser(userId);   // OK
getUser(orderId);  // Error: OrderId is not assignable to UserId
getUser("raw-id"); // Error: string is not assignable to UserId
```

The `__brand` property never exists at runtime — it is purely a type-level discriminator. The `as` cast in the factory function is the controlled escape hatch.

### Numeric Branded Types

The same pattern works for numbers:

```typescript
type Celsius = number & { readonly __brand: "Celsius" };
type Fahrenheit = number & { readonly __brand: "Fahrenheit" };

function Celsius(value: number): Celsius { return value as Celsius; }
function Fahrenheit(value: number): Fahrenheit { return value as Fahrenheit; }

function boil(temp: Celsius): boolean {
  return temp >= 100;
}

boil(Celsius(100));      // OK
boil(Fahrenheit(212));   // Error — cannot mix temperature scales
boil(100);               // Error — raw number rejected
```

### Branded Types in Domain-Driven Design

Branded types are the TypeScript implementation of **Value Objects** with identity semantics. In a DDD codebase, every entity ID, every monetary amount, every email address should be branded to prevent accidental mixing. See [DDD Tactical Patterns in TypeScript](../patterns/ddd-tactical-patterns.md) for the full pattern with aggregate roots and repository interfaces.

### Validation at the Brand Boundary

The factory function is the natural place to validate:

```typescript
type Email = string & { readonly __brand: "Email" };

function Email(value: string): Email {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(value)) {
    throw new Error(`Invalid email: ${value}`);
  }
  return value as Email;
}

// Now Email is not just nominally distinct — it is validated
const email = Email("user@example.com"); // OK
const bad = Email("not-an-email");       // Throws at runtime
```

This pairs naturally with the error handling patterns in [Error Handling Architecture](../patterns/error-handling-architecture.md), where discriminated unions model success/failure without exceptions.

---

## The `satisfies` Operator

### The Problem with Type Annotations

Type annotations widen your values:

```typescript
interface Config {
  env: "development" | "staging" | "production";
  port: number;
  features: Record<string, boolean>;
}

// Type annotation — widens the type
const config: Config = {
  env: "production",
  port: 3000,
  features: { darkMode: true, betaAccess: false },
};

config.env;      // type: "development" | "staging" | "production" — widened!
config.features; // type: Record<string, boolean> — lost the specific keys
```

You wanted to validate that `config` matches `Config`, but the annotation widens the type. You lose the narrow literal `"production"` and the specific key names `"darkMode"` and `"betaAccess"`.

### `satisfies` Validates Without Widening

```typescript
const config = {
  env: "production",
  port: 3000,
  features: { darkMode: true, betaAccess: false },
} satisfies Config;

config.env;               // type: "production" — narrow literal preserved!
config.features.darkMode; // type: boolean — specific key access works!
config.features.nope;     // Error: Property 'nope' does not exist
```

`satisfies` checks that the value conforms to the type but does **not** use that type as the variable's type. The variable keeps its inferred narrow type.

### `satisfies` vs Type Annotation vs `as const`

| Approach | Validates against type? | Keeps narrow type? | Prevents excess properties? |
|----------|------------------------|-------------------|---------------------------|
| `const x: T = { ... }` | Yes | No (widens to T) | Yes (fresh object) |
| `const x = { ... } as const` | No | Yes (deep readonly) | No |
| `const x = { ... } satisfies T` | Yes | Yes | Yes (fresh object) |
| `const x = { ... } as const satisfies T` | Yes | Yes (deep readonly) | Yes |

### When to Use Each

**Type annotation** (`const x: Config = ...`) — When you want consumers to see the interface type, not the implementation details. Good for exported API boundaries.

**`as const`** — When you want deep immutability and literal types but do not need to validate against a specific interface. Good for constant lookup tables.

**`satisfies`** — When you want to validate the shape but keep the narrow inferred type for internal use. Good for configuration objects, route definitions, and exhaustive maps.

**`as const satisfies`** — When you want all three: validation, narrow types, and immutability. The strongest guarantee.

### Real-World: Route Definitions

```typescript
interface RouteConfig {
  path: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  auth: boolean;
}

// Without satisfies — typos in 'method' are not caught
const routes = {
  getUser:  { path: "/users/:id", method: "GET",  auth: true },
  postUser: { path: "/users",     method: "POSST", auth: true }, // typo not caught!
} as const;

// With satisfies — validation catches the typo
const routes2 = {
  getUser:  { path: "/users/:id", method: "GET",  auth: true },
  postUser: { path: "/users",     method: "POSST", auth: true }, // Error: "POSST" not in union
} as const satisfies Record<string, RouteConfig>;
```

### `satisfies` with Discriminated Unions

`satisfies` works well with discriminated unions from [Error Handling Architecture](../patterns/error-handling-architecture.md):

```typescript
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function fetchUser(): Result<User> {
  // satisfies ensures you return a valid Result shape
  // while keeping the narrow 'success: true' type for downstream narrowing
  return { success: true, data: { id: "1", name: "Alice" } } satisfies Result<User>;
}
```

---

## Related

### Sibling Type System Docs

- [Generics & Constraints](generics-and-constraints.md) — generic type parameters, `extends` constraints, conditional types
- [Type Narrowing & Control Flow](type-narrowing.md) — type guards, discriminated unions, control flow analysis
- [Recursive & Advanced Types](recursive-and-advanced-types.md) — template literal types, recursive conditionals, branded types explored deeper

### Cross-References

- [Java Type System for TypeScript Developers](../../java/java-fundamentals/type-system-for-ts-devs.md) — nominal vs structural from the Java perspective
- [Declaration Files & Type Acquisition](../tooling/declaration-files.md) — declaration merging relates to type compatibility; augmented types participate in structural checks
- [Error Handling Architecture](../patterns/error-handling-architecture.md) — discriminated unions and `Result` types used with `satisfies`
- [DDD Tactical Patterns in TypeScript](../patterns/ddd-tactical-patterns.md) — branded types for entity IDs and value objects

---

## References

- [TypeScript Handbook — Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)
- [TypeScript Handbook — Structural Type System](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html#structural-type-system)
- [TypeScript Handbook — Object Types (Excess Property Checks)](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks)
- [TypeScript 4.7 — Variance Annotations (`in` / `out`)](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#optional-variance-annotations-for-type-parameters)
- [TypeScript 4.9 — The `satisfies` Operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)
- [TypeScript Deep Dive — Freshness](https://basarat.gitbook.io/typescript/type-system/freshness)
- [Wikipedia — Covariance and Contravariance (computer science)](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))
