---
title: "Refactoring Catalog — Mechanics That Pair With Code Smells"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, refactoring, code-smells, mechanics]
---

# Refactoring Catalog — Mechanics That Pair With Code Smells

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `refactoring` `code-smells` `mechanics`

## Summary

This is the companion to [code-smells-and-refactoring-triggers.md](./code-smells-and-refactoring-triggers.md). The smells doc tells you **when** the design is hurting; this doc tells you **what mechanical move to perform**, drawn from Martin Fowler's *Refactoring: Improving the Design of Existing Code*, 2nd edition.

A refactoring is a *behavior-preserving* transformation. Each entry below follows a fixed shape:

- **Smell that triggers it** (cross-link to the smells doc)
- **Mechanics** — the literal step-by-step
- **Pitfalls** — what trips people up
- **Tiny Java example** — before / after

Run a green test suite before and after every step. The whole point of having a named refactoring is that the steps are small enough to keep tests green between each one.

## Table of Contents

1. Why a catalog instead of "just refactor"
2. Extract Method
3. Inline Method
4. Extract Variable (Introduce Explaining Variable)
5. Rename (Variable / Method / Class)
6. Move Method
7. Move Field
8. Encapsulate Field
9. Replace Magic Number with Symbolic Constant
10. Replace Conditional with Polymorphism
11. Replace Type Code with Subclasses / Strategy
12. Introduce Parameter Object
13. Preserve Whole Object
14. Replace Constructor with Factory Method
15. Pull Up Method
16. Push Down Method
17. Replace Inheritance with Delegation
18. How to use the catalog day-to-day
19. Related
20. References

## 1. Why a catalog instead of "just refactor"

"Refactor" without a name is mood. Fowler's discipline:

- Each move has a name → makes pairing and review precise.
- Each move has *mechanics* → small, reversible, testable steps.
- Each move pairs to a smell → you do not refactor for sport.

The catalog also gives the IDE something to automate. Modern IDEs (IntelliJ IDEA, Eclipse) implement most of these as menu items — use them. The IDE applies the rename across all references in a way grep cannot.

## 2. Extract Method

**Smell:** *Long Method*; *Duplicated Code*; comment that explains a block.

**Mechanics:**

1. Create a new method, name it after *intent*, not implementation.
2. Copy the extracted code into the new method.
3. Pass any locals it reads as parameters; return any local it modifies.
4. Replace the original code with a call to the new method.
5. Run tests.

**Pitfalls:**

- Extracting too small loses context; extracting too big keeps the smell.
- If the block reads or writes many locals, consider Introduce Parameter Object first.

```java
// Before
void printOwing(Invoice invoice) {
    System.out.println("***********************");
    System.out.println("**** Customer Owes ****");
    System.out.println("***********************");

    double outstanding = 0.0;
    for (Order o : invoice.getOrders()) outstanding += o.getAmount();

    System.out.println("name: " + invoice.getCustomer());
    System.out.println("amount: " + outstanding);
}

// After
void printOwing(Invoice invoice) {
    printBanner();
    double outstanding = calculateOutstanding(invoice);
    printDetails(invoice, outstanding);
}
```

## 3. Inline Method

**Smell:** Method body is as clear as its name; needless indirection from over-eager extraction.

**Mechanics:**

1. Confirm the method is not polymorphic (no overrides).
2. Find every caller; replace each call with the body.
3. Delete the method.
4. Run tests.

**Pitfalls:**

- Do not inline through an interface or override boundary.
- If the body has side effects, inline carefully — caller order may change.

## 4. Extract Variable (Introduce Explaining Variable)

**Smell:** A complex expression in the middle of a larger statement.

**Mechanics:**

1. Declare a `final` (or `val`) local with a name that explains the expression.
2. Replace the expression with the variable.
3. Run tests.

```java
// Before
if ((platform.toLowerCase().contains("mac")) &&
    (browser.toLowerCase().contains("ie")) &&
    wasInitialized() && resize > 0) { ... }

// After
final boolean isMacOs    = platform.toLowerCase().contains("mac");
final boolean isIE       = browser.toLowerCase().contains("ie");
final boolean wasResized = resize > 0;

if (isMacOs && isIE && wasInitialized() && wasResized) { ... }
```

