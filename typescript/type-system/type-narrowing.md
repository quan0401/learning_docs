---
title: "Type Narrowing & Control Flow"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, type-system, narrowing, discriminated-unions]
---

# Type Narrowing & Control Flow

**Date:** 2026-04-23 | **Updated:** 2026-04-23
**Tags:** `typescript` `type-system` `narrowing` `discriminated-unions`

## Table of Contents

- [Summary](#summary)
- [The Problem: Impossible States That Compile](#the-problem-impossible-states-that-compile)
- [Built-in Narrowing](#built-in-narrowing)
- [Discriminated Unions](#discriminated-unions)
- [Exhaustive Checking with `never`](#exhaustive-checking-with-never)
- [Type Predicates (`is`)](#type-predicates-is)
- [Assertion Functions (`asserts`)](#assertion-functions-asserts)
- [`in` Operator Narrowing](#in-operator-narrowing)
- [`satisfies` for Narrowing](#satisfies-for-narrowing)
- [Real-World Patterns](#real-world-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

You write a function that handles events. The `event` parameter is typed as `string | Event | null`. Inside the function body you check `if (event !== null)` — and suddenly TypeScript knows `event` is `string | Event` in that branch. You did not cast anything. The compiler traced your control flow and **narrowed** the type automatically.

This document covers every narrowing mechanism TypeScript provides, builds toward **discriminated unions** as the most important pattern for domain modeling, and shows how `never` turns the compiler into a safety net that catches missing cases at build time — not in production at 3 AM.

Coming from Java: you know sealed classes and pattern matching (Java 21+). TypeScript's discriminated unions + exhaustive `switch` achieve the same thing, but they arrived years earlier and are more deeply integrated into the type system.

---

## The Problem: Impossible States That Compile

Consider a naive event system:

```typescript
// BAD: impossible states are representable
interface AppEvent {
  type: string;
  userId?: string;
  orderId?: string;
  amount?: number;
  reason?: string;
}

function handleEvent(event: AppEvent) {
  if (event.type === "payment_failed") {
    // Is event.reason guaranteed to exist? No.
    // Is event.amount guaranteed to exist? No.
    // TypeScript cannot help you here.
    console.log(event.reason!.toUpperCase()); // runtime 💥
  }
}
```

Every field is optional. Every combination is valid at the type level. The compiler cannot distinguish a `UserCreated` from a `PaymentFailed`. You are back to runtime checks and `!` assertions — JavaScript with extra syntax.

The goal: **make impossible states unrepresentable**. When TypeScript knows the event type, it should know exactly which fields exist.

---

## Built-in Narrowing

TypeScript builds a **control flow graph** of your code. At every branch point (`if`, `switch`, ternary, `||`, `??`), it tracks which types are still possible. This is not a simple "check then cast" — it is flow-sensitive type analysis that updates continuously.

### `typeof` Guards

The most basic narrowing. Works for primitives.

```typescript
function formatValue(value: string | number | boolean): string {
  if (typeof value === "string") {
    // TS knows: value is string
    return value.toUpperCase();
  }
  if (typeof value === "number") {
    // TS knows: value is number
    return value.toFixed(2);
  }
  // TS knows: value is boolean (only option left)
  return value ? "yes" : "no";
}
```

`typeof` narrows to: `"string"`, `"number"`, `"bigint"`, `"boolean"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`. Note that `typeof null === "object"` — a famous JS bug that TypeScript inherits.

### `instanceof` Guards

For class instances.

```typescript
class HttpError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

class ValidationError extends Error {
  constructor(public fields: string[]) {
    super(`Invalid fields: ${fields.join(", ")}`);
  }
}

function handleError(err: Error): void {
  if (err instanceof HttpError) {
    // TS knows: err is HttpError
    console.log(`HTTP ${err.statusCode}: ${err.message}`);
  } else if (err instanceof ValidationError) {
    // TS knows: err is ValidationError
    console.log(`Bad fields: ${err.fields}`);
  } else {
    // TS knows: err is Error (base class)
    console.log(err.message);
  }
}
```

Java parallel: `instanceof` with pattern matching (`if (err instanceof HttpError he)`). Same idea, TypeScript just narrows the existing variable rather than introducing a new binding.

### Truthiness Narrowing

```typescript
function greet(name: string | null | undefined): string {
  if (name) {
    // TS knows: name is string (null, undefined, and "" are falsy)
    return `Hello, ${name}`;
  }
  return "Hello, stranger";
}
```

**Watch out**: truthiness eliminates `null`, `undefined`, `0`, `""`, `NaN`, and `false`. If `0` or `""` are valid values, truthiness narrowing removes them incorrectly.

```typescript
function processCount(count: number | null): number {
  // BAD: if count is 0, this branch is skipped
  if (count) {
    return count * 2;
  }
  // GOOD: explicit null check
  if (count !== null) {
    return count * 2; // count is number, including 0
  }
  return -1;
}
```

### Equality Narrowing

`===`, `!==`, `==`, `!=` all narrow.

```typescript
function process(value: string | null | undefined): void {
  if (value === null) {
    // TS knows: value is null
    return;
  }
  if (value === undefined) {
    // TS knows: value is undefined
    return;
  }
  // TS knows: value is string
  console.log(value.trim());
}

// Shortcut: == null checks for both null and undefined
function processShorter(value: string | null | undefined): void {
  if (value == null) {
    // TS knows: value is null | undefined
    return;
  }
  // TS knows: value is string
  console.log(value.trim());
}
```

### The Control Flow Graph

TypeScript does not just check the immediate `if` block. It tracks narrowing across sequential statements, including early returns:

```typescript
function parse(input: string | null): number {
  if (input === null) {
    throw new Error("Input required");
  }
  // After the throw, TS knows: input is string for ALL remaining code
  // Not just "inside an else block"

  if (input.length === 0) {
    return 0;
  }
  // TS knows: input is string AND input.length !== 0
  return parseInt(input, 10);
}
```

This is why early returns are idiomatic in TypeScript — they narrow the type for the entire rest of the function body.

---

## Discriminated Unions

This is **the** most important narrowing pattern in TypeScript. If you learn one thing from this document, learn this.

A discriminated union is a union of types that share a common literal field (the **discriminant**). When you check the discriminant, TypeScript narrows to the exact variant.

### Basic Shape Example

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}

interface Square {
  kind: "square";
  sideLength: number;
}

interface Triangle {
  kind: "triangle";
  base: number;
  height: number;
}

type Shape = Circle | Square | Triangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      // TS knows: shape is Circle
      return Math.PI * shape.radius ** 2;
    case "square":
      // TS knows: shape is Square
      return shape.sideLength ** 2;
    case "triangle":
      // TS knows: shape is Triangle
      return (shape.base * shape.height) / 2;
  }
}
```

No casts. No optional fields. No runtime errors from accessing a property that does not exist. The type system proves correctness.

### Event System — The Real Payoff

Now back to the original problem. Redesign the event system with discriminated unions:

```typescript
interface UserCreated {
  type: "user_created";
  userId: string;
  email: string;
  createdAt: Date;
}

interface OrderPlaced {
  type: "order_placed";
  orderId: string;
  userId: string;
  items: ReadonlyArray<{ sku: string; quantity: number }>;
  total: number;
}

interface PaymentFailed {
  type: "payment_failed";
  orderId: string;
  amount: number;
  reason: string;
  retryable: boolean;
}

interface OrderShipped {
  type: "order_shipped";
  orderId: string;
  trackingNumber: string;
  carrier: string;
}

type AppEvent = UserCreated | OrderPlaced | PaymentFailed | OrderShipped;
```

Every variant carries exactly the fields it needs. No optional fields. No impossible combinations. See [DDD Tactical Patterns](../patterns/ddd-tactical-patterns.md) for how domain events map to discriminated unions, and [CQRS & Event Patterns](../patterns/cqrs-and-events.md) for how command and event types use this same structure.

```typescript
function handleEvent(event: AppEvent): void {
  switch (event.type) {
    case "user_created":
      // TS knows: event is UserCreated
      sendWelcomeEmail(event.email);
      break;
    case "order_placed":
      // TS knows: event is OrderPlaced
      reserveInventory(event.items);
      break;
    case "payment_failed":
      // TS knows: event is PaymentFailed
      if (event.retryable) {
        scheduleRetry(event.orderId, event.amount);
      } else {
        notifySupport(event.orderId, event.reason);
      }
      break;
    case "order_shipped":
      // TS knows: event is OrderShipped
      sendTrackingEmail(event.orderId, event.trackingNumber);
      break;
  }
}
```

### Rules for Discriminated Unions

1. **The discriminant must be a literal type** — `"circle"`, `42`, `true`, not `string` or `number`.
2. **Every variant must have the discriminant field** — if one variant omits `type`, narrowing breaks.
3. **Use `readonly` on the discriminant** to prevent accidental reassignment.
4. **Prefer string literals** over numeric or boolean discriminants — they are self-documenting and appear in error messages.

---

## Exhaustive Checking with `never`

The discriminated union is powerful. But what happens when you add a new variant six months later and forget to handle it?

```typescript
// Someone adds this new event variant
interface OrderCancelled {
  type: "order_cancelled";
  orderId: string;
  cancelledBy: string;
  refundAmount: number;
}

type AppEvent =
  | UserCreated
  | OrderPlaced
  | PaymentFailed
  | OrderShipped
  | OrderCancelled; // new!
```

Without exhaustive checking, the `handleEvent` function silently ignores `OrderCancelled`. The compiler says nothing. You find out in production.

### The `assertNever` Pattern

```typescript
function assertNever(value: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(value)}`);
}

function handleEvent(event: AppEvent): void {
  switch (event.type) {
    case "user_created":
      sendWelcomeEmail(event.email);
      break;
    case "order_placed":
      reserveInventory(event.items);
      break;
    case "payment_failed":
      if (event.retryable) {
        scheduleRetry(event.orderId, event.amount);
      }
      break;
    case "order_shipped":
      sendTrackingEmail(event.orderId, event.trackingNumber);
      break;
    default:
      // COMPILE ERROR: Argument of type 'OrderCancelled' is
      // not assignable to parameter of type 'never'.
      assertNever(event);
  }
}
```

How it works: after all `case` branches, the only remaining type should be `never` (the empty type — no values inhabit it). If a variant is unhandled, its type leaks into the `default` branch, and `never` assignment fails at compile time. See [Conditional Types & `infer`](conditional-types-and-infer.md) for more on how `never` functions as the bottom type.

### Using `satisfies never`

A lighter alternative without a helper function (TypeScript 4.9+):

```typescript
function handleEvent(event: AppEvent): void {
  switch (event.type) {
    case "user_created":
      // ...
      break;
    case "order_placed":
      // ...
      break;
    case "payment_failed":
      // ...
      break;
    case "order_shipped":
      // ...
      break;
    default: {
      const _exhaustive: never = event; // compile error if not exhaustive
      throw new Error(`Unhandled: ${JSON.stringify(_exhaustive)}`);
    }
  }
}
```

The runtime `throw` is a safety net for impossible paths. The real protection is the compile-time error. This is where the [Error Handling Architecture](../patterns/error-handling-architecture.md) Result type pattern uses exhaustive `switch` — every error variant must be handled.

---

## Type Predicates (`is`)

Sometimes TypeScript cannot infer the narrowing from your runtime check. Custom type guard functions bridge the gap.

```typescript
interface ApiSuccess<T> {
  status: "success";
  data: T;
}

interface ApiError {
  status: "error";
  message: string;
  code: number;
}

type ApiResponse<T> = ApiSuccess<T> | ApiError;

// Type predicate: the return type `x is ApiError` tells TS
// that if this function returns true, x is ApiError
function isApiError<T>(response: ApiResponse<T>): response is ApiError {
  return response.status === "error";
}

async function fetchUser(id: string): Promise<User> {
  const response = await callApi<User>(`/users/${id}`);

  if (isApiError(response)) {
    // TS knows: response is ApiError
    throw new Error(`API error ${response.code}: ${response.message}`);
  }

  // TS knows: response is ApiSuccess<User>
  return response.data;
}
```

### Validating Unknown Data

Type predicates shine when validating data from the outside world — API responses, parsed JSON, user input:

```typescript
interface UserPayload {
  id: string;
  email: string;
  role: "admin" | "user";
}

function isUserPayload(value: unknown): value is UserPayload {
  if (typeof value !== "object" || value === null) return false;
  const obj = value as Record<string, unknown>;
  return (
    typeof obj.id === "string" &&
    typeof obj.email === "string" &&
    (obj.role === "admin" || obj.role === "user")
  );
}

// Usage
const raw: unknown = JSON.parse(body);
if (isUserPayload(raw)) {
  // TS knows: raw is UserPayload
  console.log(raw.email);
}
```

### The Danger

**TypeScript trusts your predicate unconditionally.** If your runtime check is wrong, the types lie:

```typescript
// WRONG: this predicate lies
function isString(value: unknown): value is string {
  return typeof value === "number"; // bug: checks for number, claims string
}

const x: unknown = 42;
if (isString(x)) {
  // TS thinks x is string, but it is actually 42
  x.toUpperCase(); // runtime 💥
}
```

Type predicates are an escape hatch. Use them when the compiler cannot infer the narrowing, and **test them thoroughly**.

---

## Assertion Functions (`asserts`)

Type predicates narrow inside a branch. Assertion functions narrow **everything after the call**.

```typescript
function assertDefined<T>(
  value: T | null | undefined,
  name: string
): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(`Expected ${name} to be defined`);
  }
}

// Usage
function processOrder(userId: string | null, orderId: string | undefined) {
  assertDefined(userId, "userId");
  assertDefined(orderId, "orderId");

  // From here on, TS knows:
  // userId is string (not string | null)
  // orderId is string (not string | undefined)
  console.log(userId.toUpperCase()); // no error
  console.log(orderId.trim());       // no error
}
```

### When to Use Assertion Functions vs Type Predicates

| Scenario | Use |
|----------|-----|
| Checking inside an `if` branch | Type predicate (`is`) |
| Guarding at the top of a function | Assertion function (`asserts`) |
| Returning a boolean for conditional logic | Type predicate |
| Throwing on invalid state (fail-fast) | Assertion function |

Assertion functions pair well with the precondition pattern — validate inputs at the function boundary, then work with narrow types throughout the body.

```typescript
function assertInstanceOf<T>(
  value: unknown,
  ctor: new (...args: any[]) => T,
  message?: string
): asserts value is T {
  if (!(value instanceof ctor)) {
    throw new Error(message ?? `Expected instance of ${ctor.name}`);
  }
}
```

---

## `in` Operator Narrowing

When you do not have a discriminant field, the `in` operator checks for property existence:

```typescript
interface Fish {
  swim(): void;
  habitat: string;
}

interface Bird {
  fly(): void;
  wingspan: number;
}

function move(animal: Fish | Bird): void {
  if ("swim" in animal) {
    // TS knows: animal is Fish
    animal.swim();
  } else {
    // TS knows: animal is Bird
    animal.fly();
  }
}
```

### Limitations

- **Does not narrow the property type** — only confirms the property exists.
- **Breaks with overlapping properties** — if both `Fish` and `Bird` have `swim`, the `in` check does not help.
- **Discriminated unions are almost always better** — use `in` as a fallback when you cannot add a discriminant (e.g., third-party types).

```typescript
// This does NOT narrow well
interface Dog { bark(): void; swim(): void; }
interface Fish { swim(): void; }

function move(animal: Dog | Fish): void {
  if ("swim" in animal) {
    // TS knows: animal is Dog | Fish — no narrowing happened!
    // Both types have 'swim', so 'in' cannot distinguish them.
  }
}
```

---

## `satisfies` for Narrowing

The `satisfies` operator (TypeScript 4.9+) validates that a value conforms to a type **without widening it**. This preserves narrow literal types that assignment would erase.

```typescript
// Without satisfies — types are widened
const routes: Record<string, string> = {
  home: "/",
  about: "/about",
  contact: "/contact",
};
// typeof routes.home is string — literal "/" is lost

// With satisfies — literal types preserved
const routes = {
  home: "/",
  about: "/about",
  contact: "/contact",
} satisfies Record<string, string>;
// typeof routes.home is "/" — the literal is kept
// AND TypeScript verified it matches Record<string, string>
```

This matters for narrowing because you keep the compiler's ability to distinguish exact values while still checking against a broader contract. See [Structural Typing](structural-typing.md) for the assignability rules that `satisfies` checks against.

### Configuration Objects

```typescript
type LogLevel = "debug" | "info" | "warn" | "error";

interface LoggerConfig {
  level: LogLevel;
  prefix: string;
  silent: boolean;
}

const config = {
  level: "info",
  prefix: "[app]",
  silent: false,
} satisfies LoggerConfig;

// config.level is "info" (not LogLevel)
// config.silent is false (not boolean)
// You get both: type checking AND narrow inference
```

### With Discriminated Unions

```typescript
const defaultEvent = {
  type: "user_created",
  userId: "system",
  email: "system@app.com",
  createdAt: new Date(),
} satisfies AppEvent;

// defaultEvent.type is "user_created" (not string)
// TypeScript knows this is specifically a UserCreated event
```

---

## Real-World Patterns

### State Machine with Discriminated Unions

Model an order lifecycle where each state carries only the data relevant to that state:

```typescript
interface DraftOrder {
  status: "draft";
  items: ReadonlyArray<LineItem>;
}

interface PlacedOrder {
  status: "placed";
  items: ReadonlyArray<LineItem>;
  placedAt: Date;
  orderId: string;
}

interface PaidOrder {
  status: "paid";
  items: ReadonlyArray<LineItem>;
  placedAt: Date;
  orderId: string;
  paidAt: Date;
  paymentId: string;
}

interface ShippedOrder {
  status: "shipped";
  items: ReadonlyArray<LineItem>;
  orderId: string;
  paidAt: Date;
  shippedAt: Date;
  trackingNumber: string;
}

interface DeliveredOrder {
  status: "delivered";
  orderId: string;
  deliveredAt: Date;
  signature: string;
}

type Order = DraftOrder | PlacedOrder | PaidOrder | ShippedOrder | DeliveredOrder;

// State transitions are type-safe
function shipOrder(order: PaidOrder, tracking: string): ShippedOrder {
  return {
    status: "shipped",
    items: order.items,
    orderId: order.orderId,
    paidAt: order.paidAt,
    shippedAt: new Date(),
    trackingNumber: tracking,
  };
}

// You CANNOT ship a draft order — it is a compile error
// shipOrder(draftOrder, "TRACK123"); // Error: DraftOrder is not PaidOrder
```

This is the TypeScript equivalent of Java's sealed interface with records (Java 17+). The compiler enforces valid transitions.

### API Response Handling

A generic response wrapper used across your entire API layer:

```typescript
type ApiResponse<T> =
  | { status: "success"; data: T; meta?: { total: number; page: number } }
  | { status: "error"; message: string; code: string }
  | { status: "loading" };

function renderResponse<T>(
  response: ApiResponse<T>,
  renderData: (data: T) => string
): string {
  switch (response.status) {
    case "loading":
      return "Loading...";
    case "error":
      return `Error [${response.code}]: ${response.message}`;
    case "success":
      return renderData(response.data);
    default:
      const _exhaustive: never = response;
      throw new Error(`Unhandled: ${JSON.stringify(_exhaustive)}`);
  }
}
```

See [Error Handling Architecture](../patterns/error-handling-architecture.md) for the `Result<T, E>` type — the canonical example of discriminated unions applied to error handling across architectural layers.

### Form Validation States

```typescript
interface InvalidForm {
  state: "invalid";
  errors: ReadonlyMap<string, string>;
  values: Readonly<Record<string, string>>;
}

interface ValidForm {
  state: "valid";
  values: Readonly<Record<string, string>>;
}

interface SubmittedForm {
  state: "submitted";
  values: Readonly<Record<string, string>>;
  submittedAt: Date;
}

interface SubmissionFailed {
  state: "submission_failed";
  values: Readonly<Record<string, string>>;
  error: string;
}

type FormState = InvalidForm | ValidForm | SubmittedForm | SubmissionFailed;

function getSubmitButtonLabel(form: FormState): string {
  switch (form.state) {
    case "invalid":
      return `Fix ${form.errors.size} error(s)`;
    case "valid":
      return "Submit";
    case "submitted":
      return "Submitted ✓";
    case "submission_failed":
      return "Retry";
    default:
      const _: never = form;
      throw new Error(`Unhandled state: ${JSON.stringify(_)}`);
  }
}
```

### Combining Predicates with Discriminated Unions

When working with external data that might be any of your event types:

```typescript
function isAppEvent(value: unknown): value is AppEvent {
  if (typeof value !== "object" || value === null) return false;
  const obj = value as Record<string, unknown>;
  const validTypes = [
    "user_created", "order_placed", "payment_failed", "order_shipped"
  ] as const;
  return (
    typeof obj.type === "string" &&
    (validTypes as readonly string[]).includes(obj.type)
  );
}

// At the system boundary: validate, then narrow
function processIncomingMessage(raw: unknown): void {
  if (!isAppEvent(raw)) {
    console.error("Unknown event shape", raw);
    return;
  }
  // TS knows: raw is AppEvent — discriminated union narrowing works from here
  handleEvent(raw);
}
```

---

## Related

- [Structural Typing & Type Compatibility](structural-typing.md) — the assignability rules that underpin narrowing; why structure determines what narrows to what
- [Generics & Constraints in Practice](generics-and-constraints.md) — generic type guards, constrained utility types
- [Conditional Types & `infer`](conditional-types-and-infer.md) — `never` as the bottom type, distributive conditionals over unions
- [Error Handling Architecture](../patterns/error-handling-architecture.md) — `Result<T, E>` as discriminated union, exhaustive error handling
- [DDD Tactical Patterns](../patterns/ddd-tactical-patterns.md) — domain events modeled as discriminated unions
- [CQRS & Event Patterns](../patterns/cqrs-and-events.md) — command and event types built on discriminated unions

---

## References

- [TypeScript Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TypeScript Handbook — Discriminated Unions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions)
- [TypeScript Handbook — Type Predicates](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)
- [TypeScript Handbook — Assertion Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions)
- [TypeScript Handbook — The `satisfies` Operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)
- [TypeScript Handbook — The `never` Type](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-never-type)
- [Making Impossible States Impossible — Richard Feldman (Elm Conf)](https://www.youtube.com/watch?v=IcgmSRJHu_8) — the original talk that popularized this philosophy
