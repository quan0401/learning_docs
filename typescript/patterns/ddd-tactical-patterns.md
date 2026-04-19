---
title: "DDD Tactical Patterns in TypeScript"
date: 2026-04-19
updated: 2026-04-19
tags: [ddd, architecture, aggregates, value-objects, entities, typescript, domain-events]
---

# DDD Tactical Patterns in TypeScript

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `ddd` `architecture` `aggregates` `value-objects` `entities` `typescript` `domain-events`

## Table of Contents

- [Problem Framing](#problem-framing)
- [Entities](#entities)
- [Value Objects](#value-objects)
- [Aggregates](#aggregates)
- [Domain Events](#domain-events)
- [Repository Interface (The Port)](#repository-interface-the-port)
- [Application Services / Use Cases](#application-services--use-cases)
- [Java Parallel](#java-parallel)
- [Related](#related)
- [References](#references)

---

## Problem Framing

Model an order system where **invalid states are unrepresentable**. An `Order` cannot exist without at least one line item at checkout. Money cannot mix currencies. An `OrderId` cannot be confused with a `CustomerId`. The type system enforces these constraints at compile time; domain logic enforces them at runtime. Together they eliminate entire categories of bugs that unit tests alone cannot catch.

The patterns below are the TypeScript equivalents of the Java tactical DDD patterns covered in [Java DDD Tactical Patterns](../../java/architecture/ddd-tactical-patterns.md). Where Java leans on `record`, `sealed`, and JPA annotations, TypeScript leans on branded types, private constructors, and explicit immutability.

---

## Entities

An entity has a **stable identity** that persists across state changes. Two entities are equal when their IDs match, regardless of their other fields.

### Branded Types for IDs

Plain `string` IDs are stringly typed — nothing stops you from passing a `CustomerId` where an `OrderId` is expected. Branded types fix this at zero runtime cost:

```typescript
type OrderId = string & { readonly __brand: "OrderId" };
type CustomerId = string & { readonly __brand: "CustomerId" };

function toOrderId(raw: string): OrderId {
  return raw as OrderId;
}

function toCustomerId(raw: string): CustomerId {
  return raw as CustomerId;
}
```

Now `toOrderId("abc")` and `toCustomerId("abc")` are assignment-incompatible despite both being strings at runtime. The compiler catches mix-ups; the runtime pays nothing.

### Base Entity Class

```typescript
abstract class Entity<ID> {
  protected constructor(private readonly _id: ID) {}

  get id(): ID {
    return this._id;
  }

  equals(other: Entity<ID>): boolean {
    if (other === null || other === undefined) return false;
    if (this === other) return true;
    return this._id === other._id;
  }
}
```

### Concrete Entity with Factory Validation

Entities expose a static factory instead of a public constructor. The factory validates invariants and returns a `Result` or throws — never a half-built object:

```typescript
class Customer extends Entity<CustomerId> {
  private constructor(
    id: CustomerId,
    private readonly _name: string,
    private readonly _email: Email
  ) {
    super(id);
  }

  static create(params: {
    id: string;
    name: string;
    email: string;
  }): Customer {
    if (params.name.trim().length === 0) {
      throw new DomainError("Customer name cannot be empty");
    }
    const email = Email.create(params.email);
    return new Customer(
      toCustomerId(params.id),
      params.name.trim(),
      email
    );
  }

  get name(): string {
    return this._name;
  }

  get email(): Email {
    return this._email;
  }
}
```

Key points:
- The `private constructor` prevents `new Customer(...)` from outside the class.
- `create()` is the only entry point — it validates before constructing.
- Fields are `readonly`. To change the name, you produce a new `Customer` (or use a domain method that returns a new instance).

---

## Value Objects

A value object has **no identity**. Two `Money` instances with the same amount and currency are interchangeable. Value objects are always immutable.

### Branded Primitives

For simple wrappers around a single primitive, a branded type plus a factory function is enough:

```typescript
type Email = string & { readonly __brand: "Email" };

const Email = {
  create(raw: string): Email {
    const trimmed = raw.trim().toLowerCase();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(trimmed)) {
      throw new DomainError(`Invalid email: ${raw}`);
    }
    return trimmed as Email;
  },
} as const;
```

The companion object pattern (a `type` and a `const` with the same name) gives you both a type and a namespace with no class overhead.

### Compound Value Object: Money

`Money` combines an amount and a currency. Arithmetic across currencies is a domain error:

```typescript
type Currency = "USD" | "EUR" | "JPY";

class Money {
  private constructor(
    private readonly _amount: number,
    private readonly _currency: Currency
  ) {}

  static create(amount: number, currency: Currency): Money {
    if (!Number.isFinite(amount)) {
      throw new DomainError("Amount must be finite");
    }
    if (amount < 0) {
      throw new DomainError("Amount cannot be negative");
    }
    return new Money(amount, currency);
  }

  static zero(currency: Currency): Money {
    return new Money(0, currency);
  }

  get amount(): number {
    return this._amount;
  }

  get currency(): Currency {
    return this._currency;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    const result = this._amount - other._amount;
    if (result < 0) {
      throw new DomainError("Subtraction would result in negative amount");
    }
    return new Money(result, this._currency);
  }

  multiply(factor: number): Money {
    if (factor < 0) {
      throw new DomainError("Factor cannot be negative");
    }
    return new Money(
      Math.round(this._amount * factor * 100) / 100,
      this._currency
    );
  }

  equals(other: Money): boolean {
    return (
      this._amount === other._amount && this._currency === other._currency
    );
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new DomainError(
        `Currency mismatch: ${this._currency} vs ${other._currency}`
      );
    }
  }
}
```

### Quantity Value Object

```typescript
class Quantity {
  private constructor(private readonly _value: number) {}

  static create(value: number): Quantity {
    if (!Number.isInteger(value) || value < 1) {
      throw new DomainError("Quantity must be a positive integer");
    }
    return new Quantity(value);
  }

  get value(): number {
    return this._value;
  }

  add(other: Quantity): Quantity {
    return new Quantity(this._value + other._value);
  }

  equals(other: Quantity): boolean {
    return this._value === other._value;
  }
}
```

Notice every operation returns a **new** instance. The original is never mutated.

---

## Aggregates

An aggregate is a cluster of entities and value objects with a single **aggregate root** that enforces all invariants. Outside code never reaches into the aggregate's internals — every mutation goes through the root.

### OrderLine (Internal Entity)

`OrderLine` is part of the `Order` aggregate. It is never accessed independently:

```typescript
type OrderLineId = string & { readonly __brand: "OrderLineId" };

class OrderLine {
  private constructor(
    private readonly _id: OrderLineId,
    private readonly _productId: string,
    private readonly _productName: string,
    private readonly _unitPrice: Money,
    private readonly _quantity: Quantity
  ) {}

  static create(params: {
    id: string;
    productId: string;
    productName: string;
    unitPrice: Money;
    quantity: Quantity;
  }): OrderLine {
    if (params.productName.trim().length === 0) {
      throw new DomainError("Product name cannot be empty");
    }
    return new OrderLine(
      params.id as OrderLineId,
      params.productId,
      params.productName.trim(),
      params.unitPrice,
      params.quantity
    );
  }

  get id(): OrderLineId {
    return this._id;
  }

  get productId(): string {
    return this._productId;
  }

  get lineTotal(): Money {
    return this._unitPrice.multiply(this._quantity.value);
  }

  get quantity(): Quantity {
    return this._quantity;
  }
}
```

### Order Aggregate Root

The `Order` is the aggregate root. It owns the `OrderLine` collection and enforces every business rule:

```typescript
type OrderStatus = "DRAFT" | "PLACED" | "PAID" | "CANCELLED";

const MINIMUM_ORDER_TOTAL = Money.create(10, "USD");

class Order extends Entity<OrderId> {
  private _lines: readonly OrderLine[];
  private _status: OrderStatus;
  private readonly _domainEvents: DomainEvent[] = [];

  private constructor(
    id: OrderId,
    private readonly _customerId: CustomerId,
    lines: readonly OrderLine[],
    status: OrderStatus
  ) {
    super(id);
    this._lines = lines;
    this._status = status;
  }

  static create(params: { id: string; customerId: string }): Order {
    const order = new Order(
      toOrderId(params.id),
      toCustomerId(params.customerId),
      [],
      "DRAFT"
    );
    return order;
  }

  /** Reconstitute from persistence — skips validation */
  static reconstitute(params: {
    id: OrderId;
    customerId: CustomerId;
    lines: readonly OrderLine[];
    status: OrderStatus;
  }): Order {
    return new Order(params.id, params.customerId, params.lines, params.status);
  }

  get customerId(): CustomerId {
    return this._customerId;
  }

  get lines(): readonly OrderLine[] {
    return this._lines;
  }

  get status(): OrderStatus {
    return this._status;
  }

  get total(): Money {
    return this._lines.reduce(
      (sum, line) => sum.add(line.lineTotal),
      Money.zero("USD")
    );
  }

  get domainEvents(): readonly DomainEvent[] {
    return [...this._domainEvents];
  }

  clearEvents(): void {
    this._domainEvents.length = 0;
  }

  addItem(params: {
    lineId: string;
    productId: string;
    productName: string;
    unitPrice: Money;
    quantity: Quantity;
    availableStock: number;
  }): void {
    if (this._status !== "DRAFT") {
      throw new DomainError("Cannot add items to a non-draft order");
    }
    if (params.quantity.value > params.availableStock) {
      throw new DomainError(
        `Insufficient stock for ${params.productName}: ` +
          `requested ${params.quantity.value}, available ${params.availableStock}`
      );
    }
    const line = OrderLine.create({
      id: params.lineId,
      productId: params.productId,
      productName: params.productName,
      unitPrice: params.unitPrice,
      quantity: params.quantity,
    });
    this._lines = [...this._lines, line];
  }

  checkout(): void {
    if (this._status !== "DRAFT") {
      throw new DomainError("Only draft orders can be checked out");
    }
    if (this._lines.length === 0) {
      throw new DomainError("Cannot checkout an empty order");
    }
    if (!this.total.isGreaterThan(MINIMUM_ORDER_TOTAL)) {
      throw new DomainError(
        `Order total must exceed ${MINIMUM_ORDER_TOTAL.amount} ` +
          `${MINIMUM_ORDER_TOTAL.currency}`
      );
    }
    this._status = "PLACED";
    this._domainEvents.push(
      OrderPlaced.create({
        orderId: this.id,
        customerId: this._customerId,
        total: this.total,
        occurredAt: new Date(),
      })
    );
  }

  markPaid(): void {
    if (this._status !== "PLACED") {
      throw new DomainError("Only placed orders can be marked as paid");
    }
    this._status = "PAID";
    this._domainEvents.push(
      PaymentReceived.create({
        orderId: this.id,
        amount: this.total,
        occurredAt: new Date(),
      })
    );
  }
}
```

Key invariants enforced by the aggregate:
- Items can only be added to `DRAFT` orders.
- Stock is checked at add time (the caller passes available stock; the aggregate validates).
- Checkout requires at least one line and a minimum total.
- State transitions follow `DRAFT -> PLACED -> PAID` (or `CANCELLED`).
- Domain events are collected internally — only the application layer dispatches them.

---

## Domain Events

Domain events are **immutable records** of something that happened in the domain. They carry all the data a subscriber needs without exposing the aggregate internals.

### Event Definitions

```typescript
interface DomainEvent {
  readonly eventType: string;
  readonly occurredAt: Date;
}

class OrderPlaced implements DomainEvent {
  readonly eventType = "OrderPlaced" as const;

  private constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly total: Money,
    readonly occurredAt: Date
  ) {}

  static create(params: {
    orderId: OrderId;
    customerId: CustomerId;
    total: Money;
    occurredAt: Date;
  }): OrderPlaced {
    return new OrderPlaced(
      params.orderId,
      params.customerId,
      params.total,
      params.occurredAt
    );
  }
}

class PaymentReceived implements DomainEvent {
  readonly eventType = "PaymentReceived" as const;

  private constructor(
    readonly orderId: OrderId,
    readonly amount: Money,
    readonly occurredAt: Date
  ) {}

  static create(params: {
    orderId: OrderId;
    amount: Money;
    occurredAt: Date;
  }): PaymentReceived {
    return new PaymentReceived(
      params.orderId,
      params.amount,
      params.occurredAt
    );
  }
}
```

### Typed In-Memory Event Bus

A simple event bus using a typed event map. Each event type maps to its handler signature:

```typescript
type EventMap = {
  OrderPlaced: (event: OrderPlaced) => Promise<void>;
  PaymentReceived: (event: PaymentReceived) => Promise<void>;
};

class DomainEventBus {
  private readonly handlers = new Map<string, Array<(event: DomainEvent) => Promise<void>>>();

  on<K extends keyof EventMap>(
    eventType: K,
    handler: EventMap[K]
  ): void {
    const existing = this.handlers.get(eventType) ?? [];
    this.handlers.set(eventType, [
      ...existing,
      handler as (event: DomainEvent) => Promise<void>,
    ]);
  }

  async dispatch(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) ?? [];
    await Promise.all(handlers.map((h) => h(event)));
  }

  async dispatchAll(events: readonly DomainEvent[]): Promise<void> {
    for (const event of events) {
      await this.dispatch(event);
    }
  }
}
```

### How Aggregates Collect Events

The aggregate holds an internal `_domainEvents` array (shown in the `Order` class above). Domain methods like `checkout()` and `markPaid()` push events onto this list. The application service reads them after saving:

```
1. Load aggregate from repository
2. Call domain method (events accumulate inside aggregate)
3. Save aggregate (persistence transaction)
4. Dispatch events from aggregate
5. Clear events
```

This ordering guarantees events are only dispatched after successful persistence. If the save fails, no events leak.

---

## Repository Interface (The Port)

The repository interface lives in the **domain layer**. It knows nothing about Prisma, Drizzle, or any other ORM. It is a port in hexagonal architecture terms — the adapter lives in the infrastructure layer.

### The Interface

```typescript
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
  findByCustomerId(customerId: CustomerId): Promise<readonly Order[]>;
}
```

No Prisma types. No database column names. No `@Injectable()` decorator. Pure domain contract.

### Prisma Adapter (Infrastructure Layer)

```typescript
import { PrismaClient } from "@prisma/client";

class PrismaOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({
      where: { id: id as string },
      include: { lines: true },
    });
    if (row === null) return null;
    return this.toDomain(row);
  }

  async save(order: Order): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      await tx.order.upsert({
        where: { id: order.id as string },
        create: {
          id: order.id as string,
          customerId: order.customerId as string,
          status: order.status,
        },
        update: {
          status: order.status,
        },
      });

      await tx.orderLine.deleteMany({
        where: { orderId: order.id as string },
      });

      if (order.lines.length > 0) {
        await tx.orderLine.createMany({
          data: order.lines.map((line) => ({
            id: line.id as string,
            orderId: order.id as string,
            productId: line.productId,
            quantity: line.quantity.value,
            unitPrice: line.lineTotal.amount,
            currency: line.lineTotal.currency,
          })),
        });
      }
    });
  }

  async findByCustomerId(customerId: CustomerId): Promise<readonly Order[]> {
    const rows = await this.prisma.order.findMany({
      where: { customerId: customerId as string },
      include: { lines: true },
    });
    return rows.map((row) => this.toDomain(row));
  }

  private toDomain(row: any): Order {
    const lines = row.lines.map((l: any) =>
      OrderLine.create({
        id: l.id,
        productId: l.productId,
        productName: l.productName,
        unitPrice: Money.create(l.unitPrice, l.currency),
        quantity: Quantity.create(l.quantity),
      })
    );
    return Order.reconstitute({
      id: row.id as OrderId,
      customerId: row.customerId as CustomerId,
      lines,
      status: row.status,
    });
  }
}
```

The domain layer has **zero imports from `@prisma/client`**. Swapping Prisma for Drizzle, TypeORM, or a plain SQL adapter means writing a new class that implements `OrderRepository` — no domain changes.

---

## Application Services / Use Cases

Application services orchestrate domain objects. They load aggregates, call domain methods, persist changes, and dispatch events. They contain **no business logic** — that belongs in the domain layer.

### PlaceOrderUseCase

```typescript
interface PlaceOrderCommand {
  readonly orderId: string;
  readonly customerId: string;
  readonly items: ReadonlyArray<{
    readonly lineId: string;
    readonly productId: string;
    readonly productName: string;
    readonly unitPrice: number;
    readonly currency: Currency;
    readonly quantity: number;
  }>;
}

class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly stockService: StockQueryService,
    private readonly eventBus: DomainEventBus
  ) {}

  async execute(command: PlaceOrderCommand): Promise<OrderId> {
    // 1. Create the aggregate
    const order = Order.create({
      id: command.orderId,
      customerId: command.customerId,
    });

    // 2. Add items — domain validates each one
    for (const item of command.items) {
      const availableStock = await this.stockService.getAvailable(
        item.productId
      );
      order.addItem({
        lineId: item.lineId,
        productId: item.productId,
        productName: item.productName,
        unitPrice: Money.create(item.unitPrice, item.currency),
        quantity: Quantity.create(item.quantity),
        availableStock,
      });
    }

    // 3. Checkout — domain enforces minimum total, non-empty lines
    order.checkout();

    // 4. Persist
    await this.orderRepo.save(order);

    // 5. Dispatch domain events
    await this.eventBus.dispatchAll(order.domainEvents);
    order.clearEvents();

    return order.id;
  }
}
```

### The Full Flow

```
Controller / Route Handler
  │
  ▼
PlaceOrderUseCase.execute(command)
  │
  ├─ Order.create(...)               ← aggregate factory
  ├─ stockService.getAvailable(...)   ← infrastructure query
  ├─ order.addItem(...)               ← domain logic (stock check)
  ├─ order.checkout()                 ← domain logic (invariants + event)
  ├─ orderRepo.save(order)            ← persistence (adapter)
  ├─ eventBus.dispatchAll(...)        ← side effects after persistence
  └─ return orderId
```

The use case is thin glue. If you find business rules creeping into the use case, push them down into the aggregate or a domain service.

---

## Java Parallel

If you already know Java DDD (from [Java DDD Tactical Patterns](../../java/architecture/ddd-tactical-patterns.md)), this table maps the concepts:

| Concept | Java | TypeScript |
|---------|------|------------|
| **Entity identity** | `@Id` JPA field, `equals()`/`hashCode()` on ID | `Entity<ID>` base class, branded type for ID |
| **Entity ID type safety** | Dedicated `OrderId` record wrapping `UUID` | Branded type: `string & { __brand: "OrderId" }` |
| **Value Object** | Java `record` + `@Embeddable` | Class with private constructor + static factory, or branded type for primitives |
| **Immutability** | `record` fields are final by default | `readonly` fields, new instance on every operation |
| **Aggregate root** | Class with `@Aggregate` (Modulith) or convention | Class extending `Entity`, sole public API for the cluster |
| **Invariant enforcement** | Guard clauses in domain methods | Same — guard clauses, `DomainError` throws |
| **Domain events** | `@DomainEvents` + `AbstractAggregateRoot` (Spring Data) | Internal `_domainEvents` list, manual dispatch in use case |
| **Event dispatch** | Spring auto-publishes after `save()` | Explicit `eventBus.dispatchAll()` after `repo.save()` |
| **Repository** | `interface OrderRepository extends JpaRepository` | `interface OrderRepository { findById, save }` — no framework base |
| **Repository impl** | Spring Data generates it from interface | Hand-written Prisma/Drizzle adapter class |
| **Application service** | `@Service` class with `@Transactional` | Plain class, injected dependencies, no decorators required |
| **Dependency injection** | Spring IoC container (`@Autowired`) | Constructor injection, wired manually or via lightweight DI (tsyringe, awilix) |
| **Validation** | Bean Validation (`@NotNull`, `@Size`) | Factory method guards, branded types, Zod at API boundary |

### Key Differences to Watch

1. **No free ORM magic.** Java Spring Data generates repository implementations from interfaces. In TypeScript you write the adapter yourself. This is more work but gives you full control over the mapping between domain and persistence models.

2. **No annotation-driven events.** Spring's `@DomainEvents` auto-publishes after `save()`. In TypeScript you dispatch explicitly — which is actually easier to reason about since the timing is visible in the use case code.

3. **Branded types replace wrapper classes.** Java needs a `record OrderId(UUID value)` with a real wrapper at runtime. TypeScript branded types exist only at compile time — zero overhead, same safety.

4. **No `@Transactional`.** You manage transactions explicitly (Prisma's `$transaction`, Drizzle's `db.transaction`). This makes transaction boundaries visible rather than hidden behind an annotation.

---

## Related

- [Java DDD Tactical Patterns](../../java/architecture/ddd-tactical-patterns.md) — the Java equivalent of this document
- [Java Event Sourcing and CQRS](../../java/architecture/event-sourcing-cqrs.md) — advanced event patterns

---

## References

- Evans, Eric. *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Addison-Wesley, 2003.
- Brandolini, Alberto. *Introducing EventStorming*. Leanpub, 2021.
- Vernon, Vaughn. *Implementing Domain-Driven Design*. Addison-Wesley, 2013.
- [TypeScript Handbook — Type Branding](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)
- [Prisma Client API Reference](https://www.prisma.io/docs/orm/reference/prisma-client-reference)
- [Khalil Stemmler — DDD in TypeScript](https://khalilstemmler.com/articles/categories/domain-driven-design/)
- [Node.js DDD Starter](https://github.com/stemmlerjs/ddd-forum) — reference implementation
