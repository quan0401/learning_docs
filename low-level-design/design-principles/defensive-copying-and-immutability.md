---
title: "Defensive Copying and Immutability Patterns"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, immutability, defensive-copying, persistent-data-structures, records]
---

# Defensive Copying and Immutability Patterns

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `immutability` `defensive-copying` `persistent-data-structures` `records`

## Summary

A class that accepts a `List` and stashes the reference is at the mercy of whoever else holds that reference. They can mutate it later — after the constructor's invariants checked — and quietly corrupt your state. Defensive copying is the discipline of taking your own snapshot at the boundary so external mutation cannot reach into your invariants. Its companion, immutability, is the same idea pushed all the way: if nothing can mutate, no defence is needed.

This doc covers the three classical defensive-copy points (constructor input, setter input, getter output), the difference between an unmodifiable *view* and a true *copy*, the immutable-collection landscape (Java 9+ `List.of`, Guava, Vavr, Kotlin), what records actually freeze, and when defensive copying is overkill. It ends with persistent data structures — the path to immutability that is also performant.

## Table of Contents

- [Why defensive copying](#why-defensive-copying)
- [Three defensive points](#three-defensive-points)
- [Unmodifiable views vs true copies](#unmodifiable-views-vs-true-copies)
- [Immutable collections](#immutable-collections)
- [Records and shallow immutability](#records-and-shallow-immutability)
- [The really-immutable checklist](#the-really-immutable-checklist)
- [TypeScript equivalents](#typescript-equivalents)
- [When defensive copying is overkill](#when-defensive-copying-is-overkill)
- [Persistent data structures](#persistent-data-structures)
- [Related](#related)
- [References](#references)

## Why defensive copying

The bug:

```java
public final class Period {
  private final Date start;
  private final Date end;

  public Period(Date start, Date end) {
    if (start.compareTo(end) > 0) throw new IllegalArgumentException("end before start");
    this.start = start;             // stored reference
    this.end = end;
  }
  public Date start() { return start; }   // leaked reference
}

Date s = new Date();
Date e = new Date(s.getTime() + 1000);
Period p = new Period(s, e);
e.setTime(0);                       // mutate after construction — period invariant violated.
p.start().setTime(Long.MAX_VALUE);  // also violated, via the getter.
```

The constructor verified `start <= end` once and walked away. Two paths can break the invariant: the caller mutates the array/object they passed in, and the caller mutates an object the getter returned.

`java.util.Date` is mutable — that is why this example is famous (Bloch, *Effective Java*). Modern code uses `java.time.Instant` (immutable), and the bug disappears at the source. But many domain types, especially collections, still come in mutable shapes.

## Three defensive points

### 1. Constructor input — copy on the way in

```java
public Period(Date start, Date end) {
  // Copy first, validate after — so a hostile caller cannot mutate between checks.
  this.start = new Date(start.getTime());
  this.end   = new Date(end.getTime());
  if (this.start.compareTo(this.end) > 0)
    throw new IllegalArgumentException("end before start");
}
```

Order matters: copy *before* validating, otherwise a multithreaded attacker can swap the value between check and store (a TOCTOU bug). For collections:

```java
public Order(List<LineItem> items) {
  this.items = List.copyOf(items);   // null-hostile, immutable copy (Java 9+)
}
```

`List.copyOf` returns an unmodifiable list, and copies — two birds, one stone.

### 2. Setter / mutator input — same rule

If your class has setters that take collections, copy there too:

```java
public void replaceItems(List<LineItem> newItems) {
  this.items = List.copyOf(newItems);
}
```

### 3. Getter output — return safe handles

```java
public Date start() { return new Date(start.getTime()); }    // copy
public List<LineItem> items() { return items; }              // ok if `items` is unmodifiable already
```

If your internal collection is already unmodifiable (because you used `List.copyOf` to build it), the getter is safe to return as-is. If it is a mutable internal `ArrayList`, return `Collections.unmodifiableList(items)` or `List.copyOf(items)`.

## Unmodifiable views vs true copies

A common confusion. Two distinct mechanisms:

- **Unmodifiable view** (`Collections.unmodifiableList(list)`) — a wrapper that throws `UnsupportedOperationException` on mutation. **Backed by the original list.** If someone else still holds the underlying mutable list, they can change it and your "unmodifiable" view will reflect those changes.
- **True copy** (`List.copyOf(list)`, `new ArrayList<>(list)` then wrap, `Set.copyOf(set)`) — a new list with new internal storage. Independent of the source.

```java
List<String> backing = new ArrayList<>(List.of("a", "b"));
List<String> view = Collections.unmodifiableList(backing);
List<String> copy = List.copyOf(backing);

backing.add("c");
view.size();   // 3  — view sees mutation!
copy.size();   // 2  — copy is independent.
```

Use **copies** at API boundaries. Use **views** internally where you want to expose a read-only handle to your own data structure without paying for a copy.

## Immutable collections

The Java landscape, ordered roughly by recency:

| API                         | Notes                                                                 |
| --------------------------- | --------------------------------------------------------------------- |
| `Collections.unmodifiable*` | View, not copy. Reflects backing-list mutations. Allows `null`.       |
| `List.of`, `Set.of`, `Map.of` (Java 9+) | True immutable. Reject `null`. Compact memory layout. **No `null` elements allowed.** |
| `List.copyOf` (Java 10+)    | True immutable copy. If input is already an unmodifiable copy, returns it unchanged. |
| Guava `ImmutableList`/`ImmutableSet`/`ImmutableMap` | Battle-tested predecessors of `List.of`. Builder support. Still useful for ordered map/multimap variants. |
| Vavr `io.vavr.collection.List` etc. | **Persistent** — structural sharing, O(1)/O(log n) "modification" returns a new list. |
| Eclipse Collections         | High-performance immutable + primitive specialisations.               |

Kotlin:

- `listOf`/`setOf`/`mapOf` — return `List<T>` (read-only interface), but the underlying object is `ArrayList`. Read-only **interface**, not immutable.
- `kotlinx.collections.immutable` — `persistentListOf`, true persistent collections.

Worked example — locking down a domain object:

```java
public record Order(String id, List<LineItem> items, Map<String, String> metadata) {
  public Order {
    Objects.requireNonNull(id);
    items    = List.copyOf(items);       // canonical copy at construction
    metadata = Map.copyOf(metadata);
  }
}
```

This single record is now safe to share freely — no caller can mutate the items or metadata, and the components are stored as truly immutable copies.

## Records and shallow immutability

Records make all components `final` and provide accessors but no setters. That gives **shallow immutability** — the *reference* in each component is fixed, but the *thing the reference points at* may still be mutable.

```java
record Order(String id, List<LineItem> items) {}

List<LineItem> items = new ArrayList<>(List.of(new LineItem("sku", 1)));
Order o = new Order("1", items);
items.add(new LineItem("sku2", 1));   // mutated! o.items() now has 2 items.
```

The fix, every time, is the compact constructor:

```java
record Order(String id, List<LineItem> items) {
  Order {
    items = List.copyOf(items);
  }
}
```

Same applies to Kotlin `data class` — `val` only freezes the reference. Wrap collections in `toList()` / `toMap()` calls in `init {}`, or use `kotlinx.collections.immutable`.

## The really-immutable checklist

Bloch's "minimise mutability" item (*Effective Java*) gives the criteria for a class to be deeply immutable:

- [ ] **No mutators.** No setters, no methods that change state.
- [ ] **Class is final** (or all constructors are private, with static factories) — so subclasses cannot add mutability.
- [ ] **All fields are `final`.**
- [ ] **All fields are `private`.**
- [ ] **Defensive copy on input** — for any field whose type is mutable.
- [ ] **Defensive copy on output** — or return an unmodifiable view of an immutable internal copy.
- [ ] **No leaking `this` from the constructor.** Don't pass `this` to listeners, executors, or fields of arguments before the constructor returns. Other threads might see a partially-constructed object.
- [ ] **Hash and equals are consistent with deep value semantics** (see [./equality-and-identity.md](./equality-and-identity.md)).

If all eight hold, the class is thread-safe by construction — sharing across threads needs no synchronisation.

## TypeScript equivalents

TypeScript's `readonly` is structural and shallow:

```ts
type Order = {
  readonly id: string;
  readonly items: readonly LineItem[];
};
```

`readonly` is *type-checker enforcement only* — at runtime, the array is still a normal `Array`, mutable via `(items as LineItem[]).push(...)`.

For runtime immutability:

- `Object.freeze(obj)` — shallow freeze, throws in strict mode on writes.
- Deep freeze utilities (`deep-freeze-strict`) — recursive.
- Immer — write code that *looks* mutating, library produces a new immutable structure (uses Proxies + structural sharing under the hood).
- Persistent data structures: Immutable.js (older), `@reduxjs/toolkit` patterns.

```ts
import { produce } from "immer";

const next = produce(order, draft => {
  draft.items.push({ sku: "x", qty: 1 });   // looks mutating, isn't
});
```

Idiomatic modern TS rarely uses Immutable.js — `readonly` types + spread / `produce` covers most cases.

## When defensive copying is overkill

The discipline costs allocations and code. Skip it when:

- **The component is already immutable.** `String`, `Integer`, `BigDecimal`, `Instant`, `UUID`, enums, records-of-immutables — no copy needed.
- **The reference never escapes.** A private internal `ArrayList` used only inside one method, never returned, never passed to listeners — copying it is wasted work.
- **Performance-critical inner loops.** Hot paths handling millions of objects per second may not afford a copy. Document the contract ("caller must not mutate this list after passing it in") and move on. This is the same trade-off the JDK itself makes in `Arrays.asList`.
- **Same-package collaborators with a known contract.** Inside a single bounded module, "we don't mutate each other's collections" can be enforced by review and convention. Across module boundaries, defend.

The default should still be "copy"; the exceptions should be deliberate and commented.

## Persistent data structures

The cost of immutability is allocations: every "modification" creates a new object. For trivial value objects this is fine. For large collections, naïve copy-on-write is O(n) per change — terrible.

**Persistent data structures** solve this with structural sharing. A change returns a new structure that shares most of its internal nodes with the previous version. Common implementations:

- **Hash Array Mapped Trie (HAMT)** — Clojure's `PersistentHashMap`, used in Vavr, Scala, and Immer. ~O(log32 n) for get/set, with very low constant factors.
- **RRB-Trees** — for indexed lists with fast concat/split.

```java
// Vavr persistent list
import io.vavr.collection.List;

List<Integer> a = List.of(1, 2, 3);
List<Integer> b = a.append(4);     // new list, shares internal structure with a
// a is still [1,2,3]; b is [1,2,3,4]
```

This is the path that lets you write fully-immutable domain code without paying linear cost on every update. It also enables time-travel debugging, undo/redo, and lock-free concurrent reads of arbitrary snapshots.

The trade-off is API friction: persistent collections do not implement `java.util.List`, so interop with libraries that expect JDK collections requires conversion. Pick them when the immutability ergonomics outweigh the interop tax — typically in domain cores, less so at framework boundaries.

For more on the functional side of this, see how it composes with [./type-driven-design.md](./type-driven-design.md): persistent collections are the data layer that makes "construct a new value rather than mutate" practical at scale.

## Related

- [./type-driven-design.md](./type-driven-design.md) — illegal states unrepresentable + immutability are the same project.
- [./equality-and-identity.md](./equality-and-identity.md) — value-equality only makes sense on immutable values.
- [./ddd-tactical-patterns.md](./ddd-tactical-patterns.md) — Value Objects are immutable by definition.
- [./composing-objects-principle.md](./composing-objects-principle.md) — composition over inheritance pairs naturally with immutable building blocks.
- [./separation-of-concerns.md](./separation-of-concerns.md)
- [../oop-fundamentals/encapsulation.md](../oop-fundamentals/encapsulation.md) — defensive copying is encapsulation enforcement.
- [../oop-fundamentals/abstraction.md](../oop-fundamentals/abstraction.md)
- [../solid/single-responsibility-principle.md](../solid/single-responsibility-principle.md)
- [../solid/open-closed-principle.md](../solid/open-closed-principle.md)
- [../design-patterns/creational/builder.md](../design-patterns/creational/builder.md) — the Builder is the canonical way to assemble an immutable object with many fields.
- [../design-patterns/creational/prototype.md](../design-patterns/creational/prototype.md) — copy-based construction.
- [../design-patterns/structural/flyweight.md](../design-patterns/structural/flyweight.md) — sharing is safe because the shared object is immutable.

## References

- Joshua Bloch, *Effective Java* (3rd ed.), items on minimising mutability and on defensive copies.
- JEP 359 / 395 — Java Records.
- Java API: `List.of`, `List.copyOf`, `Collections.unmodifiableList`.
- Guava `ImmutableCollections` documentation.
- Vavr documentation — persistent collections.
- Phil Bagwell, "Ideal Hash Trees" — original HAMT paper.
- Rich Hickey's Clojure talks on persistent data structures and structural sharing.
- Immer documentation (TypeScript).
