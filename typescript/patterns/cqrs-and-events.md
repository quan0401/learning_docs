---
title: "CQRS & Event Patterns in TypeScript"
date: 2026-04-19
updated: 2026-04-19
tags: [cqrs, event-sourcing, architecture, domain-events, projections, typescript]
---

# CQRS & Event Patterns in TypeScript

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `cqrs` `event-sourcing` `architecture` `domain-events` `projections` `typescript`

## Table of Contents

- [Problem Framing](#problem-framing)
- [CQRS Basics](#cqrs-basics)
- [Commands](#commands)
- [Command Handlers](#command-handlers)
- [Events](#events)
- [Projections (Read Models)](#projections-read-models)
- [Event Bus Implementation](#event-bus-implementation)
- [When NOT to Use CQRS](#when-not-to-use-cqrs)
- [Java Parallel](#java-parallel)
- [Related](#related)
- [References](#references)

---

## Problem Framing

You have a high-read API. Orders are written once, updated a few times, but queried thousands of times per minute across multiple shapes: order summaries, customer dashboards, admin search, analytics. A single relational model tries to serve all of them. The queries get complex, the indexes fight each other, and every new read requirement means changing the schema or adding yet another join.

CQRS says: stop using one model for both. Write through a model optimized for enforcing invariants (the aggregate). Read through models optimized for each query shape (projections). Events bridge the two.

---

## CQRS Basics

```
                        ┌─────────────────────────────────┐
                        │          Write Side              │
  Command ──────────▶   │  Command Handler → Aggregate     │
  (PlaceOrder)          │       │                          │
                        │       ▼                          │
                        │  Event Store (append-only)       │
                        └───────────┬─────────────────────┘
                                    │ events
                                    ▼
                        ┌───────────────────────┐
                        │      Event Bus        │
                        └──┬────────┬───────────┘
                           │        │
                ┌──────────▼─┐  ┌───▼──────────┐
                │ Projection │  │ Projection   │
                │ Order List │  │ User Activity│
                └──────┬─────┘  └──────┬───────┘
                       │               │
  Query ───────────────▶               │
  (GetOrders)     Read Model      Read Model
```

**Write side**: commands arrive, a handler loads the aggregate, the aggregate enforces invariants, events are persisted. Strong consistency within one aggregate.

**Read side**: event handlers consume events and update denormalized read models (projections). Each projection is shaped exactly for its query. No joins. No compromise indexes.

**The bridge**: events. The write side emits them; the read side consumes them. The read model is eventually consistent with the write model.

### When CQRS is worth the complexity

| Signal | Plain CRUD is fine | CQRS adds value |
|--------|-------------------|-----------------|
| Read/write ratio | Balanced | 10:1 or higher reads-to-writes |
| Query shapes | 1-2 views on the data | 5+ distinct read shapes |
| Performance | Single DB handles both | Reads need separate optimization |
| Audit requirements | None | Full event trail required |
| Team size | Small, one team | Multiple teams own read/write independently |

---

## Commands

A command is an **intent** to change state. It can be rejected. It carries all the data the handler needs, nothing more. Commands are immutable DTOs — they never change after creation.

### Command definitions

```typescript
// Commands are plain immutable objects. No behavior, just data.

interface Command {
  readonly type: string;
}

interface PlaceOrderCommand extends Command {
  readonly type: "PlaceOrder";
  readonly orderId: string;
  readonly customerId: string;
  readonly items: readonly {
    readonly productId: string;
    readonly productName: string;
    readonly unitPrice: number;
    readonly quantity: number;
  }[];
}

interface CancelOrderCommand extends Command {
  readonly type: "CancelOrder";
  readonly orderId: string;
  readonly reason: string;
}
```

### Command validation — structural vs domain

Structural validation happens before the handler sees the command. Use Zod at the API boundary:

```typescript
import { z } from "zod";

const PlaceOrderSchema = z.object({
  type: z.literal("PlaceOrder"),
  orderId: z.string().uuid(),
  customerId: z.string().uuid(),
  items: z
    .array(
      z.object({
        productId: z.string().uuid(),
        productName: z.string().min(1),
        unitPrice: z.number().positive(),
        quantity: z.number().int().positive(),
      })
    )
    .min(1, "Order must have at least one item"),
});

// Structural: "is this a valid shape?"
// Domain: "can this customer place this order?" — handled inside the aggregate
```

### Command bus — a simple Map-based dispatcher

```typescript
type CommandHandler<C extends Command> = (command: C) => Promise<void>;

class CommandBus {
  private readonly handlers = new Map<string, CommandHandler<any>>();

  register<C extends Command>(
    commandType: C["type"],
    handler: CommandHandler<C>
  ): void {
    if (this.handlers.has(commandType)) {
      throw new Error(`Handler already registered for ${commandType}`);
    }
    this.handlers.set(commandType, handler);
  }

  async dispatch(command: Command): Promise<void> {
    const handler = this.handlers.get(command.type);
    if (!handler) {
      throw new Error(`No handler registered for ${command.type}`);
    }
    await handler(command);
  }
}
```

The command bus is a glorified `Map<string, function>`. In small services, you can skip it entirely and call the handler directly. It shines when you want middleware (logging, retries, transactions) applied uniformly.

---

## Command Handlers

A command handler is the application-layer orchestrator. It loads the aggregate, calls domain logic, persists state, and emits events. This is the exact same concept as the "use case" in the [DDD Tactical Patterns doc](./ddd-tactical-patterns.md#application-services--use-cases) — CQRS just gives it a formal name.

### Full handler for PlaceOrderCommand

```typescript
import { DomainEvent } from "./events";

class PlaceOrderHandler {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventStore: EventStore,
    private readonly eventBus: EventBus
  ) {}

  async handle(command: PlaceOrderCommand): Promise<void> {
    // 1. Create aggregate and execute domain logic
    const order = Order.create({
      id: command.orderId,
      customerId: command.customerId,
    });

    for (const item of command.items) {
      order.addItem({
        lineId: crypto.randomUUID(),
        productId: item.productId,
        productName: item.productName,
        unitPrice: Money.fromNumber(item.unitPrice, "USD"),
        quantity: Quantity.create(item.quantity),
        availableStock: await this.checkStock(item.productId),
      });
    }

    order.checkout(); // enforces invariants, pushes OrderPlaced event

    // 2. Persist — events go to the event store
    const events = order.domainEvents;
    await this.eventStore.append(command.orderId, events);

    // 3. Dispatch events to projections and side effects
    await this.eventBus.publishAll(events);

    // 4. Clear events from aggregate (they've been dispatched)
    order.clearEvents();
  }

  private async checkStock(productId: string): Promise<number> {
    // In real code: call inventory service or read model
    return 100;
  }
}
```

### Registering the handler

```typescript
const commandBus = new CommandBus();
const placeOrderHandler = new PlaceOrderHandler(orderRepo, eventStore, eventBus);

commandBus.register<PlaceOrderCommand>(
  "PlaceOrder",
  (cmd) => placeOrderHandler.handle(cmd)
);

// In your HTTP controller:
// const command = PlaceOrderSchema.parse(req.body);
// await commandBus.dispatch(command);
```

The handler has no framework annotations. It is a plain class with injected dependencies. Compare this to Axon's `@CommandHandler` annotation in the Java doc — same concept, explicit wiring instead of annotation scanning.

---

## Events

Domain events are **immutable facts** — something that happened. Past tense naming: `OrderPlaced`, not `PlaceOrder`. An event can never be rejected; it has already occurred.

### Event definitions

```typescript
interface DomainEvent {
  readonly eventType: string;
  readonly aggregateId: string;
  readonly occurredAt: Date;
  readonly sequence: number; // position in aggregate's event stream
}

interface OrderPlaced extends DomainEvent {
  readonly eventType: "OrderPlaced";
  readonly customerId: string;
  readonly items: readonly {
    readonly productId: string;
    readonly productName: string;
    readonly unitPrice: number;
    readonly quantity: number;
  }[];
  readonly totalAmount: number;
}

interface OrderCancelled extends DomainEvent {
  readonly eventType: "OrderCancelled";
  readonly reason: string;
}
```

### Event serialization

Events are persisted as JSON. They must be self-describing (include `eventType`) so a generic deserializer can reconstruct them:

```typescript
interface StoredEvent {
  readonly streamId: string;   // aggregate id
  readonly sequence: number;
  readonly eventType: string;
  readonly payload: string;    // JSON-serialized event
  readonly metadata: string;   // correlation ID, user ID, etc.
  readonly occurredAt: Date;
}
```

### Simple in-memory event store

```typescript
class InMemoryEventStore implements EventStore {
  private readonly streams = new Map<string, readonly StoredEvent[]>();

  async append(
    streamId: string,
    events: readonly DomainEvent[],
    expectedSequence?: number
  ): Promise<void> {
    const existing = this.streams.get(streamId) ?? [];

    // Optimistic concurrency check
    if (
      expectedSequence !== undefined &&
      existing.length !== expectedSequence
    ) {
      throw new ConcurrencyError(
        `Expected sequence ${expectedSequence}, found ${existing.length}`
      );
    }

    const newStored: StoredEvent[] = events.map((event, i) => ({
      streamId,
      sequence: existing.length + i,
      eventType: event.eventType,
      payload: JSON.stringify(event),
      metadata: JSON.stringify({ timestamp: new Date().toISOString() }),
      occurredAt: event.occurredAt,
    }));

    // Immutable append — create a new array
    this.streams.set(streamId, [...existing, ...newStored]);
  }

  async getStream(streamId: string): Promise<readonly StoredEvent[]> {
    return this.streams.get(streamId) ?? [];
  }
}
```

### Real event store options

| Option | Good for | Notes |
|--------|----------|-------|
| **PostgreSQL append-only table** | Most projects | `UNIQUE (stream_id, sequence)`, BRIN index on timestamp. Simple, battle-tested. |
| **EventStoreDB** | Dedicated event-sourced systems | Purpose-built. Native projections, subscriptions, built-in stream management. |
| **Kafka** | Already Kafka-heavy infra | Awkward fit — Kafka is a log, not a database. Deleting/rewinding streams is painful. |

For a TypeScript stack, Postgres is the pragmatic default. Libraries like [Emmett](https://event-driven-io.github.io/emmett/) provide TS-native patterns over Postgres.

---

## Projections (Read Models)

A projection is a function that consumes events and builds a denormalized read model. Each projection is optimized for one query shape. No joins. No compromise.

### OrderSummaryProjection

```typescript
interface OrderSummaryRow {
  readonly orderId: string;
  readonly customerId: string;
  readonly totalAmount: number;
  readonly itemCount: number;
  readonly status: "PLACED" | "CANCELLED" | "SHIPPED";
  readonly placedAt: Date;
  readonly updatedAt: Date;
}

class OrderSummaryProjection {
  // In production: a database table. Here: in-memory for clarity.
  private readonly rows = new Map<string, OrderSummaryRow>();

  async handleOrderPlaced(event: OrderPlaced): Promise<void> {
    const row: OrderSummaryRow = {
      orderId: event.aggregateId,
      customerId: event.customerId,
      totalAmount: event.totalAmount,
      itemCount: event.items.reduce((sum, i) => sum + i.quantity, 0),
      status: "PLACED",
      placedAt: event.occurredAt,
      updatedAt: event.occurredAt,
    };
    this.rows.set(event.aggregateId, row);
  }

  async handleOrderCancelled(event: OrderCancelled): Promise<void> {
    const existing = this.rows.get(event.aggregateId);
    if (!existing) return; // projection is tolerant of missing data

    // Immutable update
    this.rows.set(event.aggregateId, {
      ...existing,
      status: "CANCELLED",
      updatedAt: event.occurredAt,
    });
  }

  // Query method — this is what the read API calls
  getOrdersForCustomer(customerId: string): readonly OrderSummaryRow[] {
    return [...this.rows.values()].filter(
      (row) => row.customerId === customerId
    );
  }
}
```

### Eventual consistency

The read model **lags behind** the write model. After `PlaceOrderCommand` succeeds, a query against the projection might not see the order yet. This is the fundamental tradeoff.

Strategies to handle it:

1. **Sync projection** — run the projection handler in the same process, synchronously after event dispatch. Simple but couples write throughput to projection speed.
2. **Return write-side data** — after a command, return the aggregate state directly instead of querying the read model. The next query hits the projection.
3. **Polling with expected version** — the client knows the event sequence and polls the read model until it catches up. Works well for UIs.
4. **Accept the lag** — for dashboards and analytics, milliseconds of lag don't matter.

---

## Event Bus Implementation

### In-process synchronous bus

The [DDD doc](./ddd-tactical-patterns.md#typed-in-memory-event-bus) showed a basic typed event bus. Here is a more complete version with subscription management and error handling:

```typescript
type EventHandler<E extends DomainEvent = DomainEvent> = (
  event: E
) => Promise<void>;

interface EventBus {
  subscribe(eventType: string, handler: EventHandler): () => void;
  publish(event: DomainEvent): Promise<void>;
  publishAll(events: readonly DomainEvent[]): Promise<void>;
}

class InProcessEventBus implements EventBus {
  private readonly handlers = new Map<string, EventHandler[]>();

  subscribe(eventType: string, handler: EventHandler): () => void {
    const existing = this.handlers.get(eventType) ?? [];
    this.handlers.set(eventType, [...existing, handler]);

    // Return unsubscribe function
    return () => {
      const current = this.handlers.get(eventType) ?? [];
      this.handlers.set(
        eventType,
        current.filter((h) => h !== handler)
      );
    };
  }

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) ?? [];
    const results = await Promise.allSettled(
      handlers.map((h) => h(event))
    );

    // Log failures but don't crash the write side
    for (const result of results) {
      if (result.status === "rejected") {
        console.error(
          `Event handler failed for ${event.eventType}:`,
          result.reason
        );
      }
    }
  }

  async publishAll(events: readonly DomainEvent[]): Promise<void> {
    for (const event of events) {
      await this.publish(event);
    }
  }
}
```

### Wiring projections to the bus

```typescript
const eventBus = new InProcessEventBus();
const orderSummary = new OrderSummaryProjection();

eventBus.subscribe("OrderPlaced", (event) =>
  orderSummary.handleOrderPlaced(event as OrderPlaced)
);
eventBus.subscribe("OrderCancelled", (event) =>
  orderSummary.handleOrderCancelled(event as OrderCancelled)
);
```

### When to go distributed

| In-process sync bus | Distributed async bus |
|----|---|
| Single service, moderate load | Multiple services consuming events |
| Sub-millisecond projection updates | Projections can tolerate lag |
| Simpler to reason about, debug, test | Decoupled deployment, independent scaling |
| Failure in handler can block writes (use `allSettled`) | Retry, dead-letter queues, at-least-once delivery |

Distributed options for the Node/TS ecosystem:

- **Redis Pub/Sub** — fire-and-forget. No persistence, no replay. Good for cache invalidation.
- **BullMQ (Redis-backed)** — job queues with retry, backoff, dead-letter. Good for reliable async event processing.
- **RabbitMQ** — full message broker. Exchanges, bindings, acknowledgments. Overkill for single-service CQRS but right for multi-service.
- **Kafka** — distributed commit log. Replay from any offset. Right for high-throughput, multi-consumer event streaming. Wrong for small apps.

Start in-process. Move to BullMQ or RabbitMQ when you have a real multi-service need. Do not pre-optimize into Kafka.

---

## When NOT to Use CQRS

CQRS adds real overhead. Two models. Eventual consistency. Idempotency concerns (what if a projection handles the same event twice?). Schema evolution for events. More infrastructure.

### Skip CQRS when:

- **Simple CRUD** — if your read and write models are the same shape, CQRS is just extra code.
- **Low read/write disparity** — if reads and writes scale similarly, a single model works.
- **Small team** — CQRS splits ownership. If one person owns the whole service, the abstraction boundary costs more than it saves.
- **No audit trail requirement** — if you don't need event history, a simpler architecture suffices.
- **Prototype / MVP** — ship the simple thing first. Refactor to CQRS when read performance becomes the bottleneck.

### The idempotency tax

With eventual consistency, projections may process the same event more than once (at-least-once delivery). Every projection handler must be idempotent:

```typescript
async handleOrderPlaced(event: OrderPlaced): Promise<void> {
  // Idempotent: upsert, not insert
  // If the row already exists with this event's data, this is a no-op
  const existing = this.rows.get(event.aggregateId);
  if (existing && existing.placedAt <= event.occurredAt) {
    return; // already processed this or a later event
  }
  this.rows.set(event.aggregateId, { /* ... */ });
}
```

Track the last processed sequence number per projection to avoid replaying events you have already handled.

---

## Java Parallel

If you have been through the [Java Event Sourcing and CQRS](../../java/architecture/event-sourcing-cqrs.md) doc, the concepts are identical. The difference is tooling.

### Side-by-side comparison

| Concept | Java / Spring | TypeScript |
|---------|--------------|------------|
| **Command** | `record PlaceOrderCommand(...)` | `interface PlaceOrderCommand { readonly ... }` |
| **Command handler** | `@CommandHandler` annotation (Axon) | Plain class method, registered on `CommandBus` |
| **Aggregate** | `@Aggregate` class, `@EventSourcingHandler` | Plain class with `domainEvents` array |
| **Event dispatch** | Spring `@EventListener`, Axon event bus | Manual `EventBus.publish()` after persist |
| **Event store** | Axon Server, or JPA `events` table | Postgres table, or EventStoreDB, or in-memory |
| **Projection** | `@EventHandler` class (Axon), `@EventListener` (Spring) | Handler function subscribed to `EventBus` |
| **Framework magic** | Axon handles wiring, replay, snapshots | You build it yourself (or use Emmett) |

### Spring `@EventListener` vs TS manual dispatch

In Spring, publishing a domain event is a one-liner:

```java
// Spring Modulith — events dispatched after transaction commits
applicationEventPublisher.publishEvent(new OrderPlaced(...));
```

Spring discovers `@EventListener` methods automatically via component scanning. In TypeScript, you wire it explicitly:

```typescript
eventBus.subscribe("OrderPlaced", handler);
```

The TS approach is more transparent — you can see every subscription in one place. The Spring approach is more convenient but hides the wiring in annotations.

### Axon Framework vs manual TS

Axon gives you command routing, event replay, snapshot management, saga orchestration, and distributed command/event buses out of the box. In TypeScript, you build each of these yourself or compose libraries. This is both the cost (more code) and the benefit (no framework lock-in, full control over behavior).

For most TS services that need CQRS, the manual approach with a Postgres event table and an in-process event bus covers 80% of use cases. Reach for [Emmett](https://event-driven-io.github.io/emmett/) or EventStoreDB when you need replay, subscriptions, or multi-service projections.

---

## Related

- [DDD Tactical Patterns in TypeScript](./ddd-tactical-patterns.md) — entities, value objects, aggregates, domain events, repository interface
- [Hexagonal Architecture / Ports & Adapters](./hexagonal-architecture.md) — the layering model that CQRS command handlers live inside
- [Java Event Sourcing and CQRS](../../java/architecture/event-sourcing-cqrs.md) — the Java/Axon perspective on the same patterns

---

## References

- Greg Young, [CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) — the original paper
- Martin Fowler, [CQRS](https://martinfowler.com/bliki/CQRS.html)
- Martin Fowler, [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- Oskar Dudycz, [Emmett — Event Sourcing for TypeScript](https://event-driven-io.github.io/emmett/)
- EventStoreDB, [Getting Started](https://developers.eventstore.com/)
- Axon Framework, [Reference Guide](https://docs.axoniq.io/reference-guide/)
- Udi Dahan, [Clarified CQRS](https://udidahan.com/2009/12/09/clarified-cqrs/)