**Pitfall:** Naming. A bad variable name is worse than the original expression — it lies.

## 5. Rename (Variable / Method / Class)

**Smell:** Identifier no longer matches what the thing does.

**Mechanics:** *Always* use the IDE rename refactoring — it updates every reference, including reflective string usages your grep would miss only if your IDE knows about them.

**Pitfalls:**

- Public API renames break consumers — use deprecation, not just rename.
- Names persisted to a database, message broker, or serialized payload have wider blast radius. Treat them as public API.

## 6. Move Method

**Smell:** *Feature Envy* — a method on class `A` accesses class `B`'s data more than its own.

**Mechanics:**

1. Examine the method's dependencies in `A` and `B`.
2. Create the new method on `B` with an appropriate signature.
3. Copy the body, adjust references.
4. In `A`, replace the original with a delegating call (or inline the call site directly).
5. Run tests; remove the delegate when ready.

```java
// Before — Account computes overdraft using Account-Type fields
class Account {
    private AccountType type;
    private int daysOverdrawn;
    double overdraftCharge() {
        if (type.isPremium()) {
            double result = 10;
            if (daysOverdrawn > 7) result += (daysOverdrawn - 7) * 0.85;
            return result;
        }
        return daysOverdrawn * 1.75;
    }
}

// After — moved to AccountType, where the rule belongs
class AccountType {
    double overdraftCharge(int daysOverdrawn) { ... }
}
class Account {
    double overdraftCharge() { return type.overdraftCharge(daysOverdrawn); }
}
```

## 7. Move Field

**Smell:** A field on `A` is consistently used by `B`, or only makes sense in `B`'s lifecycle.

**Mechanics:**

1. Encapsulate the field on `A` (getter + setter) if not already.
2. Add the field on `B`, with accessors.
3. Update `A`'s accessors to delegate to `B`.
4. Migrate callers off `A`'s accessors.
5. Remove the field from `A`.

**Pitfall:** With JPA-mapped fields, mind the database schema — a field move usually implies a schema migration.

## 8. Encapsulate Field

**Smell:** A `public` field that should not be directly mutable; logic that needs to fire on every read or write.

**Mechanics:**

1. Make the field `private`.
2. Provide a getter (and setter only if mutation is part of the contract).
3. Update all callers to use the accessors.
4. Run tests.

```java
// Before
public class Person { public String name; }

// After
public class Person {
    private String name;
    public String getName() { return name; }
    public void setName(String name) { this.name = Objects.requireNonNull(name); }
}
```

**Pitfall:** Encapsulation is *not* "add a getter and setter for every field." Add accessors only where they earn their keep. Records express true value semantics more cheaply.

## 9. Replace Magic Number with Symbolic Constant

**Smell:** A literal whose meaning is not obvious from context (`* 9.81`, `if (status == 3)`).

**Mechanics:**

1. Declare a constant with a meaningful name.
2. Replace every occurrence of the literal with the constant.
3. Run tests.

```java
// Before
double potentialEnergy(double mass, double height) { return mass * 9.81 * height; }

// After
private static final double STANDARD_GRAVITY = 9.81;
double potentialEnergy(double mass, double height) { return mass * STANDARD_GRAVITY * height; }
```

**Pitfall:** Resist false constants — `0` and `1` in loops or `"-"` as a separator are usually clearer left as literals.

## 10. Replace Conditional with Polymorphism

**Smell:** *Switch Statement* / repeated `if/else` chains keyed on a type.

**Mechanics:**

1. Identify the type code that drives the conditional.
2. Create a base interface (or sealed interface) with a method per branch.
3. Create a subtype per case; move that branch's body into the subtype.
4. Replace the conditional with a call to the polymorphic method.

```java
// Before
double tax(Item item) {
    return switch (item.getType()) {
        case BOOK    -> item.price() * 0.0;
        case ALCOHOL -> item.price() * 0.20;
        default      -> item.price() * 0.10;
    };
}

// After
sealed interface Item permits Book, Alcohol, GeneralGood {
    Money price();
    Money tax();
}
record Book(Money price)        implements Item { public Money tax() { return Money.ZERO; } }
record Alcohol(Money price)     implements Item { public Money tax() { return price.times(0.20); } }
record GeneralGood(Money price) implements Item { public Money tax() { return price.times(0.10); } }
```

