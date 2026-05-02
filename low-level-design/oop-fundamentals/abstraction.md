---
title: "Abstraction — Modeling at the Right Level"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, abstraction, modeling]
---

# Abstraction — Modeling at the Right Level

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `abstraction` `modeling`

## Summary

Abstraction is the act of choosing what to expose and what to ignore so a problem is tractable at the level you care about. A good abstraction names the right concept; a bad one either smears multiple concepts together or pretends a hard concept is easy. Abstraction is *not* the same as encapsulation — abstraction picks the level, encapsulation defends the boundary.

## Table of Contents

- [Abstract classes](#abstract-classes)
- [Abstract data types](#abstract-data-types)
- [Picking the abstraction line](#picking-the-abstraction-line)
- [Leaky abstractions](#leaky-abstractions)
- [Abstraction vs encapsulation](#abstraction-vs-encapsulation)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## Abstract classes

An abstract class is a partial implementation: it declares some methods, supplies some others, and leaves at least one abstract method for subclasses to fill in. It cannot be instantiated directly.

```java
public abstract class HttpClient {
    public final Response send(Request req) {     // template method — final, shared flow
        var augmented = addDefaultHeaders(req);
        log(augmented);
        return doSend(augmented);                  // hook — subclass-specific
    }

    protected abstract Response doSend(Request req);

    protected Request addDefaultHeaders(Request req) { /* shared default */ }
    protected void log(Request req) { /* shared default */ }
}

public class JdkHttpClient extends HttpClient {
    @Override
    protected Response doSend(Request req) { /* uses java.net.http */ }
}
```

```typescript
abstract class HttpClient {
  send(req: Request): Promise<Response> {
    const augmented = this.addDefaultHeaders(req);
    this.log(augmented);
    return this.doSend(augmented);
  }
  protected abstract doSend(req: Request): Promise<Response>;
  protected addDefaultHeaders(req: Request): Request { return req; }
  protected log(req: Request): void { /* ... */ }
}
```

When to use an abstract class instead of an interface:

- You want to share *implementation*, not just a contract.
- You want a template-method skeleton with subclass-supplied steps.
- You need protected helper methods or shared state.

When to prefer an interface:

- You only need a contract.
- You want multiple implementers from unrelated hierarchies.
- You want freedom to mix in via composition.

In Java, prefer interfaces with default methods for shared *behaviour* and abstract classes only when you also need shared *state*. In TypeScript, abstract classes are useful when interfaces aren't enough; for "behaviour but no state," an interface plus a helper module is often cleaner.

## Abstract data types

An Abstract Data Type (ADT) is a specification of a type purely in terms of operations and their behaviour, independent of representation. `List`, `Set`, `Map`, `Queue`, `Stack` are classic ADTs.

The ADT for a stack might say:

- `push(x)` — adds `x` to the top.
- `pop()` — removes and returns the top; error if empty.
- `peek()` — returns the top without removing; error if empty.
- `size()` — number of elements.

Three different concrete classes can satisfy this ADT: array-backed, linked-list-backed, ring-buffer-backed. The caller depends on the ADT and is unaffected by the choice.

```java
Deque<Integer> stack = new ArrayDeque<>(); // could swap for LinkedList; same ADT
stack.push(1);
stack.push(2);
stack.pop(); // 2
```

```typescript
class Stack<T> {
  private readonly data: T[] = [];
  push(x: T): void { this.data.push(x); }
  pop(): T {
    const v = this.data.pop();
    if (v === undefined) throw new Error("empty");
    return v;
  }
  peek(): T { return this.data[this.data.length - 1]; }
  size(): number { return this.data.length; }
}
```

The ADT is the *what*; the data structure is the *how*. Programming against the ADT lets you change the representation without rippling.

## Picking the abstraction line

The hardest part of abstraction is choosing where to draw it. A few practical heuristics.

### 1. Name the *concept*, not the implementation

`PaymentGateway` is a concept. `StripeService` is an implementation. `StripeBasedPaymentProcessorImpl` is a smell.

### 2. Group operations that change together

If `read` and `write` always change together, they probably belong on the same abstraction. If `read` is stable and `write` keeps adding parameters, you may have two concepts forced into one.

### 3. Don't abstract until the second use

The first occurrence is data. The second is a coincidence. The third is a pattern. Premature abstraction baked in from one example is usually wrong, because you don't yet know which axes vary. (Common refactor: "wrong abstraction" is more painful than "duplication.")

### 4. Match the abstraction to the audience

A `PaymentGateway` exposed to product code should speak in `Money`, `Customer`, `Result`. A `RawHttpClient` exposed to gateway implementations speaks in headers and bytes. The same concept lives at multiple levels; pick the one for the caller.

### 5. Prefer explicit over magic

A method named `process(Object)` that does forty things based on instanceof checks isn't an abstraction — it's a switch statement in a trench coat. Multiple named methods are usually a better abstraction than one polymorphic method with implicit branches.

## Leaky abstractions

> **The Law of Leaky Abstractions** (Joel Spolsky, 2002): *All non-trivial abstractions, to some degree, are leaky.*

The point is not "abstractions are bad" but "the abstraction can never fully hide the layer below." Examples:

- TCP abstracts a reliable byte stream — but you still must handle latency, head-of-line blocking, and connection resets.
- ORMs abstract SQL — until N+1 queries appear and you must reach for the underlying SQL.
- HTTP libraries abstract sockets — until DNS, TLS, or proxy weirdness leaks through.
- File systems abstract storage — until you hit case sensitivity, max path length, or sync vs async fsync semantics.
- React abstracts the DOM — until layout thrash, focus restoration, or scroll position betrays you.

Practical implications:

1. **Pick abstractions whose leaks you can live with.** An ORM is great until your hottest path becomes the leak. Then you drop into raw SQL for that path.
2. **Document the leaks.** "This client retries on 429 but not on 503" is a leak the caller must know about.
3. **Provide escape hatches.** Good libraries expose a high-level API for 95% of cases and a low-level API for the 5% where the abstraction breaks.
4. **Don't try to hide what cannot be hidden.** Latency, partial failure, and concurrency cannot be abstracted away — only modeled.

## Abstraction vs encapsulation

These two get conflated constantly. They are different concerns.

| Concern | Abstraction | Encapsulation |
|---|---|---|
| Question | *What* should be exposed? | *How* do we prevent misuse? |
| Tool | Interfaces, abstract classes, naming, layering | Visibility modifiers, immutability, defensive copies |
| Failure mode | Wrong concept exposed | Right concept exposed but invariants leak |
| Lives at | Design / API surface | Implementation / class boundary |

A class can be perfectly encapsulated (no public mutable state) and still a terrible abstraction (the methods don't model anything coherent). It can also be a clean abstraction with leaky encapsulation (right methods, but they let callers mutate internals).

Example of clean encapsulation, bad abstraction:

```java
public class DataManager {
    private final Map<String, Object> data = new HashMap<>();
    public Optional<Object> get(String key) { return Optional.ofNullable(data.get(key)); }
    public void put(String key, Object value) { data.put(key, value); }
}
```

Encapsulation is fine — `data` is private. Abstraction is dismal — `DataManager` doesn't model anything specific. It's a Map with extra steps.

Example of clean abstraction, weak encapsulation:

```java
public class Order {
    public final List<LineItem> items;          // exposed mutable list
    public final OrderStatus status;
    /* methods that name real domain operations */
}
```

The abstraction `Order` is a clear concept. The encapsulation is broken — anyone can mutate `items` directly.

Aim for both. Get the abstraction right first (rename, restructure, until the type *means something*), then defend it with encapsulation.

## Common pitfalls

- **Abstracting things that don't vary.** If there will only ever be one implementation, the interface is overhead. YAGNI.
- **Abstracting the wrong axis.** Splitting `User` from `Admin` via inheritance when the difference is *role* rather than *kind* — leads to refactors when an admin can also be a regular user.
- **Generic-looking names.** `Manager`, `Helper`, `Service`, `Util`, `Handler`, `Processor`, `Data` — these are placeholders for "I don't know what this is yet." Push for a more specific noun.
- **Hiding behaviour callers need.** A timeout, a retry budget, an idempotency key — sometimes these *belong* in the abstraction, even though they feel like implementation details.
- **Treating frameworks as abstractions you fully understand.** They leak. Read the source when the abstraction puzzles you.
- **Too many layers.** Five abstraction layers between controller and database, each a thin wrapper around the next, is worse than one chunky layer that does its job.

## Related

- [classes-and-objects.md](./classes-and-objects.md) — what an object is, before deciding what to abstract
- [encapsulation.md](./encapsulation.md) — the boundary defense
- [interfaces.md](./interfaces.md) — the most common abstraction tool
- [inheritance.md](./inheritance.md) — abstract classes vs concrete bases
- [../../java/INDEX.md](../../java/INDEX.md) — abstract classes, sealed types, records
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — abstract classes vs interfaces vs union types
