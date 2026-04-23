---
title: "Mapped Types & Key Remapping"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, type-system, mapped-types, template-literals]
---

# Mapped Types & Key Remapping

| | |
|---|---|
| **Problem** | Generate API route types from an object schema |
| **Audience** | Intermediate TS/Node developer with deep Java knowledge |
| **Prereqs** | [Generics & Constraints](generics-and-constraints.md), [Conditional Types & `infer`](conditional-types-and-infer.md) |
| **Bridges to** | [NestJS for Spring Boot Developers](../frameworks/nestjs-for-spring-devs.md), [Prisma Deep Dive](../data-access/prisma-deep-dive.md) |

---

## 1. `keyof` and Index Access Types — The Building Blocks

Before writing a single mapped type, you need two primitives: `keyof` and indexed access `T[K]`.

### `keyof T` Produces a Union of Keys

```typescript
interface User {
  id: string;
  name: string;
  age: number;
  isActive: boolean;
}

type UserKeys = keyof User;
// "id" | "name" | "age" | "isActive"
```

This is the type-level equivalent of `Object.keys()` — but it returns a **union of string literal types**, not a runtime array.

### `T[K]` — Indexed Access (Lookup) Types

```typescript
type UserName = User["name"];     // string
type UserId = User["id"];         // string
type UserAge = User["age"];       // number
```

You can pass a union to the index position:

```typescript
type StringFields = User["id" | "name"];  // string
type AllValues = User[keyof User];        // string | number | boolean
```

`T[keyof T]` gives you a union of all value types in `T`. This pattern appears constantly when you need "any value this object could hold."

### Java Comparison

Java has no equivalent. The closest is reflection: `field.getType()`. In TS, `keyof` and index access happen entirely at compile time — zero runtime cost, full type safety. This is one of the areas where TS's type system genuinely surpasses Java's. See [Generics & Constraints](generics-and-constraints.md) for how `keyof` interacts with generic type parameters.

---

## 2. Basic Mapped Types

A mapped type iterates over a union of keys and produces a new object type. The syntax:

```typescript
type MappedType<T> = {
  [K in keyof T]: T[K]
};
```

This is the **identity mapped type** — it produces an exact copy of `T`. Not useful on its own, but it is the skeleton for everything else.

### Adding Modifiers: `readonly` and `?`

```typescript
// Build Readonly<T> from scratch
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K]
};

// Build Partial<T> from scratch
type MyPartial<T> = {
  [K in keyof T]?: T[K]
};

// Combine both
type ReadonlyPartial<T> = {
  readonly [K in keyof T]?: T[K]
};
```

### Removing Modifiers: `-readonly` and `-?`

The `-` prefix strips a modifier. This is how `Required<T>` works:

```typescript
// Build Required<T> from scratch
type MyRequired<T> = {
  [K in keyof T]-?: T[K]
};

// Remove readonly
type Mutable<T> = {
  -readonly [K in keyof T]: T[K]
};
```

### Practical Example: Immutable Domain Model

When working with [DDD patterns](../patterns/ddd-tactical-patterns.md), value objects should be deeply immutable. A shallow `Readonly<T>` is a start:

```typescript
interface OrderLine {
  productId: string;
  quantity: number;
  unitPrice: number;
}

type ImmutableOrderLine = MyReadonly<OrderLine>;
// { readonly productId: string; readonly quantity: number; readonly unitPrice: number }
```

For deep immutability you need recursion (covered in section 8).

---

## 3. Key Remapping With `as`

TypeScript 4.1 added `as` clauses to mapped types. This is where mapped types go from "copy with modifications" to "transform the entire shape."

```typescript
type RemappedType<T> = {
  [K in keyof T as NewKeyExpression]: T[K]
};
```

### Filtering Keys: Map to `never` to Remove

When the `as` clause evaluates to `never`, that key is excluded:

```typescript
type OmitId<T> = {
  [K in keyof T as K extends "id" ? never : K]: T[K]
};

type UserWithoutId = OmitId<User>;
// { name: string; age: number; isActive: boolean }
```

This is an alternative to `Omit<T, "id">` — but with `as` you can express arbitrary filtering logic.

### Transforming Keys With Template Literals

Combine `as` with template literal types to rename every key:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<K & string>}`]: () => T[K]
};

type UserGetters = Getters<User>;
// {
//   getId: () => string;
//   getName: () => string;
//   getAge: () => number;
//   getIsActive: () => boolean;
// }
```

The `K & string` intersection is necessary because `keyof T` can include `symbol` and `number`, but template literals only work with `string`. The intersection narrows it.

### Generate Setter Methods Too

