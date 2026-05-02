---
title: "Equality and Identity Deep Dive"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, equality, hashcode, value-objects, records]
---

# Equality and Identity Deep Dive

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `equality` `hashcode` `value-objects` `records`

## Summary

"Are these two things the same?" is one of the most overloaded questions in programming. Three different concepts hide behind it: **identity** (do these two references point at the same object in memory?), **equality** (do these two values represent the same thing by domain rules?), and **equivalence** (does some custom relation hold between them — case-insensitive, locale-aware, tolerance-based?).

Languages collapse them differently. Java has `==` for identity and `.equals()` for equality, with a strict contract that ties `equals` to `hashCode`. TypeScript has `===`, `==`, and `Object.is` — all reference-based for objects — and structural typing only at the *type* level, never at runtime. DDD models domain equality explicitly via Value Objects vs Entities.

This doc walks the contracts, the bugs that come from breaking them (mutable hash keys, fields-then-mutate, asymmetric equals across class hierarchies), and the modern solutions: Java records, Kotlin data classes, TypeScript deep-equality libraries.

## Table of Contents

- [Three concepts](#three-concepts)
- [Java equals/hashCode contract](#java-equalshashcode-contract)
- [Records and data classes](#records-and-data-classes)
- [The mutable-key disaster](#the-mutable-key-disaster)
- [Inheritance and equality](#inheritance-and-equality)
- [TypeScript equality](#typescript-equality)
- [DDD: Entities vs Value Objects](#ddd-entities-vs-value-objects)
- [Lombok pitfalls](#lombok-pitfalls)
- [Performance: caching hashCode](#performance-caching-hashcode)
- [Related](#related)
- [References](#references)

## Three concepts

| Concept     | Meaning                                                | Java                  | TS / JS                  |
| ----------- | ------------------------------------------------------ | --------------------- | ------------------------ |
| Identity    | Same memory cell                                       | `==`                  | `===` for objects, `Object.is` |
| Equality    | Same logical value                                     | `.equals()`           | manual / library (`fast-deep-equal`, `lodash.isEqual`) |
| Equivalence | Custom relation (case-insensitive, tolerance, ordering) | `Comparator`, custom  | custom comparator        |

A `User` with id `42` loaded twice from a database produces two distinct objects in memory (identity differs), but they are equal as domain values. Conversely, two `Money` instances with `100 USD` should be equal even though they are distinct objects.

`Object.is` in JS is mostly `===` with two corrections: `Object.is(NaN, NaN)` is `true`, and `Object.is(+0, -0)` is `false`. None of this changes the fact that for objects, both check identity.

## Java equals/hashCode contract

`Object.equals` and `Object.hashCode` form a contract that the standard library *relies on*. Break it and `HashMap`, `HashSet`, `equals`-based stream operations, and JPA caches misbehave in subtle ways.

The five rules from `Object`'s Javadoc:

1. **Reflexive** — `x.equals(x)` is `true`.
2. **Symmetric** — `x.equals(y)` iff `y.equals(x)`.
3. **Transitive** — if `x.equals(y)` and `y.equals(z)`, then `x.equals(z)`.
4. **Consistent** — repeated calls return the same result while the inputs are unchanged.
5. **Null-safe** — `x.equals(null)` is `false`, never throws.

And the hash invariant:

> If `a.equals(b)`, then `a.hashCode() == b.hashCode()`.

The reverse is *not* required: distinct objects may share a hash. But equal objects **must** hash the same, or `HashMap` lookups will return `null` for keys that are present.

A correct implementation:

```java
public final class Money {
  private final long minorUnits;
  private final Currency currency;

  public Money(long minorUnits, Currency currency) {
    this.minorUnits = minorUnits;
    this.currency = Objects.requireNonNull(currency);
  }

  @Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Money other)) return false;
    return minorUnits == other.minorUnits && currency.equals(other.currency);
  }

  @Override public int hashCode() {
    return Objects.hash(minorUnits, currency);
  }
}
```

`Objects.hash(...)` and `Objects.equals(a, b)` (null-safe) are the canonical helpers.

## Records and data classes

Java 14+ records auto-generate `equals`, `hashCode`, and `toString` based on the components. Most value-object boilerplate disappears:

```java
public record Money(long minorUnits, Currency currency) {
  public Money {
    Objects.requireNonNull(currency);
    if (minorUnits < 0) throw new IllegalArgumentException("negative");
  }
}
```

Two `Money` records with equal components are equal, and their hashes match. Done.

Kotlin's `data class` does the same:

```kotlin
data class Money(val minorUnits: Long, val currency: Currency)
```

Both also generate component-wise `toString`, which is good for logs but a *risk* when fields contain sensitive data — override `toString` if a record holds tokens or PII.

A subtle catch: records and data classes use **all** declared components. If you add a non-essential field (e.g., a cached display string), it will incorrectly participate in equality. The fix is usually to compute that field on demand instead.

## The mutable-key disaster

This is the most common equality bug in production:

```java
Set<Person> seen = new HashSet<>();
Person p = new Person("Ada", 36);
seen.add(p);
p.setAge(37);              // mutate AFTER insertion
seen.contains(p);          // false! the bucket index changed.
```

`HashSet` placed `p` in a bucket based on its old hash. Mutating a field that participates in `hashCode` changes the bucket — but the existing entry stays where it was. Now the set "contains" `p` (it's the same reference) but `contains(p)` returns false.

Rule: **objects used as keys in hash-based collections must be effectively immutable for the duration they are in the collection.** The safest path is to make value-like classes immutable end-to-end (records, `final` fields, no setters). If you must keep mutability, do not put those objects in `HashSet`/`HashMap`/`ConcurrentHashMap` keys.

## Inheritance and equality

Joshua Bloch (Effective Java, item on equals) shows why mixing concrete subclasses and value-equality across a class boundary is unsafe.

```java
class Point { int x, y;
  public boolean equals(Object o) {
    if (!(o instanceof Point p)) return false;
    return p.x == x && p.y == y;
  }
}

class ColorPoint extends Point { Color color;
  public boolean equals(Object o) {
    if (!(o instanceof ColorPoint cp)) return false;
    return super.equals(o) && cp.color == color;
  }
}
```

Now: `new Point(1,2).equals(new ColorPoint(1,2,RED))` is `true` but `new ColorPoint(1,2,RED).equals(new Point(1,2))` is `false` — symmetry broken.

Two ways out:

- Make `Point` final (sealed hierarchy) and let `ColorPoint` *contain* a `Point` rather than extend it.
- Use `getClass() == o.getClass()` instead of `instanceof` — pedantic but symmetric. Cost: prevents legitimate subclass equality.

Records cannot be subclassed (other than implementing interfaces), which is by design — they sidestep this problem entirely.

## TypeScript equality

TypeScript's type system is structural — `{ a: number }` and `{ a: number }` from different declarations are interchangeable. **At runtime**, none of that matters: `===` on objects is identity.

```ts
const a = { id: 1 };
const b = { id: 1 };
a === b;          // false
Object.is(a, b);  // false
JSON.stringify(a) === JSON.stringify(b); // brittle: depends on key order, fails for Date, BigInt, etc.
```

For deep equality use a library (`fast-deep-equal`, `lodash.isEqual`, or `node:util.isDeepStrictEqual`). For value-like data, the idiomatic approach is to make the data immutable (`readonly`, `Object.freeze`) and treat reference equality as adequate, or memoise constructors.

`==` (loose) does coercions that almost no one wants in 2026 production code. Treat the rule as: **always use `===`, except `value == null` to mean "null or undefined."**

For Map/Set keys, JavaScript hashes by reference. To get value-keyed lookups, build a serialised key:

```ts
const cache = new Map<string, Result>();
function key(req: Request): string { return `${req.userId}:${req.path}`; }
cache.get(key(req));
```

Records and Tuples (TC39 stage proposal) would give value-equal compound keys natively, but as of 2026 they are not yet shipped.

## DDD: Entities vs Value Objects

Domain-Driven Design (see [./ddd-tactical-patterns.md](./ddd-tactical-patterns.md)) makes the distinction explicit:

- **Entities** have identity. A `Customer` is the same customer across time even as their email and address change. Equality is by id.
- **Value Objects** are defined by their attributes. A `Money(100, USD)` is interchangeable with any other `Money(100, USD)`. Equality is by component, and they should be immutable.

In Java, model entities as classes with id-based `equals`, and value objects as records:

```java
public class Customer {                      // entity
  private final UUID id;
  private String email;                      // mutable attribute
  @Override public boolean equals(Object o) {
    return o instanceof Customer c && c.id.equals(id);
  }
  @Override public int hashCode() { return id.hashCode(); }
}

public record Money(long minorUnits, Currency currency) {}  // value object
```

JPA muddies this: managed entities have generated ids that may be `null` until persisted. The standard pattern is to use a domain-natural id when possible, or to compare by reference for transient entities and by id once persisted — and document the choice.

## Lombok pitfalls

`@EqualsAndHashCode` is convenient but has sharp edges:

- **Default ignores superclass fields.** With inheritance, you almost always need `@EqualsAndHashCode(callSuper = true)`. The default, `callSuper = false`, will silently produce broken equality on subclasses.
- **Includes all non-static, non-transient fields** by default. If you have a derived/cached field marked non-transient, it leaks in. Use `@EqualsAndHashCode.Exclude` or `onlyExplicitlyIncluded = true` plus `@EqualsAndHashCode.Include` per field.
- **JPA proxies**. Lombok-generated `equals` calls `getClass()`. With Hibernate proxies, `getClass()` of a proxy is not the entity class — equality fails between proxy and real instance. Either implement `equals` manually using `Hibernate.unproxy` / `instanceof`, or use id-based equality on entities.
- **Bidirectional associations**. If `Order.equals` includes `customer`, and `Customer.equals` includes `orders`, you get infinite recursion. Always exclude back-references.

Modern alternative: prefer Java records for value objects; reserve Lombok for legacy classes you cannot easily migrate. Even then, `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` is safer than the default.

## Performance: caching hashCode

For deeply-immutable value objects used as map keys at high frequency (think `String`, `BigInteger`), caching the hash makes sense:

```java
public final class Path {
  private final List<String> segments;
  private int cachedHash;            // 0 = not computed (or actually 0)

  @Override public int hashCode() {
    int h = cachedHash;
    if (h == 0) {
      h = segments.hashCode();
      cachedHash = h;
    }
    return h;
  }
}
```

`String` does this in the JDK with the same "0 means unset" trick. Notes:

- Only safe for **truly immutable** objects. Otherwise the cached value goes stale.
- The `0` sentinel collides with the legitimate hash value of 0 — recomputing in that rare case is cheap, so it is acceptable.
- The cache write is a benign data race in practice (the field is `int`, writes are atomic on the JVM, and any thread observing the field either sees `0` and recomputes, or sees the correct value). For strict happens-before, mark it `volatile`.
- For most application code, do **not** cache. Object hashing is fast, and you are optimising the wrong thing.

## Related

- [./ddd-tactical-patterns.md](./ddd-tactical-patterns.md) — Entities vs Value Objects in detail.
- [./type-driven-design.md](./type-driven-design.md) — branded/value types are equality-by-construction.
- [./defensive-copying-and-immutability.md](./defensive-copying-and-immutability.md) — immutability is what makes value-equality safe.
- [./composing-objects-principle.md](./composing-objects-principle.md) — composition vs inheritance affects equality design.
- [../oop-fundamentals/encapsulation.md](../oop-fundamentals/encapsulation.md)
- [../oop-fundamentals/inheritance.md](../oop-fundamentals/inheritance.md)
- [../solid/liskov-substitution-principle.md](../solid/liskov-substitution-principle.md) — equality across hierarchies is the LSP problem in miniature.
- [../design-patterns/creational/builder.md](../design-patterns/creational/builder.md)
- [../design-patterns/behavioral/iterator.md](../design-patterns/behavioral/iterator.md) — equality matters for distinct/dedup.

## References

- Joshua Bloch, *Effective Java* (3rd ed.), items on `equals`, `hashCode`, and immutability.
- *Effective Java* on the inheritance-and-equals symmetry problem.
- JEP 359 / 395 — Java Records.
- JLS §5.6.2 and `java.lang.Object` Javadoc — the official equals/hashCode contract.
- ECMA-262 — `===`, `==`, and `Object.is` semantics.
- Eric Evans, *Domain-Driven Design* — Entity vs Value Object.
- Project Lombok documentation, `@EqualsAndHashCode` configuration options.
