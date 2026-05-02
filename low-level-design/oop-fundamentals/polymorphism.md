---
title: "Polymorphism — Subtype, Parametric, Ad-Hoc"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, polymorphism, generics, dispatch]
---

# Polymorphism — Subtype, Parametric, Ad-Hoc

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `polymorphism` `generics` `dispatch`

## Summary

Polymorphism is "one name, many forms." There are three flavours that are constantly conflated: subtype polymorphism (one method dispatched to different classes at runtime), parametric polymorphism (generics — one definition, many type instantiations), and ad-hoc polymorphism (overloading and operator overloading — same name, different functions chosen by argument types). Most design patterns rely on subtype polymorphism, so understanding dispatch is foundational.

## Table of Contents

- [Subtype polymorphism](#subtype-polymorphism)
- [Parametric polymorphism](#parametric-polymorphism)
- [Ad-hoc polymorphism](#ad-hoc-polymorphism)
- [Dynamic vs static dispatch](#dynamic-vs-static-dispatch)
- [Polymorphism in design patterns](#polymorphism-in-design-patterns)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## Subtype polymorphism

The most familiar form. A reference of type `Parent` may at runtime point to any subclass; calling a virtual method dispatches to the actual class's implementation.

```java
public interface Shape { double area(); }
public class Circle implements Shape {
    private final double r;
    public Circle(double r) { this.r = r; }
    public double area() { return Math.PI * r * r; }
}
public class Square implements Shape {
    private final double s;
    public Square(double s) { this.s = s; }
    public double area() { return s * s; }
}

double total(List<Shape> shapes) {
    return shapes.stream().mapToDouble(Shape::area).sum();
}
```

`total` doesn't know or care what concrete shapes it has — only that each one responds to `area`. Adding `Triangle` later does not require touching `total`.

This is the engine of OO design. The Strategy, Template Method, State, Command, and many other patterns are essentially subtype polymorphism applied to a specific situation.

```typescript
interface Shape { area(): number; }
class Circle implements Shape {
  constructor(private readonly r: number) {}
  area(): number { return Math.PI * this.r ** 2; }
}
class Square implements Shape {
  constructor(private readonly s: number) {}
  area(): number { return this.s ** 2; }
}

const total = (shapes: Shape[]): number => shapes.reduce((a, s) => a + s.area(), 0);
```

## Parametric polymorphism

A single definition that works uniformly for many types — generics. The function or class is parameterised by a type variable.

```java
public class Box<T> {
    private final T value;
    public Box(T value) { this.value = value; }
    public T get() { return value; }
}

Box<String>  s = new Box<>("hi");
Box<Integer> i = new Box<>(42);
```

```typescript
class Box<T> {
  constructor(private readonly value: T) {}
  get(): T { return this.value; }
}

const s = new Box<string>("hi");
const i = new Box<number>(42);
```

Generics enforce that the operations work for *any* `T`, which means the body of `Box` cannot make assumptions about `T`. To assume more, *constrain* the type variable.

```java
public <T extends Comparable<T>> T max(List<T> xs) {
    T best = xs.get(0);
    for (T x : xs) if (x.compareTo(best) > 0) best = x;
    return best;
}
```

```typescript
function max<T>(xs: T[], cmp: (a: T, b: T) => number): T {
  let best = xs[0];
  for (const x of xs) if (cmp(x, best) > 0) best = x;
  return best;
}
```

### Erasure vs reified generics

| Language | Strategy | Consequence |
|---|---|---|
| Java | Type erasure — `List<String>` becomes `List<Object>` at runtime | No `instanceof List<String>`; one class file for all type args |
| C# | Reified — type info preserved at runtime | Can check `typeof(T)`; per-instantiation specialization |
| TypeScript | Erased entirely after compile | No runtime generic info at all |
| Kotlin | Erased like Java; `inline` + `reified` opt-in for runtime types | Only inside inline functions |
| Swift, Rust, C++ | Monomorphization (per-type code) | Fast at runtime, larger code size |

This affects what you can do at runtime. In Java/TS, you cannot reflect on `T`; you must pass a `Class<T>` token or shape descriptor when you need it.

## Ad-hoc polymorphism

Same name, multiple definitions chosen by argument types (overloading) or by operator (operator overloading). The compiler picks at compile time based on the static types of the arguments.

```java
public class Logger {
    public void log(String msg) { /* ... */ }
    public void log(int code, String msg) { /* ... */ }
    public void log(Throwable t) { /* ... */ }
}

new Logger().log("started");
new Logger().log(500, "internal error");
new Logger().log(new IOException("boom"));
```

TypeScript supports overload signatures that resolve to a single implementation:

```typescript
class Logger {
  log(msg: string): void;
  log(code: number, msg: string): void;
  log(err: Error): void;
  log(a: unknown, b?: string): void {
    // implementation handles all variants
  }
}
```

Java does not support operator overloading (other than `+` for strings). C++, C#, Kotlin, Swift, and Python do — with various restrictions. Operator overloading is convenient for value-like types (`Money + Money`, `Vector + Vector`) but quickly degenerates if used to mean unrelated things.

### Type classes / extensions as ad-hoc polymorphism

Modern languages express ad-hoc polymorphism through traits / type classes / protocols / extension methods:

- **Rust:** `impl Trait for MyType` adds methods retroactively.
- **Swift:** `extension MyType: Protocol`.
- **Kotlin / C#:** extension functions/methods.
- **Haskell / Scala:** type classes / implicits / `given`.

These let you add behaviour to existing types without inheritance — frequently a cleaner alternative.

## Dynamic vs static dispatch

When does the runtime decide which method to call?

| Form | Decided | Examples |
|---|---|---|
| Static dispatch | Compile time, by static type | Java overloading, C++ non-virtual methods, Rust generic monomorphization |
| Dynamic dispatch | Runtime, by actual object | Java instance methods (all virtual unless `final`/`private`/`static`), TS class methods, C++ `virtual`, Rust `dyn Trait` |

A subtle Java example:

```java
class Animal { public String sound() { return "generic"; } }
class Dog extends Animal { public String sound() { return "woof"; } }

Animal a = new Dog();
a.sound();   // "woof" — DYNAMIC dispatch on the actual object

class Logger {
    public static void log(Animal a) { System.out.println("animal"); }
    public static void log(Dog d)    { System.out.println("dog"); }
}
Logger.log(a); // "animal" — STATIC dispatch chooses overload by static type Animal
```

Instance method `sound` is virtual: dispatched on what `a` actually is at runtime. Overloaded `static` `log` is chosen at compile time based on the declared type of `a`.

```typescript
class Animal { sound(): string { return "generic"; } }
class Dog extends Animal { sound(): string { return "woof"; } }

const a: Animal = new Dog();
a.sound(); // "woof" — methods on prototype chain resolve dynamically
```

### Devirtualization and inlining

Modern JVMs and V8 detect when a virtual call always hits one implementation and inline it for free. Marking methods `final` or classes `final` is a hint to humans more than to the JIT — but it also enables aggressive optimization on AOT compilers (GraalVM, ahead-of-time .NET, etc.).

Performance-wise: dynamic dispatch costs a vtable lookup (one indirection). On hot paths in tight loops it can matter; in 99% of code it does not.

## Polymorphism in design patterns

Most GoF patterns are clever uses of subtype polymorphism. A few examples:

- **Strategy.** Pluggable algorithms behind a common interface. `SortStrategy` with `QuickSort`, `MergeSort`. Clients depend on `SortStrategy.sort(...)`.
- **State.** The object holds a `State` field; methods delegate to the state, which knows how to behave and which state to transition to.
- **Template Method.** A base class defines the algorithm skeleton with abstract hook methods; subclasses fill them in.
- **Command.** Each action is a small object implementing `Command.execute()`. Queues, undo stacks, schedulers all work polymorphically.
- **Visitor.** Double dispatch — pick the right method based on both the visitor and the visited type. The classic workaround in single-dispatch languages for what multimethods give you for free.
- **Decorator.** Wraps an object that implements an interface, presenting the same interface with extra behaviour. Stack decorators arbitrarily.
- **Iterator.** `hasNext` / `next` interface, many concrete iterators. The `for-each` syntactic sugar in Java/TS hides the polymorphic call.

Generics (parametric polymorphism) shows up most in container patterns: `Repository<T>`, `Cache<K, V>`, `Result<T, E>`, `Optional<T>`. Ad-hoc polymorphism via extensions powers fluent APIs, builders, and collection extensions.

## Common pitfalls

- **Confusing overriding with overloading.** Overriding (subtype, runtime) replaces an inherited method with the same signature. Overloading (ad-hoc, compile time) adds another method with a different signature. They are different mechanisms.
- **Surprise from static dispatch on overloads.** A method `log(Object)` and `log(String)` chosen by the static type of the variable, not the dynamic type — frequently surprising. Either avoid overloading on overlapping types or use `instanceof` inside one method.
- **Generics + arrays in Java.** Java forbids `new T[]` because of erasure. Use `(T[]) new Object[size]` with care, or prefer `ArrayList<T>`.
- **Variance mistakes.** `List<Cat>` is not a `List<Animal>` in most languages, even though `Cat` is an `Animal`. Read up on covariance, contravariance, and invariance — and what `? extends` / `? super` mean in Java, `in` / `out` in Kotlin/C#, `T extends` constraints in TS.
- **Over-using inheritance to enable polymorphism.** Polymorphism through *interfaces* + composition is usually cleaner than through deep inheritance trees.
- **Forgetting that `private` and `static` Java methods are not virtual.** They cannot be overridden. Same for TS class private fields/methods on the prototype chain.

## Related

- [classes-and-objects.md](./classes-and-objects.md) — what gets dispatched on
- [interfaces.md](./interfaces.md) — the most common dispatch surface
- [inheritance.md](./inheritance.md) — overriding lives here
- [abstraction.md](./abstraction.md) — picking the right polymorphic surface
- [enums.md](./enums.md) — enums with per-constant overrides as a tiny polymorphism
- [../../java/INDEX.md](../../java/INDEX.md) — Java generics, virtual dispatch, sealed types
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — TS generics, overload signatures, structural dispatch
