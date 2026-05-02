---
title: "Type-Driven Design — Making Illegal States Unrepresentable"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, type-driven-design, sealed-types, branded-types, parse-dont-validate]
---

# Type-Driven Design — Making Illegal States Unrepresentable

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `type-driven-design` `sealed-types` `branded-types` `parse-dont-validate`

## Summary

Type-Driven Design is the practice of pushing as many invariants as possible into the type system so the compiler — not your tests, not runtime checks, not code review — rejects programs that would violate them. The slogan, popularised by Yaron Minsky in his "Effective ML" talk and writings at Jane Street, is **"make illegal states unrepresentable."** If the type of `Order` cannot describe a shipped order without a tracking number, then no execution path can produce one.

Originally an OCaml/F# culture, the idea now travels naturally to TypeScript (discriminated unions, branded types), Kotlin (sealed classes, value classes), and increasingly Java (sealed types since 17, records, pattern matching). It pairs naturally with Alexis King's essay **"Parse, don't validate"** — once data crosses the boundary, it should arrive as a more precise type than it left, never as a raw `string` you then check repeatedly downstream.

This doc walks the four big tools — sealed sums, branded/phantom types, smart constructors, and refined collection types — with Java, Kotlin, and TypeScript examples, and ends with the trade-offs that make this style cost something.

## Table of Contents

