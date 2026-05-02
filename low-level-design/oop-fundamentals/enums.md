---
title: "Enums — Bounded Sets and Type-Safe Constants"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, enums, java, typescript]
---

# Enums — Bounded Sets and Type-Safe Constants

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `enums` `java` `typescript`

## Summary

An enum models a closed, finite set of named values that the type system can reason about exhaustively. Real enums are not just "named integers" — in Java they are full-fledged classes that can carry behavior; in TypeScript and C# the story is more nuanced and worth knowing in detail.

## Table of Contents

- [Enum vs constant](#enum-vs-constant)
- [Behavior on enums](#behavior-on-enums)
- [Sealed enum hierarchies](#sealed-enum-hierarchies)
- [Java vs TypeScript vs C#](#java-vs-typescript-vs-c)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## Enum vs constant

A constant is a single named value. An enum is a *bounded set* — and that boundedness is what makes the difference.

```java
// Without enums — strings or ints
public static final String STATUS_PENDING  = "PENDING";
public static final String STATUS_APPROVED = "APPROVED";
public static final String STATUS_REJECTED = "REJECTED";

void handle(String status) {
    if ("PENDIG".equals(status)) { /* typo, compiles, fails at runtime */ }
}
```

```java
// With enums
public enum Status { PENDING, APPROVED, REJECTED }

void handle(Status s) {
    switch (s) {
        case PENDING  -> ...;
        case APPROVED -> ...;
        case REJECTED -> ...;
    }
}
```

What you gain:

1. **Type safety** — you cannot pass an arbitrary string.
2. **Exhaustiveness** — modern `switch` (Java pattern switch, TS `never` checks) tells you when a case is missing.
3. **Refactor safety** — IDE rename works.
4. **Autocomplete** — the IDE knows the full set.
5. **Identity equality is safe** — enum constants are singletons; `==` works correctly in Java.

## Behavior on enums

Java enums are classes. Each constant is a singleton instance, and the enum can declare fields, constructors, and methods — including per-constant overrides.

```java
public enum Operation {
    PLUS("+") {
        public int apply(int a, int b) { return a + b; }
    },
    MINUS("-") {
        public int apply(int a, int b) { return a - b; }
    },
    TIMES("*") {
        public int apply(int a, int b) { return a * b; }
    };

    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    public String symbol() { return symbol; }
    public abstract int apply(int a, int b);
}

int result = Operation.PLUS.apply(2, 3); // 5
```

This is the Strategy pattern collapsed into a single type. Each constant carries its own implementation, and the `apply` method dispatches polymorphically.

### TypeScript: enums don't carry behavior

TS enums are reverse-mapped numeric or string objects, not classes. To attach behavior, the idiomatic move is a `const` object plus a union type, or a discriminated union.

```typescript
// Idiomatic TS — const object + literal union
const Operation = {
  PLUS: "PLUS",
  MINUS: "MINUS",
  TIMES: "TIMES",
} as const;
type Operation = typeof Operation[keyof typeof Operation];

const apply: Record<Operation, (a: number, b: number) => number> = {
  PLUS:  (a, b) => a + b,
  MINUS: (a, b) => a - b,
  TIMES: (a, b) => a * b,
};

apply[Operation.PLUS](2, 3); // 5
```

This pattern gives you the bounded-set guarantee without the compatibility footguns of `enum`.

## Sealed enum hierarchies

Sometimes you want a closed set where each member can carry *different shaped data*. That is no longer an enum — it is a sealed type / sum type / discriminated union.

```java
// Java 17+ — sealed interface as a closed sum type
public sealed interface PaymentResult
        permits PaymentResult.Approved,
                PaymentResult.Declined,
                PaymentResult.PendingReview {

    record Approved(String authCode) implements PaymentResult {}
    record Declined(String reasonCode, String message) implements PaymentResult {}
    record PendingReview(String reviewId) implements PaymentResult {}
}

String describe(PaymentResult r) {
    return switch (r) {
        case PaymentResult.Approved a       -> "ok " + a.authCode();
        case PaymentResult.Declined d       -> "no: " + d.reasonCode();
        case PaymentResult.PendingReview p  -> "wait " + p.reviewId();
    };
}
```

```typescript
// TypeScript — discriminated union does the same job
type PaymentResult =
  | { kind: "approved"; authCode: string }
  | { kind: "declined"; reasonCode: string; message: string }
  | { kind: "pendingReview"; reviewId: string };

function describe(r: PaymentResult): string {
  switch (r.kind) {
    case "approved":      return `ok ${r.authCode}`;
    case "declined":      return `no: ${r.reasonCode}`;
    case "pendingReview": return `wait ${r.reviewId}`;
  }
}
```

Use a plain enum when each member is just a label. Use a sealed type / discriminated union when each member carries its own payload.

## Java vs TypeScript vs C#

| Aspect | Java | TypeScript | C# |
|---|---|---|---|
| Underlying form | Class extending `java.lang.Enum` | `enum`: object map; `const enum`: inlined; or `as const` + union | Named integral constants |
| Singleton constants | Yes | Yes for `enum`; not for unions | No — values are not unique objects |
| Per-constant behavior | Yes (method overrides) | No directly; use lookup map | No directly; use extension methods |
| Exhaustiveness check | `switch` over enum (warning); `sealed` switch (error) | `switch` + `never` assertion | `switch` expression with patterns |
| Reverse lookup | `Status.valueOf("PENDING")` | `enum` auto reverse-map (numeric only); manual otherwise | `Enum.Parse<T>` |
| Implements interfaces | Yes | No | No |
| Common gotchas | Adding a constant breaks any non-default switch | `enum` has runtime cost and odd cross-module behavior; prefer `as const` | Any int castable to enum, even invalid |

### TypeScript: `enum` vs `const enum` vs `as const`

```typescript
// 1. enum — emits a runtime object, supports reverse mapping for numeric enums.
//    Has historical issues with isolatedModules and tree-shaking.
enum Color { Red, Green, Blue }

// 2. const enum — inlined at compile time, zero runtime cost.
//    Forbidden under isolatedModules without flags. Avoid in libraries.
const enum Direction { Up, Down }

// 3. as const + union — modern recommended pattern. No special syntax, full inference.
const Size = { S: "S", M: "M", L: "L" } as const;
type Size = typeof Size[keyof typeof Size];
```

For new TypeScript code in 2026, the `as const` + union pattern is the safe default. Reach for `enum` only when integrating with code that already uses it.

## Common pitfalls

- **Persisting enum ordinals.** Storing `Status.ordinal()` (Java) or numeric enum values in a database means reordering or inserting a constant breaks history. Persist the *name* instead.
- **`switch` without exhaustiveness.** A switch over an enum that omits a case compiles silently in many languages. Use sealed switches (Java 21+), `default -> throw`, or a `never` assertion in TS.
- **Mutable state inside an enum.** Enums are singletons. A mutable field on an enum is shared global state. Almost always wrong.
- **Treating an enum as a closed set forever.** Enums change. Plan for new members: a `default` arm, a `Status.UNKNOWN`, or a versioned schema.
- **Using TS `enum` across module boundaries.** Two modules that both `enum Color { Red }` may produce subtly different runtime values under certain bundlers. Prefer `as const`.

## Related

- [classes-and-objects.md](./classes-and-objects.md) — enums are special classes
- [interfaces.md](./interfaces.md) — Java enums can implement interfaces; sealed types use them as the closed root
- [polymorphism.md](./polymorphism.md) — per-constant method overrides as a form of subtype polymorphism
- [../../java/INDEX.md](../../java/INDEX.md) — Java enums, records, sealed types
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — discriminated unions and `as const` patterns
