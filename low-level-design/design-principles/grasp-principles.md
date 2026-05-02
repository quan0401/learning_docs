---
title: "GRASP — General Responsibility Assignment Software Patterns"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, grasp, responsibility-assignment, oop]
---

# GRASP — General Responsibility Assignment Software Patterns

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `grasp` `responsibility-assignment` `oop`

## Summary

GRASP — **General Responsibility Assignment Software Patterns** — is a set of nine principles introduced by Craig Larman in *Applying UML and Patterns*. Where SOLID describes properties a good design *has*, GRASP gives you a vocabulary for **deciding which class should hold which responsibility** in the first place. SOLID says "depend on abstractions"; GRASP says "this method belongs on the class that owns the data it needs."

In an LLD interview, GRASP is the engine running quietly when a strong candidate confidently says "the `Order` knows how to compute its total" or "we need an `OrderProcessor` because no domain class is the natural home for this workflow." Memorize the nine names, the question each one answers, and the smell each one prevents.

## Table of Contents

1. The nine principles at a glance
2. Information Expert
3. Creator
4. Controller
5. Low Coupling
6. High Cohesion
7. Polymorphism
8. Pure Fabrication
9. Indirection
10. Protected Variations
11. GRASP versus SOLID — how they fit together
12. Mapping GRASP to common LLD interview decisions
13. Related
14. References

## 1. The nine principles at a glance

| Principle | Question it answers |
|---|---|
| Information Expert | Who has the data needed to fulfill this responsibility? |
| Creator | Who should instantiate object `X`? |
| Controller | What non-UI object handles a system event? |
| Low Coupling | How do we minimize dependencies between classes? |
| High Cohesion | How do we keep each class focused on one job? |
| Polymorphism | How do we handle alternatives based on type without `if/else` chains? |
| Pure Fabrication | What if no domain class is a natural home? |
| Indirection | How do we decouple two classes that should not know about each other? |
| Protected Variations | How do we shield stable code from points of predicted change? |

## 2. Information Expert

**Assign a responsibility to the class that has the information needed to fulfill it.**

This is the GRASP principle you will use most. If a method needs three fields off `Order`, the method belongs on `Order` — not on a separate `OrderCalculator` floating outside the model.

```java
public final class Order {
    private final List<LineItem> items;

    public Order(List<LineItem> items) {
        this.items = List.copyOf(items);
    }

    // Information Expert: Order has the items, so Order computes the total.
    public Money total() {
        return items.stream()
            .map(LineItem::subtotal)
            .reduce(Money.ZERO, Money::plus);
    }
}
```

**Smell it prevents:** *Anemic Domain Model* — data on one class, behavior on another, with everything coordinated by a fat service.

**Caveat:** Information Expert pulls toward putting behavior on entities. When that conflicts with Low Coupling (e.g. you would have to pull a persistence dependency into a domain object), prefer a Pure Fabrication.

## 3. Creator

**Class `B` should create instances of class `A` if one or more of these holds:**

- B aggregates A objects.
- B contains A objects.
- B records instances of A.
- B closely uses A.
- B has the initializing data needed to construct A.

```java
public final class Order {
    private final List<LineItem> items = new ArrayList<>();

    // Order aggregates LineItems and has the data to make one — Order is the Creator.
    public LineItem addItem(Product product, int quantity) {
        LineItem item = new LineItem(product, quantity);
        items.add(item);
        return item;
    }
}
```

**Smell it prevents:** Random `new` calls scattered across services and controllers, none of which own the lifecycle.

**Tie-in:** When the construction logic itself is non-trivial, Creator naturally evolves into a *Factory Method* or *Abstract Factory*.

## 4. Controller

**Assign the responsibility for handling a system event to a non-UI class that represents either the overall system, a use case, or a "root" object.**

The Controller in GRASP is *not* the MVC controller — it is the seam between user-interface or transport-layer code and your domain model.