**Pitfall:** Polymorphism is overkill if the branches are stable and few. Keep the `switch` if the alternatives never grow.

## 11. Replace Type Code with Subclasses / Strategy

**Smell:** A `type` enum field whose value drives behavior in many methods.

**Mechanics:**

- **Subclasses** when the type is intrinsic and immutable (a book is always a book).
- **Strategy** when the behavior must change at runtime, or one object can change its "kind" over time.

```java
// Strategy variant
interface PricingStrategy { Money price(Order o); }
class StandardPricing implements PricingStrategy { ... }
class PromotionPricing implements PricingStrategy { ... }

class Order {
    private PricingStrategy pricing;
    void switchPricing(PricingStrategy newPricing) { this.pricing = newPricing; }
    Money total() { return pricing.price(this); }
}
```

**Pitfall:** Sealed types in Java 17+ blur the line. If you can model the cases as a closed set, sealed interfaces give you exhaustiveness checking without manually wiring a Strategy.

## 12. Introduce Parameter Object

**Smell:** *Long Parameter List*; the same group of parameters travels together across many methods (*Data Clump*).

**Mechanics:**

1. Create a class (or record) bundling the parameter group.
2. Replace the parameters with the new type, one method at a time.
3. Move related behavior onto the new type as opportunities appear.

```java
// Before
List<Reading> readingsInRange(LocalDate start, LocalDate end) { ... }

// After
record DateRange(LocalDate start, LocalDate end) {
    public DateRange {
        if (end.isBefore(start)) throw new IllegalArgumentException("end before start");
    }
    public boolean contains(LocalDate d) { return !d.isBefore(start) && !d.isAfter(end); }
}
List<Reading> readingsInRange(DateRange range) { ... }
```

**Pitfall:** Do not create one parameter object per method — find the groupings that recur.

## 13. Preserve Whole Object

**Smell:** A caller pulls several values out of an object and passes them individually.

**Mechanics:**

1. Change the method signature to take the whole object.
2. Update the body to access fields via the object.
3. Update callers.

```java
// Before
boolean withinPlan(int low, int high, HeatingPlan plan) {
    return plan.getMin() <= low && plan.getMax() >= high;
}
// caller: withinPlan(room.getLow(), room.getHigh(), plan);

// After
boolean withinPlan(Room room, HeatingPlan plan) {
    return plan.getMin() <= room.getLow() && plan.getMax() >= room.getHigh();
}
```

**Pitfall:** This couples the method to the whole object. If the method only ever needs two fields and you have no other use for the object, leave the parameters atomic.

## 14. Replace Constructor with Factory Method

**Smell:** Construction logic dispatches on type; constructors with confusing overloads; you want a *named* construction operation.

**Mechanics:**

1. Add a `static` factory method that calls the constructor.
2. Migrate callers to the factory method.
3. Make the constructor `private` (or package-private).

```java
class Employee {
    private final EmployeeType type;
    private Employee(EmployeeType type) { this.type = type; }

    public static Employee asEngineer() { return new Employee(EmployeeType.ENGINEER); }
    public static Employee asManager()  { return new Employee(EmployeeType.MANAGER);  }
}
```

**Pitfall:** Static factories cannot be overridden. If subclassing is part of the design, prefer Factory Method (instance-level) or Abstract Factory.

## 15. Pull Up Method

**Smell:** Two or more subclasses contain identical (or near-identical) methods.

**Mechanics:**

1. Verify the method bodies are functionally identical (parameters, return type, behavior).
2. Move the method to the superclass.
3. Delete from subclasses.
4. Run tests.

**Pitfall:** Beware of "identical-looking" methods that depend on subclass-specific fields. Pull up only the parts that truly belong to the superclass; the rest may need to become abstract methods invoked by the pulled-up method (Form Template Method).

## 16. Push Down Method

**Smell:** A method on the superclass is used by only one subclass.

