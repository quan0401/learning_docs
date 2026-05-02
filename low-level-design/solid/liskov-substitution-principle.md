---
title: "Liskov Substitution Principle (LSP)"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, solid, lsp, polymorphism, contracts]
---

# Liskov Substitution Principle (LSP)

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `solid` `lsp` `polymorphism` `contracts`

## Summary

The Liskov Substitution Principle says objects of a subtype must be usable wherever objects of the base type are expected, without breaking program correctness. Barbara Liskov's 1987 keynote and the 1994 Liskov–Wing paper formalized this as a behavioral contract: subtypes may not strengthen preconditions, weaken postconditions, or violate invariants of the supertype. LSP is what makes polymorphism actually safe; without it, every `is-a` relationship is a potential trap.

## Table of Contents

- [Liskov's Behavioral Definition](#liskovs-behavioral-definition)
- [The Contract: Preconditions, Postconditions, Invariants](#the-contract-preconditions-postconditions-invariants)
- [The Rectangle / Square Classic](#the-rectangle--square-classic)
- [Why LSP Matters for Polymorphism](#why-lsp-matters-for-polymorphism)
- [Common LSP Violations](#common-lsp-violations)
- [Designing Substitutable Hierarchies](#designing-substitutable-hierarchies)
- [Related](#related)

## Liskov's Behavioral Definition

Liskov's original phrasing, paraphrased:

> If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program.

Note: it says nothing about syntactic compatibility. The compiler will let you substitute any subclass; LSP is a *behavioral* requirement on top of that. A subtype that compiles fine but throws `UnsupportedOperationException` for a documented base-class method violates LSP, even though Java's type system shrugs.

## The Contract: Preconditions, Postconditions, Invariants

LSP can be stated as three rules on the subtype's method contracts versus the supertype's:

| Rule | Required relation | Meaning |
|------|------|------|
| Preconditions | Subtype may **weaken** (accept more) | Subtype must accept any input the supertype accepts. |
| Postconditions | Subtype may **strengthen** (guarantee more) | Subtype must deliver everything the supertype promised. |
| Invariants | Subtype must **preserve** | Anything always true of the supertype must remain true. |

Plus the **history rule**: subclasses may not allow state changes that the base class forbids over the object's lifetime. An immutable `Point` should not have a mutable subclass.

A practical mnemonic: a subtype is allowed to be more *generous* in what it accepts and more *strict* in what it produces. It cannot be pickier on input or sloppier on output than its parent.

## The Rectangle / Square Classic

The textbook example. Mathematically, every square is a rectangle. In code, modeling `Square extends Rectangle` breaks LSP.

```java
public class Rectangle {
    protected int width;
    protected int height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override public void setWidth(int w) { this.width = w; this.height = w; }
    @Override public void setHeight(int h) { this.width = h; this.height = h; }
}
```

Now consider client code that depends on the rectangle contract:

```java
void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.area() == 20;   // expected by the Rectangle contract
}
```

Pass a `Square`: `setHeight(4)` also changes the width to 4, so `area()` is 16. The assertion fails. The subtype silently strengthened the *invariant* (`width == height`) and broke the *postcondition* of `setHeight` (which the rectangle promised would only change height).

The fix: don't model `Square` as a subtype of `Rectangle`. They share a mathematical relationship but not a behavioral one. Either:

- Make both implement a read-only `Shape` interface with `area()`.
- Compose: `Square` *has* a `Rectangle` internally if you really need that.

```java
public interface Shape { int area(); }

public final class Rectangle implements Shape {
    private final int width, height;
    public Rectangle(int w, int h) { this.width = w; this.height = h; }
    public int area() { return width * height; }
}

public final class Square implements Shape {
    private final int side;
    public Square(int s) { this.side = s; }
    public int area() { return side * side; }
}
```

The lesson generalizes: **`is-a` in the real world is not the same as `is-substitutable-for` in code.**

## Why LSP Matters for Polymorphism

Polymorphism is the feature that lets a single call site work with many concrete types. LSP is the discipline that keeps that feature safe.

When you write `for (Shape s : shapes) total += s.area();`, you're betting that *every* `Shape` you'll ever encounter — including ones written next year by a colleague who's never read your code — will behave correctly when treated as a `Shape`. The base contract is the only thing the call site can rely on.

LSP violations punish this exact pattern. A `Bird.fly()` method on a base class forces `Penguin` to either throw or no-op, and any code that calls `fly()` on a list of `Bird`s gets a runtime surprise. The Open/Closed Principle wants you to add new subtypes without editing call sites; LSP is the contract that makes OCP safe.

## Common LSP Violations

**Throwing `UnsupportedOperationException`.** A subclass overrides a method to throw because "it doesn't make sense here." That's a structural admission that the type hierarchy is wrong.

```java
class ReadOnlyList<T> extends ArrayList<T> {
    @Override public boolean add(T x) { throw new UnsupportedOperationException(); }
}
```

Clients of `List` reasonably call `add`. Either don't extend `ArrayList`, or split into a separate `List` (read) and `MutableList` (write) hierarchy.

**Strengthening preconditions.** A base method accepts any positive integer; the override rejects values over 100. Existing call sites that pass 200 break.

**Returning null where the base promised non-null.** A weakened postcondition. Equally bad: throwing where the base contract said it would return.

**Changing observable state in surprising ways.** Like the square setting both dimensions in `setWidth`. The subtype kept the method *signature* but broke the method *behavior*.

```typescript
// LSP violation in TypeScript
class Bird {
    fly(): void { console.log("flap flap"); }
}

class Penguin extends Bird {
    override fly(): void { throw new Error("penguins don't fly"); }
}

function migrate(birds: Bird[]): void {
    for (const b of birds) b.fly();   // boom on penguins
}
```

Better:

```typescript
interface Bird { eat(): void; }
interface FlyingBird extends Bird { fly(): void; }
interface SwimmingBird extends Bird { swim(): void; }

class Sparrow implements FlyingBird {
    eat(): void {}
    fly(): void {}
}
class Penguin implements SwimmingBird {
    eat(): void {}
    swim(): void {}
}

function migrate(birds: readonly FlyingBird[]): void {
    for (const b of birds) b.fly();   // type system enforces correctness
}
```

## Designing Substitutable Hierarchies

A few practical heuristics:

- **Write the base contract explicitly.** What does `Shape.area()` promise? What does `List.add()` accept and guarantee? Document preconditions and postconditions, even informally. Subclass authors need a target.
- **Prefer interfaces with narrow contracts.** A small interface is easier to honor. Sprawling base classes with twenty methods create many places to violate the contract.
- **Favor composition for "kind of like" relationships.** If a subclass keeps wanting to disable, override, or warp parent behavior, it isn't really a subtype.
- **Use `final` on classes and methods you don't intend to be overridden.** Java and TypeScript both let you close off extension points; doing so prevents accidental LSP violations.
- **Test through the base type.** If you have a `Shape` test suite, run every concrete shape through it. Any test that passes for `Rectangle` and fails for `Square` is an LSP signal.
- **Beware mutable base, immutable subtype (or vice versa).** Mutability is part of the contract. Mixing the two violates the history rule.

A property-based test pattern is particularly useful here. Define the base contract as a set of properties, then run every subtype through them:

```java
abstract class ShapeContractTest {
    protected abstract Shape createShape();

    @Test
    void areaIsNonNegative() {
        assertThat(createShape().area()).isGreaterThanOrEqualTo(0);
    }

    @Test
    void areaIsDeterministic() {
        Shape s = createShape();
        assertThat(s.area()).isEqualTo(s.area());
    }
}

class RectangleContractTest extends ShapeContractTest {
    @Override protected Shape createShape() { return new Rectangle(3, 4); }
}

class CircleContractTest extends ShapeContractTest {
    @Override protected Shape createShape() { return new Circle(5); }
}
```

Every concrete subtype reuses the contract suite. A new shape that fails any base property is exposed as an LSP violation immediately, before it ships.

LSP is the principle that turns inheritance from a clever language feature into a sound design tool. Most senior engineers who avoid deep inheritance hierarchies are, implicitly, avoiding the cost of getting LSP right at depth — and that avoidance is itself a defensible design choice.

## Related

- [Single Responsibility Principle (SRP)](./single-responsibility-principle.md)
- [Open/Closed Principle (OCP)](./open-closed-principle.md)
- [Interface Segregation Principle (ISP)](./interface-segregation-principle.md)
- [Dependency Inversion Principle (DIP)](./dependency-inversion-principle.md)
- [../oop-fundamentals/inheritance.md](../oop-fundamentals/inheritance.md)
- [../oop-fundamentals/polymorphism.md](../oop-fundamentals/polymorphism.md)
- [../solid/liskov-substitution-principle.md](../solid/liskov-substitution-principle.md)
