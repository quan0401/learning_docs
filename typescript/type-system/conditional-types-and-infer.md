---
title: "Conditional Types & `infer`"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, type-system, conditional-types, infer]
---

# Conditional Types & `infer`

| | |
|---|---|
| **Problem** | Extract the success type from `Result<T, E>` at the type level |
| **Prerequisites** | [Generics & Constraints](generics-and-constraints.md), [Structural Typing](structural-typing.md) |
| **TypeScript version** | 2.8+ (conditional types), 4.7+ (infer with extends constraints) |

---

## The Problem

Your codebase uses a `Result<T, E>` type for
[typed error handling](../patterns/error-handling-architecture.md). You have a
generic function that accepts any `Result` and needs to work only with the
success type. How do you extract `T` from `Result<T, E>` without knowing the
concrete types at the call site?

```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

// Goal: given Result<Order, OrderNotFoundError>, extract Order
type SuccessType<R> = /* ??? */;
```

The answer is conditional types combined with `infer`. But to get there safely,
you need to understand the entire conditional type system — including its most
confusing behavior: distribution over unions.

---

## 1. Conditional Type Syntax

The basic form mirrors a ternary, but operates entirely at the type level:

```typescript
type T extends U ? X : Y
```

This is **not** a runtime ternary. No JavaScript is emitted. The compiler
evaluates the `extends` check at type-checking time and selects one branch.

The `extends` here means [structural compatibility](structural-typing.md) — the
same `extends` you use in generic constraints, not the `extends` from class
inheritance.

### Simple examples

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;       // true
type B = IsString<number>;       // false
type C = IsString<'hello'>;      // true  — 'hello' extends string
type D = IsString<string | number>; // true | false  — wait, what?
```

That last result is surprising. Why does `IsString<string | number>` produce
`true | false` (which simplifies to `boolean`) instead of just `false`?
The answer is distribution.

---

## 2. Distributive Conditional Types

This is the single most confusing behavior in TypeScript's type system. It
trips up experienced developers regularly.

### The rule

When `T` in `T extends U ? X : Y` is a **naked type parameter** (not wrapped
in any other type constructor), and `T` is instantiated with a **union type**,
the conditional type **distributes** over each member of the union.

### Step-by-step evaluation

```typescript
type IsString<T> = T extends string ? true : false;

// Evaluate IsString<string | number>:
// Step 1: T = string | number (union)
// Step 2: T is a naked type parameter → distribution applies
// Step 3: distribute over each union member:
//         IsString<string> | IsString<number>
//       = (string extends string ? true : false) | (number extends string ? true : false)
//       = true | false
//       = boolean
```

### Another example with filtering

Distributive behavior is actually useful — it is the foundation of built-in
utility types like `Exclude` and `Extract`:

```typescript
type Exclude<T, U> = T extends U ? never : T;

type Result = Exclude<'a' | 'b' | 'c', 'a'>;
// distributes:
//   ('a' extends 'a' ? never : 'a') | ('b' extends 'a' ? never : 'b') | ('c' extends 'a' ? never : 'c')
// = never | 'b' | 'c'
// = 'b' | 'c'
```

The `never` type disappears from unions — it is the empty type, the identity
element of union. More on that in the next section.

### Preventing distribution

Sometimes you **do not want** distribution. Wrap `T` in a tuple:

```typescript
type IsStringStrict<T> = [T] extends [string] ? true : false;

type A = IsStringStrict<string>;              // true
type B = IsStringStrict<number>;              // false
type C = IsStringStrict<string | number>;     // false  — no distribution
```

The brackets `[T]` mean `T` is no longer a "naked" type parameter. The
compiler checks `[string | number] extends [string]`, which is `false`
because a tuple containing `string | number` is not assignable to a tuple
containing only `string`.

### Java comparison

Java has no equivalent. Java generics are erased at runtime and have no
type-level conditional logic. The closest concept is `instanceof` checks at
runtime, but TypeScript conditional types operate purely at compile time with
full type information.

---

## 3. The `never` Type as the Empty Union

`never` is the bottom type — it has zero inhabitants. No value can ever have
type `never`. But in the context of conditional types, the critical mental
model is: **`never` is a union with zero members**.

### Distribution over zero members

When a distributive conditional type receives `never`, it distributes over
zero union members, producing zero results — which is `never`:

```typescript
type IsString<T> = T extends string ? true : false;

type Oops = IsString<never>;
// T = never (empty union)
// Distribution: apply to each member of the union... there are none
// Result: never  (not true, not false)
```

This is deeply unintuitive. You might expect `never extends string` to be
`true` (since `never` is assignable to everything). But distribution happens
**before** the `extends` check — and there are no members to check.

### The `IsNever` trap

```typescript
// Broken — always returns never for never input
type IsNever<T> = T extends never ? true : false;
type Test = IsNever<never>; // never  ← not true!