```typescript
type Setters<T> = {
  [K in keyof T as `set${Capitalize<K & string>}`]: (value: T[K]) => void
};

type UserSetters = Setters<User>;
// {
//   setId: (value: string) => void;
//   setName: (value: string) => void;
//   setAge: (value: number) => void;
//   setIsActive: (value: boolean) => void;
// }
```

If this reminds you of Java's getter/setter boilerplate — exactly. Lombok's `@Getter`/`@Setter` generates methods at compile time via annotation processing. TS mapped types generate *types* for those methods at the type level.

---

## 4. Template Literal Types

Template literal types let you construct string types using interpolation at the type level.

### Basic Syntax

```typescript
type Greeting = `Hello, ${string}`;
// Matches "Hello, world", "Hello, TypeScript", etc.

type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ApiPath = "/users" | "/orders";
type Endpoint = `${HttpMethod} ${ApiPath}`;
// "GET /users" | "GET /orders" | "POST /users" | "POST /orders" | ...
// 4 * 2 = 8 members — distributive over unions
```

When you interpolate a union, the result distributes over all combinations. This is the same distributive behavior you saw with [conditional types](conditional-types-and-infer.md).

### Intrinsic String Manipulation Types

TypeScript provides four built-in types that transform string literals:

```typescript
type A = Uppercase<"hello">;      // "HELLO"
type B = Lowercase<"HELLO">;      // "hello"
type C = Capitalize<"hello">;     // "Hello"
type D = Uncapitalize<"Hello">;   // "hello"
```

These are *intrinsic* — implemented inside the compiler, not expressible in userland TS.

### Generate Event Names From a Union

```typescript
type Action = "click" | "hover" | "focus" | "blur";

type EventName = `on${Capitalize<Action>}`;
// "onClick" | "onHover" | "onFocus" | "onBlur"

type EventHandler = {
  [E in EventName]: (event: Event) => void;
};
// {
//   onClick: (event: Event) => void;
//   onHover: (event: Event) => void;
//   onFocus: (event: Event) => void;
//   onBlur: (event: Event) => void;
// }
```

### HTTP Route Patterns

```typescript
type Resource = "users" | "orders" | "products";
type RoutePattern = `/${Resource}` | `/${Resource}/:id`;
// "/users" | "/orders" | "/products" | "/users/:id" | "/orders/:id" | "/products/:id"
```

### CSS Property Types

```typescript
type CSSUnit = "px" | "rem" | "em" | "vh" | "vw" | "%";
type CSSLength = `${number}${CSSUnit}`;
// Matches "16px", "1.5rem", "100vh", etc.

const width: CSSLength = "100px";   // OK
const bad: CSSLength = "100";       // Error — no unit
```

---

## 5. Filtering With Mapped Types

Combine mapped types with conditional types to keep or remove keys based on their **value type**, not their name. This goes beyond what `Pick` and `Omit` offer (which filter by key name).

### `PickByType<T, ValueType>`

Keep only the keys whose values extend a given type:

```typescript
type PickByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? K : never]: T[K]
};

type StringFieldsOfUser = PickByType<User, string>;
// { id: string; name: string }

type NumericFieldsOfUser = PickByType<User, number>;
// { age: number }

type BooleanFieldsOfUser = PickByType<User, boolean>;
// { isActive: boolean }
```

The `as` clause with a conditional does the filtering — when `T[K]` does not extend `ValueType`, the key maps to `never` and is dropped.

### `OmitByType<T, ValueType>`

The inverse — exclude keys whose values extend a given type:

```typescript
type OmitByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? never : K]: T[K]
};

type NonStringFields = OmitByType<User, string>;
// { age: number; isActive: boolean }
```

### Extract Only Serializable Fields

A real scenario — you have a model with methods, computed getters, and plain data. You want only the plain data for JSON serialization:

```typescript
interface Order {
  id: string;
  total: number;
  items: string[];
  createdAt: Date;
  validate(): boolean;
  toJSON(): string;
}

type DataOnly<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K]
};

type OrderData = DataOnly<Order>;
// { id: string; total: number; items: string[]; createdAt: Date }
```

Methods are excluded because their types extend `Function`. This pattern is useful when generating DTOs in frameworks like [NestJS](../frameworks/nestjs-for-spring-devs.md).

---

## 6. Practical Patterns

### 6.1 Generate API Route Map From Endpoint Definitions

The motivating problem from the title. You have a backend schema that defines endpoints — generate a typed route map automatically.