```java
@RestController
@RequestMapping("/orders")
public final class OrderRestAdapter {
    private final PlaceOrderUseCase placeOrder; // ← GRASP Controller

    public OrderRestAdapter(PlaceOrderUseCase placeOrder) {
        this.placeOrder = placeOrder;
    }

    @PostMapping
    public OrderResponse place(@RequestBody PlaceOrderRequest req) {
        var orderId = placeOrder.handle(req.toCommand());
        return OrderResponse.of(orderId);
    }
}

public final class PlaceOrderUseCase {
    public OrderId handle(PlaceOrderCommand cmd) { /* ... */ }
}
```

**Smell it prevents:** *Bloated Controller* — REST adapters or UI handlers that contain branching domain logic and direct repository calls.

## 5. Low Coupling

**Assign responsibilities so that coupling between classes remains low.**

Low Coupling is rarely the answer to "where does this go?" by itself — it is the *constraint* that breaks ties between competing assignments. If two designs both satisfy Information Expert, pick the one that introduces fewer cross-package dependencies.

```java
// HIGH coupling — domain knows JDBC.
public final class Customer {
    public void save(Connection conn) { /* ... */ }
}

// LOW coupling — domain knows nothing about persistence.
public final class Customer { /* pure data + behavior */ }
public interface CustomerRepository { void save(Customer c); }
```

See `coupling-and-cohesion.md` for the full taxonomy (content, common, control, stamp, data).

## 6. High Cohesion

**Each class should have a focused, related set of responsibilities.**

Cohesion is the inverse symptom of "this class is hard to name." If you cannot describe the class in one sentence without using "and," it is doing too much.

```java
// LOW cohesion — three unrelated jobs.
class OrderManager {
    void placeOrder(Order o) { ... }
    void sendOrderEmail(Order o) { ... }
    void exportOrdersToCsv(List<Order> os) { ... }
}

// HIGH cohesion — one job each.
class OrderPlacer { ... }
class OrderConfirmationMailer { ... }
class OrderCsvExporter { ... }
```

High Cohesion and Low Coupling are *partners*: cohesive classes naturally couple loosely because each touches a small slice of the world.

## 7. Polymorphism

**When alternatives vary by type, assign the responsibility using polymorphism rather than explicit type-checking.**

```java
// Type-checking — fragile, must edit every time a new type appears.
double tax(Item item) {
    if (item instanceof Book b)        return b.price() * 0.0;
    else if (item instanceof Alcohol a) return a.price() * 0.20;
    else                                return item.price() * 0.10;
}

// Polymorphism — each type owns its rule.
sealed interface Item permits Book, Alcohol, Other {
    Money price();
    default Money tax() { return price().times(taxRate()); }
    BigDecimal taxRate();
}
```

Sealed types in Java 17+ let the compiler enforce exhaustiveness, giving you Polymorphism without losing the safety of pattern matching.

## 8. Pure Fabrication

**When no domain class is a sensible home, invent a class whose sole purpose is to hold the responsibility.**

Pure Fabrications are *not* domain concepts. `OrderRepository`, `PasswordHasher`, `JwtIssuer`, `ShippingCostCalculator` — none of these exist in the business glossary, but each gives you a clean home for behavior that would otherwise pollute domain entities.

```java
public interface OrderRepository {  // Pure Fabrication
    Optional<Order> findById(OrderId id);
    void save(Order order);
}
```

**When to fabricate:** Information Expert says "put it on `Order`," but doing so would force `Order` to know about JDBC, HTTP, or message brokers. Fabricate instead.

## 9. Indirection

**Assign responsibility to an intermediate object to mediate between two components, decoupling them.**

Indirection is the principle behind Mediator, Adapter, Facade, and most of the patterns where "object C exists so A and B do not have to talk."

```java
public interface PaymentGateway { Receipt charge(Money amount, Card card); }

// Stripe-specific code lives behind the indirection.
public final class StripePaymentGateway implements PaymentGateway { /* ... */ }
```

The order-processing code talks to `PaymentGateway`, never to Stripe SDK classes. Swapping providers becomes a one-class change.

**Cost:** every layer of indirection is a layer of mental overhead. Add it when the seam pays for itself, not as decoration.

## 10. Protected Variations