// Fixed — prevent distribution with tuple wrapper
type IsNever<T> = [T] extends [never] ? true : false;
type Test = IsNever<never>; // true  ✓
```

### Practical consequences

This matters in real code. If you build a conditional type that might receive
`never` (e.g., from a `Record` key lookup that returns `never` for missing
keys), your conditional type silently returns `never` instead of the fallback
branch. The tuple wrapper is the fix.

---

## 4. The `infer` Keyword

`infer` declares a type variable **inside** the `extends` clause of a
conditional type. The compiler attempts to infer its value from the structure
of the input type.

### Basic pattern

```typescript
type Unwrap<T> = T extends Promise<infer U> ? U : T;

type A = Unwrap<Promise<string>>;  // string
type B = Unwrap<Promise<number>>;  // number
type C = Unwrap<number>;           // number (fallback branch)
```

`infer U` tells the compiler: "if `T` is structurally a `Promise<something>`,
capture that `something` as `U`."

### Building standard utility types from scratch

#### `ReturnType<T>`

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = ReturnType<() => string>;             // string
type B = ReturnType<(x: number) => boolean>;   // boolean
type C = ReturnType<string>;                   // never
```

#### `Parameters<T>`

```typescript
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

type A = Parameters<(x: string, y: number) => void>;  // [x: string, y: number]
```

#### `ConstructorParameters<T>`

```typescript
type ConstructorParameters<T> = T extends new (...args: infer P) => any ? P : never;

class UserService {
  constructor(private db: Database, private logger: Logger) {}
}

type Args = ConstructorParameters<typeof UserService>;  // [db: Database, logger: Logger]
```

### `infer` with `extends` constraints (TS 4.7+)

You can constrain what `infer` matches:

```typescript
type FirstString<T> = T extends [infer S extends string, ...any[]] ? S : never;

type A = FirstString<['hello', 42]>;   // 'hello'
type B = FirstString<[42, 'hello']>;   // never — first element is not a string
```

The `infer S extends string` means: infer `S`, but only if it extends
`string`. This narrows the inferred type and avoids post-inference narrowing.

---

## 5. Practical `infer` Patterns

### Extract function parameter types

```typescript
// First parameter
type FirstParam<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type A = FirstParam<(req: Request, res: Response) => void>;  // Request

// Last parameter (using rest + infer on the last position)
type LastParam<T> = T extends (...args: [...any[], infer L]) => any ? L : never;

type B = LastParam<(a: string, b: number, c: boolean) => void>;  // boolean

// Rest parameters
type RestParams<T> = T extends (first: any, ...rest: infer R) => any ? R : never;

type C = RestParams<(req: Request, ...middleware: Function[]) => void>;  // Function[]
```

### Extract from template literal types

Parse a route string to extract parameter names:

```typescript
type ExtractParam<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParam<Rest>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type RouteParams = ExtractParam<'/users/:userId/posts/:postId'>;
// 'userId' | 'postId'
```

This is how libraries like Express type their route parameters — the string
literal becomes a source of type information.

### Extract from arrays and tuples

```typescript
// First element
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;

// Last element
type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;

// Tail (everything except first)
type Tail<T extends any[]> = T extends [any, ...infer R] ? R : never;

type A = Head<[string, number, boolean]>;  // string
type B = Last<[string, number, boolean]>;  // boolean
type C = Tail<[string, number, boolean]>;  // [number, boolean]
```

### Extract from `Result<T, E>`

Back to the opening problem — extracting the success type from the
[Result type](../patterns/error-handling-architecture.md):

```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

type SuccessType<R> = R extends Result<infer T, any> ? T : never;
type ErrorType<R>   = R extends Result<any, infer E> ? E : never;

// Usage
type OrderResult = Result<Order, OrderNotFoundError>;
type S = SuccessType<OrderResult>;  // Order
type E = ErrorType<OrderResult>;    // OrderNotFoundError
```

You can also extract both at once using a tuple return:

```typescript
type Decompose<R> = R extends Result<infer T, infer E> ? [T, E] : never;

type Parts = Decompose<Result<Order, OrderNotFoundError>>;  // [Order, OrderNotFoundError]
```

---

## 6. Building Utility Types

The real power emerges when you combine conditional types, `infer`, mapped
types, and recursion.

### `Awaited<T>` — recursive Promise unwrapping

TypeScript ships this built-in since 4.5, but building it from scratch is
instructive:

```typescript
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any }
    ? F extends (value: infer V, ...args: infer _) => any
      ? Awaited<V>   // recursive: unwrap nested promises
      : never
    : T;

type A = Awaited<Promise<string>>;              // string
type B = Awaited<Promise<Promise<number>>>;     // number (recursively unwrapped)
type C = Awaited<string>;                        // string (not a promise, returned as-is)
```

The recursion handles `Promise<Promise<Promise<T>>>` — it keeps unwrapping
until it hits a non-thenable type.

### `FlattenArray<T>` — unwrap one level of array nesting

```typescript
type FlattenArray<T> = T extends Array<infer U> ? U : T;

type A = FlattenArray<string[]>;       // string
type B = FlattenArray<number[][]>;     // number[]  — one level only
type C = FlattenArray<string>;         // string    — not an array
```

### `EventPayload<T>` — extract payload from a handler signature