**Mechanics:**

1. Move the method down to the subclass that uses it.
2. Delete from the superclass.
3. Run tests.

**Pitfall:** If multiple subclasses share the method, push down means duplicating it. Check usage carefully before pushing.

## 17. Replace Inheritance with Delegation

**Smell:** A subclass uses only a fraction of the superclass's interface, or inheritance was chosen for code reuse rather than for true subtype semantics.

**Mechanics:**

1. Add a field to the would-be subclass referencing an instance of the would-be superclass.
2. Replace `super.x()` calls with `delegate.x()`.
3. Stop extending the superclass; expose only the methods that genuinely belong.
4. Run tests.

```java
// Before
class Stack<E> extends ArrayList<E> {
    public void push(E e) { add(e); }
    public E pop() { return remove(size() - 1); }
}

// After — delegation, hiding the irrelevant ArrayList API
class Stack<E> {
    private final List<E> elements = new ArrayList<>();
    public void push(E e) { elements.add(e); }
    public E pop()         { return elements.remove(elements.size() - 1); }
    public int size()      { return elements.size(); }
}
```

**Pitfall:** Delegation costs boilerplate. If the subclass relationship is genuinely "is-a" and the full superclass interface is appropriate, leave it alone. See [Composing Objects Principle](./composing-objects-principle.md) for when each wins.

## 18. How to use the catalog day-to-day

A practical loop:

1. **Spot the smell** (use the smells doc as a checklist).
2. **Pick the named refactoring** that addresses it.
3. **Run tests.** Green? Continue. Red? Stop and fix first.
4. **Apply the mechanics in small steps.** Commit after each green step if the change is large.
5. **Re-evaluate.** Did the smell move elsewhere? That is normal — refactoring often surfaces deeper structure.

A pattern you will see repeatedly: Extract Method → Move Method → Extract Class. Long methods get sliced; the slices reveal that some belong to a different class; the new class emerges. This is the slow, safe path from a procedural mess to an object-oriented model.

### Tiny TS analogue

Most refactorings transfer straight to TypeScript. Naming conventions and IDE support differ, but the mechanics are identical:

```ts
// Replace Magic Number — TS
const STANDARD_GRAVITY = 9.81;
const potentialEnergy = (mass: number, height: number) => mass * STANDARD_GRAVITY * height;

// Introduce Parameter Object — TS
type DateRange = { start: Date; end: Date };
const readingsInRange = (range: DateRange): Reading[] => { /* ... */ };
```

The discipline — small steps, green tests, named moves — is what matters, not the language.

## Related

- [Code Smells and Refactoring Triggers](./code-smells-and-refactoring-triggers.md) (paired companion)
- [GRASP Principles](./grasp-principles.md)
- [DDD Tactical Patterns](./ddd-tactical-patterns.md)
- [SOLID — Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [SOLID — Open/Closed Principle](../solid/open-closed-principle.md)
- [SOLID — Liskov Substitution Principle](../solid/liskov-substitution-principle.md)
- [Coupling and Cohesion](./coupling-and-cohesion.md)
- [Composing Objects Principle](./composing-objects-principle.md)
- [Separation of Concerns](./separation-of-concerns.md)
- [DRY Principle](./dry-principle.md)
- [KISS Principle](./kiss-principle.md)
- [Anti-Patterns in OO Design](./anti-patterns-in-oo-design.md)
- [OOP Fundamentals — Encapsulation](../oop-fundamentals/encapsulation.md)
- [OOP Fundamentals — Inheritance vs Composition](../design-principles/composing-objects-principle.md)
- [Design Patterns — Strategy](../design-patterns/behavioral/strategy.md)
- [Design Patterns — Factory Method](../design-patterns/creational/factory-method.md)

## References

- Martin Fowler — *Refactoring: Improving the Design of Existing Code*, 2nd edition
- Martin Fowler — *Patterns of Enterprise Application Architecture*
- Joshua Kerievsky — *Refactoring to Patterns*
- Michael Feathers — *Working Effectively with Legacy Code*
- Kent Beck — *Tidy First?*
- Robert C. Martin — *Clean Code: A Handbook of Agile Software Craftsmanship*
