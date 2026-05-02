---
title: "Interfaces — Contracts Without Implementation"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, interfaces, java, typescript]
---

# Interfaces — Contracts Without Implementation

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `interfaces` `java` `typescript`

## Summary

An interface declares *what* an object can do without saying *how* it does it. It is the primary tool for decoupling — callers depend on the interface, implementations vary independently. Different languages enforce interface conformance differently (nominal in Java, structural in TypeScript), and that distinction shapes how you design APIs.

## Table of Contents

- [Interface as contract](#interface-as-contract)
- [Default methods (Java 8+)](#default-methods-java-8)
- [Multiple interface inheritance](#multiple-interface-inheritance)
- [Interface segregation preview](#interface-segregation-preview)
- [Structural vs nominal typing](#structural-vs-nominal-typing)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## Interface as contract

An interface lists method signatures that any implementer must provide. Callers program against the interface; the runtime substitutes whatever concrete class is wired in.

```java
public interface PaymentGateway {
    ChargeResult charge(Money amount, Card card);
    RefundResult refund(String chargeId, Money amount);
}

public class StripeGateway implements PaymentGateway {
    public ChargeResult charge(Money amount, Card card) { /* ... */ }
    public RefundResult refund(String chargeId, Money amount) { /* ... */ }
}

public class CheckoutService {
    private final PaymentGateway gateway;
    public CheckoutService(PaymentGateway gateway) { this.gateway = gateway; }
    // CheckoutService never sees Stripe specifically — only the interface.
}
```

```typescript
interface PaymentGateway {
  charge(amount: Money, card: Card): Promise<ChargeResult>;
  refund(chargeId: string, amount: Money): Promise<RefundResult>;
}

class StripeGateway implements PaymentGateway {
  async charge(amount: Money, card: Card): Promise<ChargeResult> { /* ... */ }
  async refund(chargeId: string, amount: Money): Promise<RefundResult> { /* ... */ }
}
```

The interface is a *promise*: any object that fits this shape can be used here. Tests substitute a mock; production wires the real gateway; you can swap providers without touching `CheckoutService`.

## Default methods (Java 8+)

Originally Java interfaces could only declare abstract method signatures. Java 8 added `default` methods — a method body inside an interface, available to all implementers unless they override.

```java
public interface Animal {
    String name();

    default String greeting() {
        return "Hello, I am " + name();
    }
}
```

Why this exists: to evolve interfaces without breaking every implementer. Before Java 8, adding a method to `Collection` would have broken every third-party `Collection`. Default methods let `Stream`, `forEach`, and friends slide in without recompiling the world.

Rules and gotchas:

- Default methods can be overridden in any implementer.
- A class can implement two interfaces with the same default method only if it explicitly resolves the conflict (`Interface.super.method()`).
- Interfaces still cannot hold mutable instance state — only `static final` constants.
- Java 9 added `private` interface methods for sharing helpers between defaults.

```java
public interface A { default String hi() { return "from A"; } }
public interface B { default String hi() { return "from B"; } }

public class C implements A, B {
    public String hi() { return A.super.hi(); } // must resolve manually
}
```

TypeScript interfaces do not support default implementations — they are pure types erased at runtime. For shared behavior, use abstract classes or composition.

## Multiple interface inheritance

A class can implement many interfaces. This is OOP's safe answer to multiple inheritance: you compose multiple contracts without inheriting multiple state slots.

```java
public interface Readable  { String read(); }
public interface Writable  { void write(String s); }
public interface Closeable { void close(); }

public class FileChannel implements Readable, Writable, Closeable { /* ... */ }
```

```typescript
interface Readable  { read(): string; }
interface Writable  { write(s: string): void; }
interface Closeable { close(): void; }

class FileChannel implements Readable, Writable, Closeable { /* ... */ }
```

This is the diamond problem (covered in inheritance) sidestepped: interfaces have no state to conflict, and Java forces explicit resolution when default-method signatures clash.

A useful pattern is the *intersection type* — describe a parameter that needs to satisfy multiple roles at once.

```typescript
function copyAndClose(src: Readable & Closeable, dst: Writable & Closeable) {
  dst.write(src.read());
  src.close();
  dst.close();
}
```

## Interface segregation preview

Interface Segregation (the I in SOLID) is the principle that no client should be forced to depend on methods it doesn't use. Big "do everything" interfaces force every implementer to stub out methods that don't apply, and force every consumer to import behavior they don't care about.

```java
// Bad — one fat interface
interface Worker {
    void doWork();
    void eatLunch();          // robots don't eat
    void requestVacation();   // contractors don't take vacation
}

// Better — small role interfaces
interface Workable    { void doWork(); }
interface Feedable    { void eatLunch(); }
interface Vacationer  { void requestVacation(); }

class Robot       implements Workable {}
class Employee    implements Workable, Feedable, Vacationer {}
class Contractor  implements Workable, Feedable {}
```

Heuristic: if half your implementers throw `UnsupportedOperationException` for a method, the interface is too big. Split it.

The full SOLID treatment is in the SOLID directory; this is the one-paragraph mental seed.

## Structural vs nominal typing

Two systems for deciding "does this object satisfy the interface."

| System | Rule | Languages |
|---|---|---|
| Nominal | The type must be *declared* to implement it | Java, C#, Kotlin, Swift |
| Structural | If it has the right *shape*, it satisfies | TypeScript, Go (interfaces), OCaml |

Nominal example — Java refuses unless `implements` is on the class:

```java
public interface Greeter { String hello(); }

public class Person {
    public String hello() { return "hi"; }
}

Greeter g = new Person(); // ERROR — Person does not implement Greeter,
                          // even though it has the right method.
```

Structural example — TypeScript accepts any object with the right shape:

```typescript
interface Greeter { hello(): string; }

class Person {
  hello(): string { return "hi"; }
}

const g: Greeter = new Person(); // OK — Person structurally matches Greeter,
                                 // no `implements` needed.
```

Practical implications:

- **Nominal** typing makes intent explicit and catches "accidental conformance" (two classes that happen to share a method but mean different things by it).
- **Structural** typing is friendlier to mocks, plain objects, and FFI boundaries — anything that quacks works.
- Java, Kotlin, and Swift add nominal *opt-ins* via `implements`, sealed types, and protocol conformances respectively. C# is nominal but supports structural-ish patterns via duck typing in `foreach` and pattern matching.
- Go is structural at the interface level but nominal for named types — a hybrid worth knowing about.

For a cross-language API, the nominal vs structural difference often dictates how you model "this thing has trait X." In Java you write small interfaces and explicitly declare them. In TS you describe a shape and let inference do the rest.

## Common pitfalls

- **Leaking implementation details into the interface.** A `PaymentGateway` interface that has a `getStripeClient()` method is no longer an abstraction — it is a pass-through. Either drop the method or rename the interface to be Stripe-specific.
- **Over-interfacing.** Not every class needs a one-class-one-interface pair. Add an interface when you have (or imminently expect) multiple implementations or you need it for testing.
- **Interfaces with too many methods.** Apply ISP: split by role.
- **Default methods as a workaround for missing inheritance.** They are for *interface evolution*, not for sharing implementation. If you find yourself piling business logic into defaults, you probably want an abstract class or composition.
- **Forgetting that TS interfaces vanish at runtime.** `instanceof MyInterface` does not work. Check structurally (`"hello" in obj` or a discriminator field).

## Related

- [classes-and-objects.md](./classes-and-objects.md) — what an interface is contracting *against*
- [abstraction.md](./abstraction.md) — interfaces as the most pure form of abstraction
- [polymorphism.md](./polymorphism.md) — interfaces as the dispatch surface for subtype polymorphism
- [inheritance.md](./inheritance.md) — interfaces as the safe alternative to multiple inheritance
- [../../java/INDEX.md](../../java/INDEX.md) — Java interfaces, default methods, sealed types
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — structural typing, interface vs type alias