**Identify points of predicted variation or instability; assign responsibilities to create a stable interface around them.**

This is the umbrella principle — Polymorphism, Indirection, and most of SOLID's Open/Closed Principle are concrete tactics for achieving Protected Variations.

```java
// The variation point: how we hash passwords may change (bcrypt → argon2 → ...).
public interface PasswordHasher {
    String hash(String plaintext);
    boolean verify(String plaintext, String hashed);
}
```

Application code depends on `PasswordHasher`. The hashing algorithm becomes a contained, swappable detail.

**Heuristic:** Protect against variations you can *name and predict*. Wrapping every dependency "just in case" is YAGNI in disguise.

## 11. GRASP versus SOLID — how they fit together

| | GRASP | SOLID |
|---|---|---|
| Layer | How to **assign** responsibilities | How to **shape** classes once assigned |
| Granularity | Object-by-object decision making | Class-level structural properties |
| Origin | Larman, *Applying UML and Patterns* | Robert C. Martin, late 1990s |
| Typical question | "Which class should own this method?" | "Is this class doing too much?" |

A complete LLD answer usually weaves both: "By Information Expert, `Order` owns `total()` (GRASP). The `Order` class has a single responsibility — representing an order — and the calculation does not change for new payment types, so it satisfies SRP and OCP (SOLID)."

## 12. Mapping GRASP to common LLD interview decisions

| Interview moment | GRASP principle to invoke |
|---|---|
| "Where does the business rule live?" | Information Expert |
| "Who creates the `Reservation`?" | Creator |
| "How does the REST endpoint stay thin?" | Controller |
| "Why a `RateLimiter` class instead of inline checks?" | Pure Fabrication + High Cohesion |
| "Why an interface for the payment provider?" | Indirection + Protected Variations |
| "How do you avoid `if (vehicle.type == ...)`?" | Polymorphism |
| "Why split this 600-line service?" | High Cohesion |
| "How do you keep the domain model free of Spring annotations?" | Low Coupling + Protected Variations |

### A worked LLD vignette

> *"Design the checkout flow for an e-commerce service."*

A GRASP-shaped answer in three breaths:

1. **Information Expert + Creator:** `Cart` owns `LineItem`s and computes its own `subtotal()`. `Cart` creates `LineItem`s when products are added.
2. **Controller + Pure Fabrication:** `CheckoutUseCase` (Pure Fabrication, no domain analog) is the GRASP Controller for the "place order" system event. The REST adapter delegates to it.
3. **Indirection + Protected Variations + Polymorphism:** `PaymentGateway` is the indirection between the use case and Stripe/Adyen/PayPal implementations. `TaxStrategy` uses Polymorphism so adding a new jurisdiction does not edit existing code.

That paragraph alone signals senior-level thinking in an interview.

## Related

- [SOLID — Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [SOLID — Open/Closed Principle](../solid/open-closed-principle.md)
- [SOLID — Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [Coupling and Cohesion](./coupling-and-cohesion.md)
- [Separation of Concerns](./separation-of-concerns.md)
- [Composing Objects Principle](./composing-objects-principle.md)
- [Law of Demeter](./law-of-demeter.md)
- [Anti-Patterns in OO Design](./anti-patterns-in-oo-design.md)
- [Code Smells and Refactoring Triggers](./code-smells-and-refactoring-triggers.md)
- [OOP Fundamentals — Encapsulation](../oop-fundamentals/encapsulation.md)
- [OOP Fundamentals — Polymorphism](../oop-fundamentals/polymorphism.md)
- [Design Patterns — Strategy](../design-patterns/behavioral/strategy.md)
- [Design Patterns — Factory Method](../design-patterns/creational/factory-method.md)
- [Design Patterns — Repository](../design-patterns/additional/repository-pattern.md)

## References

- Craig Larman — *Applying UML and Patterns: An Introduction to Object-Oriented Analysis and Design and Iterative Development*, 3rd edition
- Robert C. Martin — *Agile Software Development, Principles, Patterns, and Practices*
- Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software*
- Martin Fowler — *Patterns of Enterprise Application Architecture*