```typescript
interface Endpoints {
  "/users": { GET: User[]; POST: User };
  "/users/:id": { GET: User; PUT: User; DELETE: void };
  "/orders": { GET: Order[]; POST: Order };
}

// Generate a type-safe fetch wrapper
type ApiClient = {
  [Path in keyof Endpoints]: {
    [Method in keyof Endpoints[Path]]: (
      ...args: Endpoints[Path][Method] extends void ? [] : [body: Endpoints[Path][Method]]
    ) => Promise<Endpoints[Path][Method]>
  }
};

// Usage (conceptual — the implementation is runtime code):
// client["/users"].GET()          → Promise<User[]>
// client["/users"].POST(newUser)  → Promise<User>
// client["/users/:id"].DELETE()   → Promise<void>
```

The double-nested mapped type iterates over paths, then over methods within each path. This is the kind of type that Prisma generates internally — see [Prisma Deep Dive](../data-access/prisma-deep-dive.md) for how Prisma uses similar patterns to generate its client types from your schema.

### 6.2 Transform a Prisma Model Into a Create DTO

Prisma models include auto-generated fields (`id`, `createdAt`, `updatedAt`) that should not appear in create inputs:

```typescript
interface PrismaUser {
  id: string;
  email: string;
  name: string;
  role: "ADMIN" | "USER";
  createdAt: Date;
  updatedAt: Date;
}

type AutoFields = "id" | "createdAt" | "updatedAt";

type CreateDTO<T, Excluded extends keyof T = AutoFields & keyof T> = {
  [K in keyof T as K extends Excluded ? never : K]: T[K]
};

type CreateUserDTO = CreateDTO<PrismaUser>;
// { email: string; name: string; role: "ADMIN" | "USER" }
```

The `Excluded` default uses an intersection `AutoFields & keyof T` so that the type only tries to exclude keys that actually exist on `T`.

### 6.3 Type-Safe Event Emitter

Link event names to their payload types so `emit` and `on` are fully typed:

```typescript
interface EventMap {
  userCreated: { userId: string; email: string };
  orderPlaced: { orderId: string; total: number };
  paymentFailed: { orderId: string; reason: string };
}

type TypedEmitter<Events> = {
  on<E extends keyof Events>(event: E, handler: (payload: Events[E]) => void): void;
  emit<E extends keyof Events>(event: E, payload: Events[E]): void;
};

// Usage:
declare const emitter: TypedEmitter<EventMap>;

emitter.on("userCreated", (payload) => {
  // payload is { userId: string; email: string } — fully inferred
  console.log(payload.userId);
});

emitter.emit("orderPlaced", { orderId: "123", total: 99.99 }); // OK
emitter.emit("orderPlaced", { orderId: "123" }); // Error — missing 'total'
```

This pattern shows generics and mapped types working together. The generic `E` constrains the event name, and the indexed access `Events[E]` links it to the correct payload. Compare this with Java's approach where you would need a separate listener interface per event type, or use `Object` and cast.

### 6.4 Snake_case to camelCase at the Type Level

Convert database column names (snake_case) to JavaScript property names (camelCase):

```typescript
// Step 1: Convert a single snake_case segment
type CamelCase<S extends string> =
  S extends `${infer Head}_${infer Tail}`
    ? `${Head}${CamelCase<Capitalize<Tail>>}`
    : S;

type Test1 = CamelCase<"created_at">;      // "createdAt"
type Test2 = CamelCase<"user_first_name">;  // "userFirstName"

// Step 2: Apply to all keys of an object
type CamelCaseKeys<T> = {
  [K in keyof T as CamelCase<K & string>]: T[K]
};

interface DbRow {
  user_id: string;
  first_name: string;
  created_at: Date;
  is_active: boolean;
}

type JsModel = CamelCaseKeys<DbRow>;
// {
//   userId: string;
//   firstName: string;
//   createdAt: Date;
//   isActive: boolean;
// }
```

This uses `infer` from [Conditional Types & `infer`](conditional-types-and-infer.md) to destructure the string at each `_` boundary, capitalize the tail, and recurse. The recursion terminates when there are no more underscores.

---

## 7. How the Built-in Utility Types Work

Now that you understand mapped types, you can read the actual implementations of the built-in utilities. No magic — just the patterns from sections 2-5.

### `Partial<T>`

```typescript
type Partial<T> = {
  [K in keyof T]?: T[K]
};
```

Adds `?` to every key. That is it.

### `Required<T>`

```typescript
type Required<T> = {
  [K in keyof T]-?: T[K]
};
```

Removes `?` from every key with the `-?` modifier.

### `Readonly<T>`

```typescript
type Readonly<T> = {
  readonly [K in keyof T]: T[K]
};
```

Adds `readonly` to every key.

### `Pick<T, K>`

```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P]
};
```

Iterates over `K` (a subset of `keyof T`) instead of `keyof T`. Only the selected keys appear in the result.

### `Record<K, V>`

```typescript
type Record<K extends keyof any, V> = {
  [P in K]: V
};
```

