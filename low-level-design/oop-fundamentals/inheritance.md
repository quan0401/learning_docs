---
title: "Inheritance — When and When Not"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, inheritance, composition]
---

# Inheritance — When and When Not

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `inheritance` `composition`

## Summary

Inheritance lets a subclass extend a parent's state and behaviour. It is the most overused OOP feature: powerful for true *is-a* relationships, fragile and inflexible for everything else. Composition + interfaces is usually the better default; inheritance earns its keep only when the subclass is genuinely a substitutable specialization of the parent.

## Table of Contents

- [Is-a relationships](#is-a-relationships)
- [Single vs multiple inheritance](#single-vs-multiple-inheritance)
- [The diamond problem](#the-diamond-problem)
- [Composition over inheritance](#composition-over-inheritance)
- [The fragile base class problem](#the-fragile-base-class-problem)
- [When inheritance is the right tool](#when-inheritance-is-the-right-tool)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## Is-a relationships

Inheritance models *is-a*: a `Square` *is a* `Shape`; a `BufferedReader` *is a* `Reader`. The Liskov Substitution Principle (LSP) is the formal version: anywhere code expects a `Parent`, you can give it a `Child` and the program still behaves correctly.

```java
public class Reader { /* read() returns next char or -1 */ }
public class BufferedReader extends Reader { /* same contract, just buffers */ }

void consume(Reader r) { /* works for any Reader, including Buffered */ }
```

When *is-a* fails, inheritance is wrong. The classic counter-example:

```java
public class Rectangle {
    private int w, h;
    public void setW(int w) { this.w = w; }
    public void setH(int h) { this.h = h; }
    public int area() { return w * h; }
}

public class Square extends Rectangle {
    @Override public void setW(int w) { super.setW(w); super.setH(w); }
    @Override public void setH(int h) { super.setW(h); super.setH(h); }
}

void grow(Rectangle r) {
    r.setW(5);
    r.setH(4);
    assert r.area() == 20;     // breaks for Square — area is 16
}
```

Mathematically a square is a rectangle. Behaviourally, with mutable width and height, a `Square` is *not* substitutable for a `Rectangle`. Inheritance is about behaviour, not taxonomy.

A useful test: write down what every parent method *promises* (preconditions, postconditions, invariants). Can the subclass honour all of them? If no — don't inherit.

## Single vs multiple inheritance

| Form | Languages | Notes |
|---|---|---|
| Single class inheritance | Java, C#, Kotlin, Swift, JS/TS | One direct superclass |
| Multiple class inheritance | C++, Python, Eiffel, Perl | Multiple superclasses |
| Multiple interface inheritance | Java, C#, Kotlin, TS, Swift | Many contracts, no state collision |
| Mixins / traits | Scala (traits), Rust (traits), Ruby (modules), TS (mixin pattern) | Reusable behaviour grafted in |

Most modern OO languages chose **single inheritance + multiple interfaces** to avoid the issues of multiple class inheritance while keeping its benefits. You inherit *state and base behaviour* from at most one parent, and you *implement contracts* from many interfaces.

```java
public class HttpServer
        extends AbstractServer        // single parent for state + lifecycle
        implements MetricsReporter,   // many contracts
                   HealthCheckable,
                   GracefulShutdown {
    /* ... */
}
```

## The diamond problem

When a class inherits from two classes that both inherit from a common ancestor, *which* version of an inherited method does it get?

```
        A
       / \
      B   C
       \ /
        D
```

If `A` has `foo()`, and both `B` and `C` override it, what does `D.foo()` do? This is the diamond problem.

How different languages answer:

- **C++**: virtual inheritance must be opted in to share `A`. Without it you get two `A` subobjects. Programmer must specify which `foo()` to use.
- **Python**: uses MRO (Method Resolution Order) via the C3 linearization algorithm. Predictable but subtle.
- **Java / C# (classes)**: side-stepped — you cannot inherit from two classes.
- **Java (default methods on interfaces)**: if two interfaces provide a default `foo()`, the implementer must explicitly resolve via `Interface.super.foo()`.
- **Scala (traits)**: linearization order resolves ambiguity.

```java
interface Walker { default String go() { return "walking"; } }
interface Swimmer { default String go() { return "swimming"; } }

class Frog implements Walker, Swimmer {
    @Override public String go() {
        return Walker.super.go() + " then " + Swimmer.super.go();
    }
}
```

Interfaces with default methods reproduce a small slice of multiple inheritance — but because interfaces have no state, the diamond is much smaller and easier to resolve.

## Composition over inheritance

> "Favor object composition over class inheritance." — *Design Patterns*, Gamma et al.

Composition: an object *has* another object and delegates work to it. Inheritance: an object *is* another object and reuses its members.

```java
// Inheritance — Stack extends Vector. Inherits all of Vector's mutating API,
// breaking the LIFO contract because callers can call insertElementAt(...).
public class Stack<E> extends Vector<E> { /* ... */ }   // historical Java mistake

// Composition — Stack has-a list, exposes only push/pop/peek.
public class BetterStack<E> {
    private final ArrayList<E> data = new ArrayList<>();
    public void push(E e) { data.add(e); }
    public E pop() { return data.remove(data.size() - 1); }
    public E peek() { return data.get(data.size() - 1); }
}
```

Why composition usually wins:

1. **Smaller surface area.** You expose only what you intend.
2. **No fragile base class.** Changes inside `ArrayList` cannot break `BetterStack` so long as the methods it uses still behave.
3. **Runtime flexibility.** Composition can be reconfigured (swap dependencies via DI). Inheritance is fixed at compile time.
4. **No accidental contract violation.** No risk that an inherited method silently does the wrong thing.
5. **Composes with interfaces cleanly.** Implement the contracts you want; delegate to the helper for behavior.

```typescript
// TypeScript — composition + interface
interface Stack<T> {
  push(x: T): void;
  pop(): T;
  peek(): T;
}

class ArrayStack<T> implements Stack<T> {
  private readonly data: T[] = [];
  push(x: T) { this.data.push(x); }
  pop(): T   {
    const v = this.data.pop();
    if (v === undefined) throw new Error("empty");
    return v;
  }
  peek(): T  { return this.data[this.data.length - 1]; }
}
```

Inheritance still has its place (see below), but the default reach should be composition.

## The fragile base class problem

A subclass depends on the *behaviour* of its superclass, not just its public API. The base class can change in a way that compiles fine but silently breaks subclasses.

Classic example: a `HashSet` subclass that wants to count add operations.

```java
public class CountingHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);    // fine in some Java versions, broken in others
    }

    public int getAddCount() { return addCount; }
}

var s = new CountingHashSet<String>();
s.addAll(List.of("a", "b", "c"));
// addCount might be 3 or 6 depending on whether HashSet.addAll calls add() internally.
```

The subclass made an *assumption* about `addAll`'s implementation. If `HashSet.addAll` happens to call `add` for each element, every element is counted twice. If it bulk-inserts, only `addAll` increments. Either way, the subclass is fragile against base-class implementation changes.

Defenses against fragility:

- **Document overridable behavior.** "These methods invoke each other; if you override `add`, also override `addAll`."
- **Mark classes `final` by default.** Effective Java item 19: "Design and document for inheritance or else prohibit it." Java's `record` and Kotlin's classes are `final` by default for this reason.
- **Prefer composition.** A `CountingSet` that wraps a `Set` and forwards calls is unaffected by the wrapped set's internal call graph.

```java
public class CountingSet<E> implements Set<E> {
    private final Set<E> inner;
    private int addCount = 0;

    public CountingSet(Set<E> inner) { this.inner = inner; }

    @Override public boolean add(E e) {
        addCount++;
        return inner.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return inner.addAll(c);
    }
    /* delegate the rest of Set... */
    public int getAddCount() { return addCount; }
}
```

Now `addCount` is correct regardless of whether the underlying `Set` calls `add` from `addAll`.

## When inheritance is the right tool

Inheritance pays off when *all* of these hold:

- The relationship is genuinely *is-a* and the LSP holds.
- The base class is designed for extension and documents its overridable hooks.
- Multiple subclasses share substantial state and behaviour, not just signatures.
- You control the base class (or it is part of a stable framework).

Examples that fit:

- Framework hooks (`AbstractHandler`, `HttpServlet`, Android's `Activity`).
- Sealed type hierarchies where the parent is the closed sum-type root.
- Algorithm template methods (the parent owns the flow; subclasses fill in steps).
- Domain hierarchies with truly shared invariants (`AbstractAccount` ➜ `CheckingAccount`, `SavingsAccount`).

Examples that don't:

- "I want to reuse a method from `Util`." Use a static helper or composition.
- "All my entities have an `id`." Use a tiny interface or a shared mapped superclass — but consider composition first.
- "I want a `Stack` based on a `List`." Compose, don't extend.

## Common pitfalls

- **Inheritance for code reuse alone.** If the only reason is "the parent has methods I want," compose.
- **Deep hierarchies.** Three levels is suspicious; five is a maintenance disaster. Each level adds coupling and constrains the lower levels.
- **`protected` everywhere.** Treats every subclass as trusted. The bigger the protected surface, the more fragile the base.
- **Overriding without `@Override`.** In Java, always annotate. Without it, a typo in the method name silently creates a new method instead of overriding.
- **Calling overridable methods from constructors.** The subclass's override runs against a not-yet-fully-constructed object. Either make those methods `final` or refactor.
- **Ignoring LSP.** Subclass throws `UnsupportedOperationException` for some parent method? You've broken substitutability. The hierarchy is wrong.

## Related

- [classes-and-objects.md](./classes-and-objects.md) — the units being inherited
- [interfaces.md](./interfaces.md) — multiple interface inheritance as the safe alternative
- [polymorphism.md](./polymorphism.md) — what dispatch actually does once you've inherited
- [encapsulation.md](./encapsulation.md) — `protected` and what subclasses can see
- [abstraction.md](./abstraction.md) — abstract classes as a partial implementation
- [../../java/INDEX.md](../../java/INDEX.md) — Java inheritance, sealed types, final classes
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — TS classes, mixins, composition patterns
