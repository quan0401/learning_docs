---
title: "Domain-Driven Design — Tactical Patterns"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, ddd, domain-modeling, aggregates]
---

# Domain-Driven Design — Tactical Patterns

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `ddd` `domain-modeling` `aggregates`

## Summary

Domain-Driven Design (DDD), introduced by Eric Evans in *Domain-Driven Design: Tackling Complexity in the Heart of Software* and made operational by Vaughn Vernon in *Implementing Domain-Driven Design*, splits cleanly into two layers:

- **Strategic DDD** — Bounded Contexts, Ubiquitous Language, Context Maps. Concerned with *where domain models live and how they relate*.
- **Tactical DDD** — Entities, Value Objects, Aggregates, Domain Events, Domain Services, Application Services, Factories, Repositories. Concerned with *how to express a model in code*.

This document focuses on the tactical layer with brief framing for the strategic side. The tactical patterns are what you actually type into a Java file: they are the building blocks for any object-oriented codebase that wants to keep business rules from being smeared across services and DTOs.

## Table of Contents

1. Strategic vs Tactical — the layer split
2. Entity — identity over time
3. Value Object — equality by value
4. Aggregate and Aggregate Root — the consistency boundary
5. Domain Event — facts the business cares about
6. Domain Service — behavior that does not belong to a single entity
7. Application Service — orchestration, no rules
8. Factory in DDD context
9. Repository — collection-like access to aggregates
10. Bounded Context (briefly)
11. JPA / Spring realities and trade-offs
12. Anti-patterns to avoid
13. Related
14. References

## 1. Strategic vs Tactical — the layer split

| Layer | Concerns | Artifacts |
|---|---|---|
| Strategic | Decomposing a large system; team boundaries; integration | Bounded Context, Ubiquitous Language, Context Map (Shared Kernel, Customer/Supplier, Anti-Corruption Layer, Open Host Service) |
| Tactical | Expressing a single model in code | Entity, Value Object, Aggregate, Domain Event, Domain/Application Service, Factory, Repository |

You cannot do tactical DDD well without at least *some* strategic clarity. Two `Order` entities in `billing` and `fulfillment` are not the same `Order` — they live in different bounded contexts and may have different fields, invariants, and lifecycles.

## 2. Entity — identity over time

An **Entity** is an object whose identity persists across state changes. Two `Customer` instances with the same name are not the same customer; identity is what distinguishes them.

```java
public final class Customer {
    private final CustomerId id;          // identity
    private CustomerName name;            // mutable state
    private EmailAddress email;

    public Customer(CustomerId id, CustomerName name, EmailAddress email) {
        this.id = Objects.requireNonNull(id);
        this.name = Objects.requireNonNull(name);
        this.email = Objects.requireNonNull(email);
    }

    public void rename(CustomerName newName) { this.name = newName; }

    @Override public boolean equals(Object o) {
        return o instanceof Customer c && id.equals(c.id);
    }
    @Override public int hashCode() { return id.hashCode(); }
}
```

**Rules of thumb:**

- Equality and hashCode use **identity only**, never mutable fields.
- Identity is assigned at creation, never changes, never reused.
- Prefer typed identifiers (`CustomerId`) over raw `Long` / `UUID`.

## 3. Value Object — equality by value

A **Value Object** has no conceptual identity. Two `Money(100, USD)` instances are interchangeable. Value Objects are immutable, side-effect-free, and compared structurally.

Java records are a near-perfect fit:

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException("scale exceeds currency precision");
        }
    }
    public Money plus(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
        return new Money(amount.add(other.amount), currency);
    }
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.getInstance("USD"));
}
```

**Why this matters:** Value Objects make invalid states unrepresentable. `Email`, `PhoneNumber`, `PostalAddress`, `DateRange`, `PercentageBetweenZeroAndOne` — once they exist, the rest of the codebase stops re-validating them.

## 4. Aggregate and Aggregate Root — the consistency boundary

An **Aggregate** is a cluster of Entities and Value Objects treated as a single unit for changes and invariants. The **Aggregate Root** is the only entry point — outside code may hold references only to the root, never to internal members.

```java
public final class Order {                       // Aggregate Root
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderLine> lines = new ArrayList<>();   // internal entity
    private OrderStatus status = OrderStatus.DRAFT;

