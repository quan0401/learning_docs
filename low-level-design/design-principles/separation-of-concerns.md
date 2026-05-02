---
title: "Separation of Concerns"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, design-principles, separation-of-concerns, architecture, layering]
---

# Separation of Concerns

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `design-principles` `separation-of-concerns` `architecture` `layering`

## Summary

Separation of Concerns (SoC), articulated by Edsger Dijkstra in 1974, says a complex system should be decomposed so that distinct concerns — distinct reasons for change, distinct stakeholders, distinct rates of evolution — live in distinct modules. SoC is the *why* behind layered architecture, the SOLID principles, and the careful handling of cross-cutting concerns like logging, security, and transactions.

## Table of Contents

- [What a "Concern" Means](#what-a-concern-means)
- [The Core Rule](#the-core-rule)
- [Cross-Cutting Concerns](#cross-cutting-concerns)
- [How SoC Drives Layering](#how-soc-drives-layering)
- [Concrete Examples](#concrete-examples)
- [SoC vs Single Responsibility](#soc-vs-single-responsibility)
- [Failure Modes](#failure-modes)
- [Checklist](#checklist)
- [Related](#related)

## What a "Concern" Means

A *concern* is anything you might need to think about, change, or test independently:

- **Functional concerns:** what the system does — pricing, search, checkout, fraud detection
- **Quality attributes:** performance, security, observability
- **Stakeholder concerns:** business rules (changes by product), UI (changes by design), persistence (changes by infra)
- **Lifecycle concerns:** code that changes weekly vs. yearly

Two things are separate concerns if they:

1. Change for different reasons
2. Change at different times
3. Are owned by different people
4. Need to be reasoned about independently

If you can answer "yes" to any of these, they probably deserve separate modules.

## The Core Rule

A module — class, file, package, service — should address **one concern**. Concerns should not be smeared across modules, and a single module should not host multiple unrelated concerns.

```typescript
// Bad — mixes HTTP handling, validation, business rule, persistence, email
app.post("/orders", async (req, res) => {
  if (!req.body.customerId || !req.body.items) {
    return res.status(400).json({ error: "missing fields" });
  }
  const customer = await db.query("SELECT * FROM customers WHERE id = $1", [req.body.customerId]);
  if (!customer) return res.status(404).json({ error: "customer not found" });
  if (req.body.items.length === 0) return res.status(400).json({ error: "empty order" });

  const total = req.body.items.reduce((s: number, i: any) => s + i.price * i.qty, 0);
  if (total > 10000 && !customer.preApproved) return res.status(403).json({ error: "approval required" });

  const order = await db.query(
    "INSERT INTO orders (...) VALUES (...) RETURNING *",
    [/* ... */]
  );
  await sendgrid.send({
    to: customer.email,
    subject: "Order placed",
    body: `Order ${order.id} confirmed.`,
  });
  res.json(order);
});
```

This handler conflates HTTP shape, validation, customer lookup, business policy, persistence, and notification. Each concern has a different change driver.

```typescript
// Good — each concern in its own place
app.post("/orders", validateBody(CreateOrderSchema), async (req, res, next) => {
  try {
    const order = await orderService.placeOrder(req.body);
    res.status(201).json(orderPresenter(order));
  } catch (e) { next(e); }
});

class OrderService {
  constructor(private customers: CustomerRepo, private orders: OrderRepo, private notifier: Notifier) {}
  async placeOrder(input: CreateOrderInput): Promise<Order> {
    const customer = await this.customers.requireById(input.customerId);
    const order = Order.create(customer, input.items);     // domain rules here
    await this.orders.save(order);
    await this.notifier.orderPlaced(order, customer);
    return order;
  }
}
```

HTTP concern, validation concern, business rule concern, persistence concern, and notification concern each live in their own seam.

## Cross-Cutting Concerns

Some concerns naturally apply to *every* module — they cut across the system. The classic list:

- **Logging** — you want it everywhere, but you don't want it tangled into every method body.
- **Security / authentication / authorization** — applies to every entry point.
- **Transactions** — boundary management around use cases.
- **Caching** — wraps existing operations.
- **Validation** — at every boundary.
- **Metrics / tracing / observability** — across the request lifecycle.
- **Error translation** — turning technical errors into stable contract errors.
- **Internationalization** — pervasive concern with a tiny per-call surface.

If you scatter these into every method, you create *tangling* (multiple concerns in one place) and *scattering* (one concern in many places). Both are bad.

Mechanisms to handle cross-cutting concerns cleanly:

### Middleware / interceptors

```java
// Spring Boot — a single interceptor handles authn for all routes
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        var token = req.getHeader("Authorization");
        if (!tokenService.isValid(token)) {
            res.setStatus(401);
            return false;
        }
        return true;
    }
}
```

### Annotations / aspects

```java
// Transactional boundary declared, not coded
@Transactional
public Order placeOrder(CreateOrderInput input) {
    // business logic — no manual begin/commit
}
```

Spring's `@Transactional` is an aspect-oriented technique. The transaction concern is woven in at runtime; the method itself stays focused on the domain.

### Decorators

```typescript
// Cache decorator — caching is a separate concern from data fetching
function cached<T extends object, K extends keyof T>(target: T, key: K): T {
  const original = target[key] as unknown as Function;
  const cache = new Map();
  (target as any)[key] = function (...args: any[]) {
    const k = JSON.stringify(args);
    if (!cache.has(k)) cache.set(k, original.apply(this, args));
    return cache.get(k);
  };
  return target;
}
```

### Pipes and filters

In stream processing, observability and validation can live as separate stages of a pipeline rather than being woven into core processors.

The unifying idea: cross-cutting concerns get a *home* — a middleware stack, an aspect, a decorator chain — so they don't pollute domain code.

## How SoC Drives Layering

Layered architecture is SoC applied at module-group scale. A typical web service:

```
┌─────────────────────────────────────────────┐
│ Presentation layer (controllers, views)     │  ← HTTP / serialization concern
├─────────────────────────────────────────────┤
│ Application layer (use cases, services)     │  ← orchestration concern
├─────────────────────────────────────────────┤
│ Domain layer (entities, value objects)      │  ← business rules concern
├─────────────────────────────────────────────┤
│ Infrastructure layer (repos, clients)       │  ← persistence / external IO concern
└─────────────────────────────────────────────┘
```

Dependencies point downward (or, in Hexagonal / Clean Architecture, inward toward the domain). Each layer can change for its own reasons:

- Presentation changes when API contracts or UI shapes change.
- Application changes when use cases change.
- Domain changes when business rules change.
- Infrastructure changes when you swap databases or vendors.

Without SoC, a "domain" change risks breaking the database, the API contract, and the UI all at once.

## Concrete Examples

### Java — separating policy from mechanism

```java
// Mechanism: persistence
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}

// Policy: business rule
public class Order {
    public void cancel(Clock clock, RefundPolicy policy) {
        if (status == Status.SHIPPED) {
            throw new CannotCancelException("already shipped");
        }
        var refund = policy.calculateRefund(this, clock.instant());
        this.status = Status.CANCELLED;
        this.refund = refund;
    }
}

// Application service: orchestration
public class CancelOrderUseCase {
    public void execute(OrderId id) {
        var order = orders.findById(id).orElseThrow();
        order.cancel(clock, refundPolicy);
        orders.save(order);
        events.publish(new OrderCancelled(id));
    }
}
```

Each module has one concern. A change to refund policy doesn't touch the repository. A change to persistence doesn't touch the cancellation rule.

### TypeScript — UI / state / data

```typescript
// Data concern
async function fetchOrders(customerId: string): Promise<Order[]> {
  const r = await fetch(`/api/customers/${customerId}/orders`);
  return r.json();
}

// State concern
function useOrders(customerId: string) {
  return useQuery(["orders", customerId], () => fetchOrders(customerId));
}

// Presentation concern
function OrdersList({ customerId }: { customerId: string }) {
  const { data, isLoading, error } = useOrders(customerId);
  if (isLoading) return <Spinner />;
  if (error) return <ErrorBanner error={error} />;
  return <ul>{data!.map(o => <OrderRow key={o.id} order={o} />)}</ul>;
}
```

Each function has one job. The component is replaceable without touching fetching; fetching is replaceable without touching the component.

## SoC vs Single Responsibility

SoC and SRP are closely related but not identical:

- **SoC** is a *system-level* principle — concerns should be separated across the whole architecture.
- **SRP** is a *module-level* principle — a class should have one reason to change.

SoC drives where you put the layer boundaries. SRP drives how you split classes within a layer. Following both produces a system where every module has one job and every job has one home.

## Failure Modes

### Tangled concerns

Multiple concerns in one module. Symptoms: a class with `database`, `email`, and `business rule` words in the same method body. Fix: extract collaborators.

### Scattered concerns

One concern in many modules. Symptoms: every endpoint has its own copy of permission checking, every service does its own logging boilerplate. Fix: move to middleware, aspects, or a shared module.

### Wrong seams

Concerns separated, but along the wrong lines — e.g., splitting by *technical layer* (`Controller`, `Service`, `Repo`) but mixing *business domains* across each. Symptoms: changing one feature requires touching every layer in tightly-coupled lockstep. Fix: organize by domain (feature folders / bounded contexts) inside each layer.

### Over-separation

Splitting concerns that genuinely belong together. Symptoms: trivial classes that just delegate, ceremony-heavy code, deep call chains. Fix: collapse what changes together.

## Checklist

- [ ] Does each module have one clearly statable concern?
- [ ] If a stakeholder asks "where do I change X", is the answer one place?
- [ ] Are cross-cutting concerns handled in middleware, aspects, or decorators rather than inline?
- [ ] Do layers depend in one direction (or inward toward the domain)?
- [ ] Can you swap the persistence layer without touching domain rules?
- [ ] Can you swap the HTTP framework without touching business logic?
- [ ] Are you separating along the right seams (domain), not just technical (layer)?

## Related

- [Coupling and Cohesion](coupling-and-cohesion.md)
- [Composing Objects Principle](composing-objects-principle.md)
- [Code Smells and Refactoring Triggers](code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](anti-patterns-in-oo-design.md)
- [Single Responsibility Principle](../solid/single-responsibility-principle.md)
- [Dependency Inversion Principle](../solid/dependency-inversion-principle.md)
- [Encapsulation](../oop-fundamentals/encapsulation.md)

## References

- Dijkstra, E. — "On the role of scientific thought" (1974)
- Parnas, D. — "On the Criteria To Be Used in Decomposing Systems into Modules"
- Martin, R. — *Clean Architecture*
- Evans, E. — *Domain-Driven Design*