- [Why illegal states matter](#why-illegal-states-matter)
- [Sealed types and exhaustive matching](#sealed-types-and-exhaustive-matching)
- [Phantom and branded types](#phantom-and-branded-types)
- [Smart constructors](#smart-constructors)
- [Parse, don't validate](#parse-dont-validate)
- [Refined collection types](#refined-collection-types)
- [Trade-offs](#trade-offs)
- [Related](#related)
- [References](#references)

## Why illegal states matter

A common shape in business code:

```java
class Order {
  String status;          // "PENDING" | "SHIPPED" | "CANCELLED"
  String trackingNumber;  // null unless SHIPPED
  Instant cancelledAt;    // null unless CANCELLED
  String cancelReason;    // null unless CANCELLED
}
```

Every read site has to remember the implicit rule: tracking only valid when status is SHIPPED, reason only valid when status is CANCELLED. Nothing in the type forbids a SHIPPED order with a cancel reason; only convention does. Bugs hide in the gap between "can be expressed" and "is allowed."

Type-driven design closes that gap by giving each state its own shape, so the wrong combinations cannot be written down in the first place.

## Sealed types and exhaustive matching

A sealed sum says: "this value is exactly one of these alternatives, and the compiler will tell you when you forget one."

### Java 17+ sealed interfaces with records

```java
public sealed interface Order
    permits Order.Pending, Order.Shipped, Order.Cancelled {

  record Pending(String id, List<LineItem> items) implements Order {}

  record Shipped(String id,
                 List<LineItem> items,
                 String trackingNumber,
                 Instant shippedAt) implements Order {}

  record Cancelled(String id,
                   List<LineItem> items,
                   Instant cancelledAt,
                   String reason) implements Order {}
}

// Exhaustive switch — compiler complains if a case is missing.
String describe(Order order) {
  return switch (order) {
    case Order.Pending p   -> "Pending with " + p.items().size() + " items";
    case Order.Shipped s   -> "Shipped via " + s.trackingNumber();
    case Order.Cancelled c -> "Cancelled: " + c.reason();
  };
}
```

A `Shipped` simply has no `cancelReason` field — there is nowhere to put bad data.

### Kotlin sealed classes

```kotlin
sealed interface Order {
  val id: String
  data class Pending(override val id: String, val items: List<LineItem>) : Order
  data class Shipped(
    override val id: String,
    val items: List<LineItem>,
    val trackingNumber: String,
    val shippedAt: Instant,
  ) : Order
  data class Cancelled(
    override val id: String,
    val items: List<LineItem>,
    val cancelledAt: Instant,
    val reason: String,
  ) : Order
}

fun describe(o: Order) = when (o) {                 // exhaustive
  is Order.Pending   -> "Pending"
  is Order.Shipped   -> "Shipped via ${o.trackingNumber}"
  is Order.Cancelled -> "Cancelled: ${o.reason}"
}
```

### TypeScript discriminated unions

```ts
type Order =
  | { kind: "pending";   id: string; items: LineItem[] }
  | { kind: "shipped";   id: string; items: LineItem[]; trackingNumber: string; shippedAt: Date }
  | { kind: "cancelled"; id: string; items: LineItem[]; cancelledAt: Date; reason: string };

function describe(o: Order): string {
  switch (o.kind) {
    case "pending":   return `Pending`;
    case "shipped":   return `Shipped via ${o.trackingNumber}`;
    case "cancelled": return `Cancelled: ${o.reason}`;
  }
  // With `noUncheckedIndexedAccess` and `--strict`, an `assertNever(o)` here
  // gives a compile error if a new variant is added without handling it.
}

function assertNever(x: never): never { throw new Error(`unhandled: ${JSON.stringify(x)}`); }
```

The `kind` discriminator unlocks narrowing inside each branch — TS knows `o.trackingNumber` exists only in the `shipped` arm.

## Phantom and branded types

Two `string` values can hold radically different meanings — a `UserId` and an `OrderId` look identical at runtime, but mixing them is a real and recurring bug. Phantom types attach a compile-time "tag" without changing runtime representation.

### TypeScript branded types

```ts
type Brand<T, B> = T & { readonly __brand: B };

export type UserId  = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;
export type Email   = Brand<string, "Email">;

// Constructors are the *only* way to mint a branded value.
export function userId(s: string): UserId {
  if (!/^u_[a-z0-9]{16}$/.test(s)) throw new Error(`invalid UserId: ${s}`);
  return s as UserId;
}

function getOrders(u: UserId): Promise<Order[]> { /* ... */ }

const raw = "u_abc...";
// getOrders(raw);          // error: string is not assignable to UserId
getOrders(userId(raw));     // ok
```

At runtime there is no overhead — `UserId` is still a string. The brand exists only in the type checker.

### Kotlin value classes

```kotlin
@JvmInline value class UserId(val value: String) {
  init { require(value.startsWith("u_")) { "invalid UserId" } }
}
@JvmInline value class OrderId(val value: String)

fun getOrders(u: UserId): List<Order> = TODO()
// getOrders(OrderId("o_1"))   // compile error
```

`value class` (or "inline class") avoids allocation in most cases while keeping the type distinct.

### Java: tiny wrapper records

Java has no native phantom type, but a record gives you the same effect at the cost of a wrapper:

```java
public record UserId(String value) {
  public UserId {
    if (value == null || !value.startsWith("u_"))
      throw new IllegalArgumentException("invalid UserId");
  }
}
public record OrderId(String value) {}
```

Use `UserId` everywhere the domain says "user id." `getOrders(OrderId)` is now a compile error.

## Smart constructors

A *smart constructor* is the single, validated entry point for a type. The constructor is private (or the type lives in a module that hides it), and a public factory does the parsing/validation. After it returns, holders of the type **know** the invariant holds.

```java
public final class Email {
  private static final Pattern PATTERN = Pattern.compile("[^@\\s]+@[^@\\s]+\\.[^@\\s]+");
  private final String value;

  private Email(String v) { this.value = v; }

  public static Email parse(String raw) {
    Objects.requireNonNull(raw);
    String trimmed = raw.trim().toLowerCase(Locale.ROOT);
    if (!PATTERN.matcher(trimmed).matches())
      throw new IllegalArgumentException("not an email: " + raw);
    return new Email(trimmed);
  }

  public String value() { return value; }
}
```

Once a function takes `Email`, it is freed from re-checking. The check happens once, at the boundary. For more complex types, a Builder (see [../design-patterns/creational/builder.md](../design-patterns/creational/builder.md)) plus a single `build()` validator is the same pattern at scale.

## Parse, don't validate

Alexis King's essay distils the rule: a *validator* takes input and returns a boolean — leaving you holding the same weakly-typed value. A *parser* takes input and returns a strictly-typed value, or fails. Always prefer parsers.

Validation style — the boolean lies to you:

```ts
function isValidEmail(s: string): boolean { /* ... */ }
function send(to: string) { if (!isValidEmail(to)) throw new Error(); /* ... */ }
```

`send` still takes `string` — every caller must remember to validate, and downstream functions cannot tell whether the validation already happened.

Parser style — the type is the proof:

```ts
type Email = Brand<string, "Email">;
function parseEmail(s: string): Email { /* validate or throw */ ... }
function send(to: Email) { /* trust */ }

send(parseEmail(req.body.to));
```

A `NonEmptyList` is another common case. Instead of every consumer rechecking `list.length > 0`, a parser turns `T[]` into `NonEmptyList<T>` once.

```ts
type NonEmptyList<T> = readonly [T, ...T[]];

function nonEmpty<T>(xs: readonly T[]): NonEmptyList<T> {
  if (xs.length === 0) throw new Error("expected non-empty");
  return xs as NonEmptyList<T>;
}

function head<T>(xs: NonEmptyList<T>): T { return xs[0]; }   // total — never throws
```

`head` cannot fail, by construction.

## Refined collection types

The same idea scales:

- `NonEmptyString` — a `String` that is guaranteed non-blank.
- `Positive`, `Percentage`, `Bps` — numbers with bounded ranges.
- `Path` vs `AbsolutePath` vs `NormalisedPath`.

Java sketch:

```java
public record NonBlank(String value) {
  public NonBlank {
    Objects.requireNonNull(value);
    if (value.isBlank())
      throw new IllegalArgumentException("must not be blank");
  }
}

public record Percentage(double value) {
  public Percentage {
    if (value < 0 || value > 100)
      throw new IllegalArgumentException("percentage out of range: " + value);
  }
}
```

The repository, controller, and service all take `Percentage`, not `double`. The validation is in one place; bugs from passing `0.95` when you meant `95` get caught early.

## Trade-offs

Type-driven design is not free. Honest costs:

- **API ergonomics.** Wrapping primitives means callers cannot pass a string literal directly — they must go through a constructor. Mostly fine, occasionally annoying for tests and scripts. Mitigations: factory helpers, test fixtures, `unsafeCast` only inside test code.
- **Refactoring cost.** Adding a new `Order` variant in a sealed hierarchy lights up every exhaustive switch. That is the *point* — the compiler shows you everywhere that needs updating — but it is real work. Plan migrations as multi-step changes (default branch first, then narrow).
- **Serialization friction.** Discriminated unions and sealed records often need a `@JsonTypeInfo`/Jackson `@JsonSubTypes` config, or a custom `kotlinx.serialization` polymorphic module, or a Zod discriminated union schema. Spend the time once; encode the discriminator explicitly.
- **Learning curve.** Teams used to anaemic data classes and validators-everywhere need a quick worked example to internalise the style. Lead with one painful bug that this would have caught — that is more persuasive than abstract argument.
- **Over-modelling.** Not every string deserves a wrapper. The bar: does mixing this value with another same-shaped value cause real bugs, or is the constraint important enough that downstream code repeatedly checks it? If yes, brand it. If not, leave it as `string`.
- **Performance.** Branded TS types cost zero. Java records and Kotlin value classes cost a small wrapper allocation in most paths. Negligible for business code; measure before worrying.

The mental shift: stop thinking of types as documentation and start thinking of them as *proofs*. Each function signature is a small theorem — "given these typed inputs, I produce this typed output, and no execution can violate that." The more invariants you can encode, the smaller your test surface and the louder the compiler when something is wrong.

## Related

- [./ddd-tactical-patterns.md](./ddd-tactical-patterns.md) — Value Objects are the DDD framing of the same idea.
- [./equality-and-identity.md](./equality-and-identity.md) — branded/value types interact with equality semantics.
- [./defensive-copying-and-immutability.md](./defensive-copying-and-immutability.md) — immutability is the natural partner to type-driven design.
- [./kiss-principle.md](./kiss-principle.md) — push complexity into types so call sites stay simple.
- [./separation-of-concerns.md](./separation-of-concerns.md) — parsing-at-the-boundary is a separation-of-concerns move.
- [../oop-fundamentals/encapsulation.md](../oop-fundamentals/encapsulation.md) — private constructors + smart factories are encapsulation in service of invariants.
- [../oop-fundamentals/abstraction.md](../oop-fundamentals/abstraction.md)
- [../solid/single-responsibility-principle.md](../solid/single-responsibility-principle.md) — one type, one invariant.
- [../solid/liskov-substitution-principle.md](../solid/liskov-substitution-principle.md) — sealed hierarchies make LSP enforceable.
- [../design-patterns/creational/builder.md](../design-patterns/creational/builder.md) — Builders are smart constructors for complex aggregates.
- [../design-patterns/creational/factory-method.md](../design-patterns/creational/factory-method.md)

## References

- Yaron Minsky, "Effective ML" (talk and Jane Street tech blog) — origin of "make illegal states unrepresentable."
- Alexis King, "Parse, don't validate" — essay on the parse vs validate distinction.
- Joshua Bloch, *Effective Java* (3rd ed.), items on minimising mutability and using static factories.
- Scott Wlaschin, *Domain Modeling Made Functional* — book-length treatment in F#.
- JEP 409 (Sealed Classes), JEP 395 (Records), JEP 441 (Pattern Matching for switch) — Java language specs.
- Kotlin language reference — sealed classes, value classes.
- TypeScript handbook — discriminated unions, narrowing.