    public void addLine(ProductId productId, int qty, Money unitPrice) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("cannot modify a placed order");
        }
        lines.add(new OrderLine(productId, qty, unitPrice));
    }

    public void place() {
        if (lines.isEmpty()) throw new IllegalStateException("empty order");
        this.status = OrderStatus.PLACED;
    }

    public Money total() {
        return lines.stream().map(OrderLine::subtotal).reduce(Money.ZERO, Money::plus);
    }

    public List<OrderLine> lines() { return List.copyOf(lines); }  // defensive copy
}
```

**The four aggregate design rules from Vernon's *Implementing DDD*:**

1. Model true invariants in consistency boundaries.
2. Design small aggregates.
3. Reference other aggregates by identity only (`CustomerId`, not `Customer`).
4. Use eventual consistency outside the boundary.

A common beginner mistake is making `Order` reference a full `Customer` object. Don't. Reference `CustomerId`. Each aggregate is its own transaction.

## 5. Domain Event — facts the business cares about

A **Domain Event** records something meaningful that happened in the domain. Past tense, immutable, named in the Ubiquitous Language.

```java
public sealed interface OrderEvent permits OrderPlaced, OrderShipped, OrderCancelled {
    OrderId orderId();
    Instant occurredAt();
}

public record OrderPlaced(OrderId orderId, CustomerId customerId,
                          Money total, Instant occurredAt) implements OrderEvent {}
```

Events are emitted *from inside the aggregate* and consumed by application-level handlers (eventual consistency to other aggregates, integration with messaging, projections to read models).

```java
public final class Order {
    private final List<OrderEvent> uncommittedEvents = new ArrayList<>();

    public void place() {
        // ... invariants ...
        this.status = OrderStatus.PLACED;
        uncommittedEvents.add(new OrderPlaced(id, customerId, total(), Instant.now()));
    }

