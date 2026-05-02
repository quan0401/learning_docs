---
title: "Classes and Objects"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, oop-fundamentals, java, typescript, semantics]
---

# Classes and Objects

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `oop-fundamentals` `java` `typescript` `semantics`

## Summary

A class is a blueprint that describes both state (fields) and behavior (methods); an object is one runtime instance of that blueprint with its own identity and its own copy of the state. Understanding the difference between a class definition and a live object — and the difference between identity and equality, value and reference semantics — is the prerequisite for every later OOP topic.

## Table of Contents

- [Class definition vs instantiation](#class-definition-vs-instantiation)
- [State vs behavior](#state-vs-behavior)
- [Identity vs equality](#identity-vs-equality)
- [Value vs reference semantics](#value-vs-reference-semantics)
- [Common pitfalls](#common-pitfalls)
- [Related](#related)

## Class definition vs instantiation

A class is a *type* — it does not exist at runtime as data. An object is a runtime entity allocated on the heap (in most managed languages) when you call a constructor.

```java
// Java
public class Account {
    private final String id;
    private long balanceCents;

    public Account(String id, long openingCents) {
        this.id = id;
        this.balanceCents = openingCents;
    }

    public long balanceCents() { return balanceCents; }
    public void deposit(long cents) { this.balanceCents += cents; }
}

Account a = new Account("acc_1", 0);   // instantiation
Account b = new Account("acc_1", 0);   // a different object with the same data
```

```typescript
// TypeScript
class Account {
  private balanceCents: number;
  constructor(public readonly id: string, openingCents: number) {
    this.balanceCents = openingCents;
  }
  getBalanceCents(): number { return this.balanceCents; }
  deposit(cents: number): void { this.balanceCents += cents; }
}

const a = new Account("acc_1", 0);
const b = new Account("acc_1", 0);
```

Key facts:

- The class is loaded once (Java: by the ClassLoader; TS/JS: by the module system).
- Each `new` returns a distinct object with its own field storage.
- Methods are typically shared (Java: in the method area / metaspace; JS: on the prototype) — they are not duplicated per instance.

## State vs behavior

State is data the object remembers between calls; behavior is code that reads or changes that state.

| Concept | What it is | Example |
|---|---|---|
| State | Fields, instance variables | `balanceCents` |
| Behavior | Methods, operations | `deposit(cents)` |
| Identity | Which specific object | The reference itself |

A class with only state and no behavior is essentially a record/struct (DTO). A class with only behavior and no state is essentially a namespace of static functions. The interesting middle is when behavior *guards invariants* over state.

```java
public void withdraw(long cents) {
    if (cents <= 0) throw new IllegalArgumentException("positive only");
    if (cents > balanceCents) throw new IllegalStateException("insufficient");
    balanceCents -= cents;
}
```

That `withdraw` is the whole reason `Account` is a class instead of a `Map<String, Long>`: the method enforces "balance never goes negative" — an invariant the data alone cannot enforce.

## Identity vs equality

Two objects can be *equal in value* but be *different objects*. This trips up almost every beginner.

| Concept | Meaning | Java | TypeScript |
|---|---|---|---|
| Identity | Same object in memory | `a == b` (for refs) | `a === b` |
| Equality | Same logical value | `a.equals(b)` | custom `equals(a,b)` or library |

```java
Account a = new Account("acc_1", 0);
Account b = new Account("acc_1", 0);
System.out.println(a == b);        // false  — different objects
System.out.println(a.equals(b));   // false unless equals() is overridden
```

By default `Object.equals` in Java is reference equality. To get value equality, override `equals` and `hashCode` together (or use a `record`, which auto-generates them).

```java
public record Money(long cents, String currency) {}

Money m1 = new Money(100, "USD");
Money m2 = new Money(100, "USD");
System.out.println(m1.equals(m2)); // true — record gives value equality for free
```

```typescript
const m1 = { cents: 100, currency: "USD" };
const m2 = { cents: 100, currency: "USD" };
m1 === m2;                                     // false
JSON.stringify(m1) === JSON.stringify(m2);     // true (cheap value equality, but fragile)
// Real codebases use lodash isEqual, fast-deep-equal, or a typed equality helper.
```

The general rule: identity is what the runtime gives you for free; equality is what you have to define.

## Value vs reference semantics

This is about how variables and parameters *carry* data.

- **Value semantics**: the variable holds the data itself. Assignment and parameter passing copy the data. Mutating one copy does not affect the other.
- **Reference semantics**: the variable holds a *reference* (pointer/handle) to a heap object. Assignment copies the reference; both names now point to the same object.

| Language | Primitives | Objects |
|---|---|---|
| Java | value (`int`, `long`, `boolean`, `char`, …) | reference (`String`, custom classes, `Integer`) |
| TypeScript / JavaScript | value (`number`, `string`, `boolean`, `bigint`, `symbol`, `undefined`, `null`) | reference (objects, arrays, functions, class instances) |
| C# | value (`struct`, primitives) | reference (`class`) |
| Swift | value (`struct`, `enum`) | reference (`class`) |

```java
// Java — reference semantics for objects
Account x = new Account("acc_1", 100);
Account y = x;            // y points to the same object
y.deposit(50);
System.out.println(x.balanceCents()); // 150 — both names see the change
```

```typescript
// TypeScript — same story for class instances and plain objects
const x = new Account("acc_1", 100);
const y = x;
y.deposit(50);
console.log(x.getBalanceCents()); // 150
```

Strings in Java look like value types but are technically references — they're just immutable, so the distinction rarely matters. `==` on strings still compares references, which is why `equals` exists.

```java
String s1 = new String("hi");
String s2 = new String("hi");
s1 == s2;        // false
s1.equals(s2);   // true
```

### Why this matters for design

If you pass a mutable object to a method, that method can mutate it and the caller will see the change. Three common defenses:

1. Make the class immutable (preferred — see encapsulation doc).
2. Defensive copy on the way in or out (`Collections.unmodifiableList`, `List.copyOf`, `structuredClone` in JS).
3. Document the contract loudly and accept the risk for performance reasons.

## Common pitfalls

- **Confusing class with object.** "The class has a balance of 100" is wrong. *The instance* has a balance of 100. The class itself only has the *concept* of a balance field.
- **Using `==` for value equality in Java.** Always `equals`, except for primitives, enums, and intentional reference checks.
- **Mutating a shared reference.** Two callers holding the same list, one calls `clear()`. Diagnose by asking "who else has a reference to this object?"
- **Assuming `const` in TS gives immutability.** `const` only prevents reassignment of the binding. `const x = []; x.push(1)` is legal. Use `readonly`, `as const`, or `Readonly<T>` for shallow immutability — and a library like Immer for deep.
- **Forgetting that records are not magic.** Java `record` and TS `Readonly<T>` give shallow immutability. A record holding a `List<String>` still exposes a mutable list unless you copy it.

## Related

- [enums.md](./enums.md) — bounded sets as a special kind of class
- [encapsulation.md](./encapsulation.md) — controlling access to state
- [interfaces.md](./interfaces.md) — describing behavior without binding to a specific class
- [inheritance.md](./inheritance.md) — sharing state and behavior across class hierarchies
- [../../java/INDEX.md](../../java/INDEX.md) — Java tier path including records, equality, and value-based classes
- [../../typescript/INDEX.md](../../typescript/INDEX.md) — TypeScript tier path covering structural typing and reference semantics