`keyof any` is `string | number | symbol` — the set of all valid property key types. `Record` creates an object type with keys from `K` and values of type `V`.

### `Omit<T, K>` — Why It Uses `Exclude`

```typescript
type Omit<T, K extends keyof any> = {
  [P in Exclude<keyof T, K>]: T[P]
};
```

`Omit` does **not** use an `as` clause to filter. Instead, it first computes `Exclude<keyof T, K>` — the keys of `T` that are not in `K` — and then maps over that reduced union. This is because `Omit` predates the `as` clause (added in TS 4.1). Both approaches produce the same result.

Note that `Omit`'s constraint is `K extends keyof any`, not `K extends keyof T`. This means you can `Omit<User, "nonexistent">` without error — a deliberate design choice for ergonomics when composing types.

### `Exclude<T, U>` and `Extract<T, U>`

These are not mapped types — they are distributive conditional types. But they appear so often alongside mapped types that they belong here:

```typescript
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
```

When `T` is a union, the conditional distributes over each member. See [Conditional Types & `infer`](conditional-types-and-infer.md) for the distribution rules.

---

## 8. Performance Considerations

Mapped types are compile-time constructs — they have zero runtime cost. But they can slow the **compiler** significantly when used carelessly.

### Deeply Nested Mapped Types

Each layer of nesting multiplies the work the compiler must do:

```typescript
// This is fine — one level
type Readonly1<T> = { readonly [K in keyof T]: T[K] };

// This gets expensive for large T
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K]
};
```

`DeepReadonly` recurses into every nested object. For a type with 5 levels of nesting and 10 properties per level, the compiler evaluates 10^5 = 100,000 type nodes. Keep recursive mapped types shallow, or apply them only where needed.

### `type` vs `interface` — Caching Behavior

```typescript
// Interface: cached by name, evaluated once
interface UserDTO {
  id: string;
  name: string;
}

// Type alias: potentially re-evaluated at each use site
type UserDTO2 = Pick<User, "id" | "name">;
```

Interfaces are **cached by name** in the compiler's type checker. Type aliases that involve mapped types or conditionals may be re-evaluated at each use site. For hot types that appear in many signatures, consider resolving to an interface:

```typescript
// Instead of this everywhere:
type CreateInput = Omit<PrismaUser, "id" | "createdAt" | "updatedAt">;

// Resolve once:
interface CreateUserInput {
  email: string;
  name: string;
  role: "ADMIN" | "USER";
}
```

This trades flexibility for compiler performance. Use the mapped type during development, then resolve to an explicit interface if the type-checker slows down.

### Avoiding Combinatorial Explosion

Template literal types with multiple union interpolations can explode:

```typescript
type Size = "sm" | "md" | "lg" | "xl";
type Color = "red" | "blue" | "green" | "yellow" | "purple";
type Variant = "solid" | "outline" | "ghost";

// 4 * 5 * 3 = 60 members — fine
type ClassName = `${Size}-${Color}-${Variant}`;

// But add more unions and it grows fast
// 10 * 10 * 10 * 10 = 10,000 members — compiler will protest
```

The compiler caps template literal unions at **100,000 members** and errors beyond that. In practice, you should stay well under this limit.

### When Mapped Types Are the Wrong Tool

If your type transformation is simple enough to express with `Pick`, `Omit`, or a manual interface, prefer the simpler form. Mapped types shine when:

- You need to transform **all** keys uniformly
- The transformation depends on the key name or value type
- You are building a reusable utility type
- The source type is generic (you do not know the keys ahead of time)

---

## Related

- [Structural Typing & Type Compatibility](structural-typing.md) — mapped types produce structurally typed objects; the assignability rules from structural typing apply to their outputs
- [Generics & Constraints](generics-and-constraints.md) — `keyof`, index access types, and generic constraints are prerequisites for mapped types
- [Conditional Types & `infer`](conditional-types-and-infer.md) — conditional types combine with mapped types for value-based filtering and string destructuring
- [NestJS for Spring Boot Developers](../frameworks/nestjs-for-spring-devs.md) — NestJS DTOs benefit from mapped types (`PartialType`, `OmitType` use them internally)
- [Prisma Deep Dive](../data-access/prisma-deep-dive.md) — Prisma's generated client types use mapped type patterns to produce model-specific CRUD types

---

## References

- [TypeScript Handbook — Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript Handbook — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [TypeScript Handbook — `keyof` Type Operator](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html)
- [TypeScript Handbook — Indexed Access Types](https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html)
- [TypeScript 4.1 Release Notes — Key Remapping](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html#key-remapping-via-as)
- [TypeScript 4.1 Release Notes — Template Literal Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html#template-literal-types)