    public List<OrderEvent> pullEvents() {
        var snapshot = List.copyOf(uncommittedEvents);
        uncommittedEvents.clear();
        return snapshot;
    }
}
```

## 6. Domain Service — behavior that does not belong to a single entity

A **Domain Service** holds domain logic that genuinely spans multiple aggregates or does not naturally belong to any one of them. The keyword is *domain* — it speaks the Ubiquitous Language and contains business rules, not infrastructure.

```java
public final class FundsTransferService {       // Domain Service
    public void transfer(Account from, Account to, Money amount) {
        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

If a "service" is mostly orchestrating repositories and emitting messages, it is an Application Service, not a Domain Service.

## 7. Application Service — orchestration, no rules

An **Application Service** is the use case layer. It coordinates: load aggregate(s), invoke domain behavior, persist, publish events. It contains **no business rules** — those live in the domain.

```java
@Service
public final class PlaceOrderApplicationService {
    private final OrderRepository orders;
    private final DomainEventPublisher publisher;

    public PlaceOrderApplicationService(OrderRepository orders, DomainEventPublisher publisher) {
        this.orders = orders;
        this.publisher = publisher;
    }

    @Transactional
    public OrderId handle(PlaceOrderCommand cmd) {
        Order order = orders.findById(cmd.orderId())
            .orElseThrow(() -> new OrderNotFoundException(cmd.orderId()));
        order.place();                          // ← domain logic
        orders.save(order);
        order.pullEvents().forEach(publisher::publish);
        return order.id();
    }
}
```

If you find an `if` that encodes a business rule inside an Application Service, push it into the aggregate.

## 8. Factory in DDD context

Use a Factory when **constructing an aggregate is itself a domain operation** — it has invariants, dispatches on type, or is non-trivial enough that a direct constructor would obscure intent.

```java
public final class OrderFactory {
    public Order createForCustomer(CustomerId customerId, List<DraftLine> draft) {
        if (draft.isEmpty()) throw new IllegalArgumentException("empty draft");
        var order = new Order(OrderId.newId(), customerId);
        for (DraftLine d : draft) {
            order.addLine(d.productId(), d.quantity(), d.unitPrice());
        }
        return order;
    }
}
```

Cross-link: this is the [Factory Method pattern](../design-patterns/creational/factory-method.md) used inside the domain layer rather than as a generic creational tool.

## 9. Repository — collection-like access to aggregates

A **Repository** mediates between the domain and persistence. From the domain's perspective it looks like an in-memory collection of aggregates; the implementation hides JDBC / JPA / Mongo.

```java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
    List<Order> findActiveForCustomer(CustomerId customerId);
}
```

Critical rules:

- One repository **per aggregate root**, never per entity inside an aggregate.
- Methods speak the Ubiquitous Language (`findActiveForCustomer`), not SQL (`selectWhereStatusIn`).
- Implementation lives in the infrastructure layer.

Cross-link: see [Repository Pattern](../design-patterns/additional/repository-pattern.md) for the pattern in its broader form, including non-DDD uses.

## 10. Bounded Context (briefly)

A **Bounded Context** is the explicit boundary within which a model applies. The same word — `Customer`, `Product`, `Order` — can mean different things in `Sales`, `Billing`, and `Fulfillment` contexts; trying to merge them into one canonical model is a recipe for an unmaintainable god-schema.

Strategic patterns to know by name:

- **Ubiquitous Language** — domain experts and developers share one vocabulary inside a bounded context.
- **Context Map** — explicit diagram of how bounded contexts integrate.
- **Anti-Corruption Layer (ACL)** — translation layer protecting your model from a foreign one.
- **Shared Kernel, Customer/Supplier, Conformist, Open Host Service, Published Language** — relationship styles between contexts.

These are the *strategic* layer; treat them as the frame around the tactical patterns above.

## 11. JPA / Spring realities and trade-offs

Pure DDD prefers persistence-ignorant domain objects. The Java ecosystem makes this awkward. Pragmatic compromises:

| Tension | Pragmatic resolution |
|---|---|
| JPA needs no-arg constructors | `protected` no-arg constructor; document it as JPA-only |
| `@Entity` annotations on domain classes | Acceptable; treat them as a leaky abstraction you have chosen to live with |
| Records cannot be JPA entities (yet) | Use records for Value Objects (often as `@Embeddable`); use classes for Entities |
| Lazy loading collections inside aggregates | Configure carefully or use explicit fetch repositories — do not let `LazyInitializationException` define your boundaries |
| Spring's `@Service`/`@Transactional` on Application Services | Standard; keep them out of the domain layer |
| `JpaRepository` as your DDD repository | Acceptable for simple cases; for richer domains, define your own repository interface and have the JPA implementation depend on `JpaRepository` internally |

A typical Spring layout:

```
order/
├── domain/                  ← no Spring, no JPA imports if possible
│   ├── Order.java
│   ├── OrderLine.java
│   ├── OrderId.java          (record)
│   ├── OrderRepository.java  (interface)
│   └── events/
│       └── OrderPlaced.java  (record)
├── application/
│   └── PlaceOrderApplicationService.java
└── infrastructure/
    ├── JpaOrderRepository.java
    └── persistence/
        └── OrderJpaEntity.java   (separate from domain Order if mismatch is large)
```

When the JPA mapping requirements diverge from the domain shape, split: keep a clean `Order` aggregate and translate to/from a separate `OrderJpaEntity`. Pay the translation tax to keep the domain pure.

## 12. Anti-patterns to avoid

- **Anemic Domain Model** — entities are bags of getters/setters; all behavior lives in services. Defeats the entire point of DDD.
- **Aggregate-as-graph** — one giant `Order` aggregate that transitively loads the entire database.
- **Cross-aggregate object references** — holding `Customer` inside `Order` instead of `CustomerId` couples lifecycles and breaks the consistency boundary.
- **Repository per entity** — repositories proliferate; aggregate boundaries dissolve.
- **Business logic in Application Services** — the use case layer ends up with `if/else` chains the domain should own.
- **Generic Subdomain treated as Core** — investing modeling effort in commodity concerns (auth, file upload) instead of the domain that gives the business its competitive edge.

## Related

- [SOLID — Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [SOLID — Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [GRASP Principles](./grasp-principles.md)
- [Coupling and Cohesion](./coupling-and-cohesion.md)
- [Separation of Concerns](./separation-of-concerns.md)
- [Composing Objects Principle](./composing-objects-principle.md)
- [Anti-Patterns in OO Design](./anti-patterns-in-oo-design.md)
- [OOP Fundamentals — Encapsulation](../oop-fundamentals/encapsulation.md)
- [OOP Fundamentals — Inheritance vs Composition](../design-principles/composing-objects-principle.md)
- [Design Patterns — Repository](../design-patterns/additional/repository-pattern.md)
- [Design Patterns — Factory Method](../design-patterns/creational/factory-method.md)
- [Design Patterns — Strategy](../design-patterns/behavioral/strategy.md)
- [Refactoring Catalog](./refactoring-catalog.md)

## References

- Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software*
- Vaughn Vernon — *Implementing Domain-Driven Design*
- Vaughn Vernon — *Domain-Driven Design Distilled*
- Scott Millett, Nick Tune — *Patterns, Principles, and Practices of Domain-Driven Design*
- Martin Fowler — *Patterns of Enterprise Application Architecture*
- Alberto Brandolini — *Introducing EventStorming*