```typescript
type HandlerPayload<T> = T extends (event: infer E) => any ? E : never;

type Handler = (event: { userId: string; email: string }) => void;
type Payload = HandlerPayload<Handler>;  // { userId: string; email: string }
```

---

## 7. Chaining and Nesting

### Multiple conditions

Chain conditions like an `if-else if-else`:

```typescript
type TypeName<T> =
  T extends string  ? 'string' :
  T extends number  ? 'number' :
  T extends boolean ? 'boolean' :
  T extends null    ? 'null' :
  T extends undefined ? 'undefined' :
  'object';

type A = TypeName<string>;   // 'string'
type B = TypeName<42>;       // 'number'
type C = TypeName<Date>;     // 'object'
```

### When nesting gets unwieldy

Once you have three or more levels of nesting inside a single conditional type,
extract named type aliases:

```typescript
// Hard to read
type Process<T> =
  T extends Promise<infer U>
    ? U extends Array<infer V>
      ? V extends Result<infer S, any>
        ? S
        : V
      : U
    : T;

// Better: decompose into named steps
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type UnwrapArray<T>   = T extends Array<infer U> ? U : T;
type UnwrapResult<T>  = T extends Result<infer S, any> ? S : T;

type Process<T> = UnwrapResult<UnwrapArray<UnwrapPromise<T>>>;
```

Named aliases read like a pipeline: unwrap the promise, then the array, then
the result. Each step is independently testable and reusable.

---

## 8. Real-World Examples

### Type-safe event emitter payload extraction

```typescript
interface AppEvents {
  'user.login': { userId: string; timestamp: Date };
  'cache.miss': { key: string };
}

// Extract the payload type for any event key
type EventPayload<M, K extends keyof M> = M[K];

// Extract from a handler function that wraps the event map
type EmitterEvent<T> =
  T extends (event: keyof infer M, handler: (payload: infer P) => void) => void
    ? P
    : never;

type LoginPayload = EventPayload<AppEvents, 'user.login'>;
// { userId: string; timestamp: Date }
```

### API response type extraction

When your API client returns wrapped responses:

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  headers: Record<string, string>;
}

type UnwrapResponse<T> = T extends ApiResponse<infer D> ? D : never;

// Given a client interface
interface ApiClient {
  getUser(id: string): Promise<ApiResponse<User>>;
  listOrders(): Promise<ApiResponse<Order[]>>;
}

// Extract the data type for any method
type ApiData<T> = T extends (...args: any[]) => Promise<ApiResponse<infer D>> ? D : never;

type UserData = ApiData<ApiClient['getUser']>;     // User
type OrderData = ApiData<ApiClient['listOrders']>; // Order[]
```

### Route parameter inference

Type-safe route parameters extracted from string patterns, similar to how
frameworks like Express or Hono implement typed routing:

```typescript
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractRouteParams<Rest>]:
        K extends keyof ExtractRouteParams<Rest>
          ? ExtractRouteParams<Rest>[K]
          : string }
    : T extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = ExtractRouteParams<'/users/:userId/posts/:postId'>;
// { userId: string; postId: string }

// Usage in a router
function get<P extends string>(
  path: P,
  handler: (params: ExtractRouteParams<P>) => Response,
): void {
  // implementation
}

get('/users/:userId/posts/:postId', (params) => {
  // params.userId — string, fully typed
  // params.postId — string, fully typed
  // params.unknown — compile error
  return new Response(params.userId);
});
```

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Unexpected distribution | Union input produces union output | Wrap in `[T] extends [U]` |
| `never` disappears | Conditional returns `never` for `never` input | Use tuple wrapper `[T] extends [never]` |
| `infer` in wrong position | Compiler cannot infer — returns fallback branch | Ensure structural match is correct |
| Recursive type depth | `Type instantiation is excessively deep` | Add a base case or use tail-recursive form |
| `any` matches everything | `any extends T` is always `true` — defeats guards | Check for `any` first with `0 extends 1 & T` |

---

## Related

- [Generics & Constraints](generics-and-constraints.md) — the `extends`
  keyword in generic parameters is prerequisite knowledge for conditional types
- [Structural Typing](structural-typing.md) — `extends` in conditional types
  checks structural compatibility, not nominal inheritance
- [Error Handling Architecture](../patterns/error-handling-architecture.md) —
  defines the `Result<T, E>` type used throughout this document as the
  motivating problem
- Mapped Types & Key Remapping _(planned)_ — conditional types combine with
  mapped types for powerful type transformations
- Recursive & Advanced Types _(planned)_ — recursive conditional types for
  deep unwrapping patterns

---

## References

- [TypeScript Handbook — Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [TypeScript Handbook — Distributive Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types)
- [TypeScript Handbook — Inferring Within Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#inferring-within-conditional-types)
- [TypeScript 2.8 Release Notes — Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html)
- [TypeScript 4.7 Release Notes — `extends` Constraints on `infer`](https://devblogs.microsoft.com/typescript/announcing-typescript-4-7/#extends-constraints-on-infer-type-variables)
