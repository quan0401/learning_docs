# Hexagonal Architecture / Ports & Adapters in TypeScript

**Updated:** 2026-04-24

> Swap Express for Fastify without touching a single line of business logic.

If that claim sounds too good to be true, you have probably worked in codebases where
`req` and `res` objects bleed into service methods, Prisma types show up in domain
logic, and "refactoring the framework" means rewriting everything. Hexagonal
architecture exists to prevent that coupling from forming in the first place. For the Spring Boot side of the same structure choices, see [Spring Boot Project Structure](../../java/architecture/project-structure.md).

---

## The Core Idea

The architecture has one non-negotiable rule: **dependencies point inward**.

```
                  ┌─────────────────────────────────┐
                  │         DRIVING SIDE             │
                  │  (things that USE the system)    │
                  │                                  │
                  │   HTTP (Express / Fastify)       │
                  │   CLI commands                   │
                  │   Test harnesses                 │
                  │   gRPC / GraphQL                 │
                  └──────────┬──────────────────────┘
                             │
                             │  calls
                             ▼
                  ┌──────────────────────────────────┐
                  │       DRIVING PORTS               │
                  │   (what the domain OFFERS)        │
                  │                                   │
                  │   Use-case interfaces              │
                  │   e.g. PlaceOrderUseCase           │
                  └──────────┬───────────────────────┘
                             │
                             ▼
               ┌──────────────────────────────────────┐
               │                                      │
               │          DOMAIN CORE                 │
               │                                      │
               │   Entities, Value Objects,            │
               │   Domain Services, Business Rules     │
               │                                      │
               │   ZERO framework imports              │
               │   ZERO infrastructure imports         │
               │                                      │
               └──────────┬───────────────────────────┘
                          │
                          │  depends on
                          ▼
               ┌──────────────────────────────────────┐
               │       DRIVEN PORTS                    │
               │   (what the domain NEEDS)             │
               │                                       │
               │   OrderRepository (interface)          │
               │   EmailService (interface)             │
               │   PaymentGateway (interface)           │
               └──────────┬───────────────────────────┘
                          │
                          │  implemented by
                          ▼
               ┌──────────────────────────────────────┐
               │         DRIVEN SIDE                   │
               │   (things the system USES)            │
               │                                       │
               │   Prisma / TypeORM repository          │
               │   SendGrid email adapter               │
               │   Stripe payment adapter               │
               │   RabbitMQ queue client                │
               └──────────────────────────────────────┘
```

**Spring parallel**: Think of it as Controller -> Service -> Repository, but with
explicit interfaces at every boundary. The domain never knows what implements those
interfaces.

---

## Directory Structure

```
src/
├── domain/                # Entities, value objects, domain services, port interfaces
│   ├── entities/
│   │   └── Order.ts
│   ├── value-objects/
│   │   ├── Money.ts
│   │   └── OrderStatus.ts
│   ├── services/
│   │   └── PricingService.ts
│   └── ports/
│       ├── driven/        # What the domain NEEDS from the outside world
│       │   ├── OrderRepository.ts
│       │   ├── EmailService.ts
│       │   └── PaymentGateway.ts
│       └── driving/       # What the domain OFFERS to the outside world
│           └── PlaceOrderUseCase.ts
│
├── application/           # Use-case implementations, DTOs, application services
│   ├── use-cases/
│   │   └── PlaceOrderService.ts
│   └── dto/
│       ├── PlaceOrderRequest.ts
│       └── OrderResponse.ts
│
├── infrastructure/        # Driven adapters (outbound)
│   ├── persistence/
│   │   └── PrismaOrderRepository.ts
│   ├── email/
│   │   └── SendGridEmailService.ts
│   └── payment/
│       └── StripePaymentGateway.ts
│
├── adapters/              # Driving adapters (inbound)
│   ├── http/
│   │   ├── express/
│   │   │   └── OrderRoutes.ts
│   │   └── fastify/
│   │       └── OrderRoutes.ts
│   └── cli/
│       └── PlaceOrderCommand.ts
│
└── main.ts                # Composition root — wires everything together
```

**Key insight**: The `domain/` directory has zero imports from `infrastructure/` or
`adapters/`. If you grep `domain/` for "express", "prisma", or "fastify" and find
anything, the architecture is broken.

---

## Port Interfaces

### Driven Ports (What the Domain NEEDS)

These are contracts the domain declares. Infrastructure provides implementations.

```typescript
// domain/ports/driven/OrderRepository.ts

import { Order } from "../../entities/Order";

export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<readonly Order[]>;
  save(order: Order): Promise<Order>;
  delete(id: string): Promise<void>;
}
```

```typescript
// domain/ports/driven/EmailService.ts

export interface EmailNotification {
  readonly to: string;
  readonly subject: string;
  readonly body: string;
}

export interface EmailService {
  send(notification: EmailNotification): Promise<void>;
}
```

```typescript
// domain/ports/driven/PaymentGateway.ts

import { Money } from "../../value-objects/Money";

export interface ChargeResult {
  readonly transactionId: string;
  readonly success: boolean;
  readonly failureReason?: string;
}

export interface PaymentGateway {
  charge(customerId: string, amount: Money): Promise<ChargeResult>;
  refund(transactionId: string): Promise<void>;
}
```

### Driving Ports (What the Domain OFFERS)

These define the use cases that adapters can invoke.

```typescript
// domain/ports/driving/PlaceOrderUseCase.ts

import { OrderResponse } from "../../../application/dto/OrderResponse";

export interface PlaceOrderCommand {
  readonly customerId: string;
  readonly items: ReadonlyArray<{
    readonly productId: string;
    readonly quantity: number;
  }>;
}

export interface PlaceOrderUseCase {
  execute(command: PlaceOrderCommand): Promise<OrderResponse>;
}
```

---

## Domain Entities

The domain is plain TypeScript. No decorators, no ORM annotations, no framework
coupling. Compare this with a Spring `@Entity` — here, persistence mapping lives
in the adapter, not the entity.

```typescript
// domain/entities/Order.ts

import { Money } from "../value-objects/Money";
import { OrderStatus } from "../value-objects/OrderStatus";

export interface OrderItem {
  readonly productId: string;
  readonly quantity: number;
  readonly unitPrice: Money;
}

export class Order {
  private constructor(
    readonly id: string,
    readonly customerId: string,
    readonly items: readonly OrderItem[],
    readonly status: OrderStatus,
    readonly createdAt: Date
  ) {}

  static create(id: string, customerId: string, items: OrderItem[]): Order {
    if (items.length === 0) {
      throw new Error("Order must contain at least one item");
    }
    return new Order(id, customerId, items, OrderStatus.PENDING, new Date());
  }

  get total(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.unitPrice.multiply(item.quantity)),
      Money.zero("USD")
    );
  }

  confirm(): Order {
    if (this.status !== OrderStatus.PENDING) {
      throw new Error(`Cannot confirm order in status ${this.status}`);
    }
    return new Order(
      this.id,
      this.customerId,
      this.items,
      OrderStatus.CONFIRMED,
      this.createdAt
    );
  }

  cancel(): Order {
    if (this.status === OrderStatus.SHIPPED) {
      throw new Error("Cannot cancel a shipped order");
    }
    return new Order(
      this.id,
      this.customerId,
      this.items,
      OrderStatus.CANCELLED,
      this.createdAt
    );
  }
}
```

Notice: `confirm()` and `cancel()` return **new Order instances** rather than
mutating in place. State transitions live in the entity, not in a service.

---

## Application Layer (Use Cases)

The application layer orchestrates domain objects and ports. It is the
implementation of the driving port interfaces.

```typescript
// application/use-cases/PlaceOrderService.ts

import { randomUUID } from "node:crypto";
import { Order, OrderItem } from "../../domain/entities/Order";
import { OrderRepository } from "../../domain/ports/driven/OrderRepository";
import { PaymentGateway } from "../../domain/ports/driven/PaymentGateway";
import { EmailService } from "../../domain/ports/driven/EmailService";
import {
  PlaceOrderUseCase,
  PlaceOrderCommand,
} from "../../domain/ports/driving/PlaceOrderUseCase";
import { OrderResponse } from "../dto/OrderResponse";

export class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly emailService: EmailService
  ) {}

  async execute(command: PlaceOrderCommand): Promise<OrderResponse> {
    // 1. Create domain entity
    const items: OrderItem[] = command.items.map((item) => ({
      productId: item.productId,
      quantity: item.quantity,
      unitPrice: Money.of(0, "USD"), // pricing looked up elsewhere
    }));

    const order = Order.create(randomUUID(), command.customerId, items);

    // 2. Charge payment through port
    const chargeResult = await this.paymentGateway.charge(
      command.customerId,
      order.total
    );

    if (!chargeResult.success) {
      throw new Error(`Payment failed: ${chargeResult.failureReason}`);
    }

    // 3. Confirm and persist through port
    const confirmedOrder = order.confirm();
    const savedOrder = await this.orderRepo.save(confirmedOrder);

    // 4. Notify through port
    await this.emailService.send({
      to: command.customerId,
      subject: "Order Confirmed",
      body: `Your order ${savedOrder.id} has been confirmed.`,
    });

    return OrderResponse.fromDomain(savedOrder);
  }
}
```

**Key observation**: `PlaceOrderService` depends on three **interfaces**, not three
concrete classes. It has no idea whether the payment goes to Stripe or a test fake.

---

## Adapter Implementations

### Driving Adapter: Express

```typescript
// adapters/http/express/OrderRoutes.ts

import { Router, Request, Response } from "express";
import { PlaceOrderUseCase } from "../../../domain/ports/driving/PlaceOrderUseCase";

export function createOrderRoutes(placeOrder: PlaceOrderUseCase): Router {
  const router = Router();

  router.post("/orders", async (req: Request, res: Response) => {
    try {
      const result = await placeOrder.execute({
        customerId: req.body.customerId,
        items: req.body.items,
      });
      res.status(201).json(result);
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  });

  return router;
}
```

### Driving Adapter: Fastify

Swapping frameworks means writing a new adapter. The use case does not change.

```typescript
// adapters/http/fastify/OrderRoutes.ts

import { FastifyInstance } from "fastify";
import { PlaceOrderUseCase } from "../../../domain/ports/driving/PlaceOrderUseCase";

export function registerOrderRoutes(
  app: FastifyInstance,
  placeOrder: PlaceOrderUseCase
): void {
  app.post("/orders", async (request, reply) => {
    try {
      const body = request.body as {
        customerId: string;
        items: Array<{ productId: string; quantity: number }>;
      };

      const result = await placeOrder.execute({
        customerId: body.customerId,
        items: body.items,
      });

      return reply.status(201).send(result);
    } catch (error) {
      return reply.status(400).send({ error: (error as Error).message });
    }
  });
}
```

### Driven Adapter: Prisma Repository

```typescript
// infrastructure/persistence/PrismaOrderRepository.ts

import { PrismaClient } from "@prisma/client";
import { Order, OrderItem } from "../../domain/entities/Order";
import { OrderRepository } from "../../domain/ports/driven/OrderRepository";
import { Money } from "../../domain/value-objects/Money";
import { OrderStatus } from "../../domain/value-objects/OrderStatus";

export class PrismaOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const record = await this.prisma.order.findUnique({
      where: { id },
      include: { items: true },
    });

    if (!record) return null;

    // Map Prisma types -> Domain types (this is the adapter's job)
    return this.toDomain(record);
  }

  async findByCustomerId(customerId: string): Promise<readonly Order[]> {
    const records = await this.prisma.order.findMany({
      where: { customerId },
      include: { items: true },
    });
    return records.map((r) => this.toDomain(r));
  }

  async save(order: Order): Promise<Order> {
    await this.prisma.order.upsert({
      where: { id: order.id },
      create: {
        id: order.id,
        customerId: order.customerId,
        status: order.status,
        createdAt: order.createdAt,
        items: {
          create: order.items.map((item) => ({
            productId: item.productId,
            quantity: item.quantity,
            unitPriceCents: item.unitPrice.cents,
            currency: item.unitPrice.currency,
          })),
        },
      },
      update: {
        status: order.status,
      },
    });
    return order;
  }

  async delete(id: string): Promise<void> {
    await this.prisma.order.delete({ where: { id } });
  }

  private toDomain(record: any): Order {
    // Translation between Prisma schema and domain entities
    // lives ONLY in the adapter — never in the domain
    return Order.reconstitute({
      id: record.id,
      customerId: record.customerId,
      items: record.items.map((i: any) => ({
        productId: i.productId,
        quantity: i.quantity,
        unitPrice: Money.ofCents(i.unitPriceCents, i.currency),
      })),
      status: record.status as OrderStatus,
      createdAt: record.createdAt,
    });
  }
}
```

The mapping between Prisma's generated types and domain entities happens **only**
inside this adapter. The domain never sees `@prisma/client`.

---

## Dependency Injection Without a Framework

### Manual DI: Composition Root

The composition root is the single place where all concrete implementations are
chosen and wired together. In Spring, this is roughly what the IoC container does
automatically with `@Autowired`.

```typescript
// main.ts — the composition root

import express from "express";
import { PrismaClient } from "@prisma/client";

// Driven adapters (infrastructure)
import { PrismaOrderRepository } from "./infrastructure/persistence/PrismaOrderRepository";
import { SendGridEmailService } from "./infrastructure/email/SendGridEmailService";
import { StripePaymentGateway } from "./infrastructure/payment/StripePaymentGateway";

// Application layer (use cases)
import { PlaceOrderService } from "./application/use-cases/PlaceOrderService";

// Driving adapters (HTTP)
import { createOrderRoutes } from "./adapters/http/express/OrderRoutes";

async function main(): Promise<void> {
  // 1. Create infrastructure (driven adapters)
  const prisma = new PrismaClient();
  const orderRepo = new PrismaOrderRepository(prisma);
  const emailService = new SendGridEmailService(process.env.SENDGRID_API_KEY!);
  const paymentGateway = new StripePaymentGateway(process.env.STRIPE_SECRET!);

  // 2. Create use cases, injecting driven adapters
  const placeOrder = new PlaceOrderService(
    orderRepo,
    paymentGateway,
    emailService
  );

  // 3. Create driving adapters, injecting use cases
  const app = express();
  app.use(express.json());
  app.use("/api", createOrderRoutes(placeOrder));

  app.listen(3000, () => console.log("Server running on :3000"));
}

main();
```

**Swapping to Fastify** means changing only the driving adapter section:

```typescript
// main.ts — Fastify variant (only section 3 changes)

import Fastify from "fastify";
import { registerOrderRoutes } from "./adapters/http/fastify/OrderRoutes";

// ... sections 1 and 2 are identical ...

const app = Fastify();
registerOrderRoutes(app, placeOrder);
app.listen({ port: 3000 });
```

### NestJS: Framework-Managed DI

NestJS provides a DI container similar to Spring. Same hexagonal idea, different
wiring mechanism.

```typescript
// In NestJS, the module acts as the composition root

@Module({
  providers: [
    // Bind the interface token to the concrete implementation
    {
      provide: "OrderRepository",
      useClass: PrismaOrderRepository,
    },
    {
      provide: "PaymentGateway",
      useClass: StripePaymentGateway,
    },
    {
      provide: "EmailService",
      useClass: SendGridEmailService,
    },
    PlaceOrderService,
  ],
  controllers: [OrderController],
})
export class OrderModule {}

// Use case injects by token (similar to Spring @Qualifier)
@Injectable()
export class PlaceOrderService implements PlaceOrderUseCase {
  constructor(
    @Inject("OrderRepository") private readonly orderRepo: OrderRepository,
    @Inject("PaymentGateway") private readonly paymentGateway: PaymentGateway,
    @Inject("EmailService") private readonly emailService: EmailService
  ) {}
}
```

---

## Testing Advantage

The architecture gives you three independent test layers, each runnable without
the layers above or below.

### Layer 1: Domain Tests (Pure Logic, No Mocks)

```typescript
// domain/entities/__tests__/Order.test.ts

import { describe, it, expect } from "vitest";
import { Order } from "../Order";
import { Money } from "../../value-objects/Money";
import { OrderStatus } from "../../value-objects/OrderStatus";

describe("Order", () => {
  const sampleItems = [
    { productId: "p1", quantity: 2, unitPrice: Money.of(1000, "USD") },
  ];

  it("rejects empty item lists", () => {
    expect(() => Order.create("id-1", "cust-1", [])).toThrow(
      "at least one item"
    );
  });

  it("calculates total from line items", () => {
    const order = Order.create("id-1", "cust-1", sampleItems);
    expect(order.total).toEqual(Money.of(2000, "USD"));
  });

  it("transitions from PENDING to CONFIRMED", () => {
    const order = Order.create("id-1", "cust-1", sampleItems);
    const confirmed = order.confirm();
    expect(confirmed.status).toBe(OrderStatus.CONFIRMED);
  });

  it("prevents confirming a non-pending order", () => {
    const order = Order.create("id-1", "cust-1", sampleItems);
    const confirmed = order.confirm();
    expect(() => confirmed.confirm()).toThrow("Cannot confirm");
  });
});
```

No mocks, no setup, no database. These tests run in milliseconds.

### Layer 2: Use-Case Tests (Fake Adapters)

```typescript
// application/use-cases/__tests__/PlaceOrderService.test.ts

import { describe, it, expect } from "vitest";
import { PlaceOrderService } from "../PlaceOrderService";
import { InMemoryOrderRepository } from "./fakes/InMemoryOrderRepository";
import { FakePaymentGateway } from "./fakes/FakePaymentGateway";
import { FakeEmailService } from "./fakes/FakeEmailService";

describe("PlaceOrderService", () => {
  function buildService() {
    const orderRepo = new InMemoryOrderRepository();
    const paymentGateway = new FakePaymentGateway();
    const emailService = new FakeEmailService();

    const service = new PlaceOrderService(
      orderRepo,
      paymentGateway,
      emailService
    );

    return { service, orderRepo, paymentGateway, emailService };
  }

  it("persists a confirmed order after successful payment", async () => {
    const { service, orderRepo, paymentGateway } = buildService();
    paymentGateway.willSucceed();

    await service.execute({
      customerId: "cust-1",
      items: [{ productId: "p1", quantity: 1 }],
    });

    const saved = await orderRepo.findByCustomerId("cust-1");
    expect(saved).toHaveLength(1);
    expect(saved[0].status).toBe("CONFIRMED");
  });

  it("throws when payment fails", async () => {
    const { service, paymentGateway } = buildService();
    paymentGateway.willFail("Insufficient funds");

    await expect(
      service.execute({
        customerId: "cust-1",
        items: [{ productId: "p1", quantity: 1 }],
      })
    ).rejects.toThrow("Insufficient funds");
  });

  it("sends confirmation email after order placed", async () => {
    const { service, paymentGateway, emailService } = buildService();
    paymentGateway.willSucceed();

    await service.execute({
      customerId: "cust-1",
      items: [{ productId: "p1", quantity: 1 }],
    });

    expect(emailService.sentEmails).toHaveLength(1);
    expect(emailService.sentEmails[0].subject).toBe("Order Confirmed");
  });
});
```

A fake adapter is trivial to write:

```typescript
// application/use-cases/__tests__/fakes/InMemoryOrderRepository.ts

import { Order } from "../../../../domain/entities/Order";
import { OrderRepository } from "../../../../domain/ports/driven/OrderRepository";

export class InMemoryOrderRepository implements OrderRepository {
  private readonly orders = new Map<string, Order>();

  async findById(id: string): Promise<Order | null> {
    return this.orders.get(id) ?? null;
  }

  async findByCustomerId(customerId: string): Promise<readonly Order[]> {
    return [...this.orders.values()].filter(
      (o) => o.customerId === customerId
    );
  }

  async save(order: Order): Promise<Order> {
    this.orders.set(order.id, order);
    return order;
  }

  async delete(id: string): Promise<void> {
    this.orders.delete(id);
  }
}
```

### Layer 3: Integration Tests (Real Adapters)

```typescript
// infrastructure/persistence/__tests__/PrismaOrderRepository.integration.test.ts

import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { PrismaClient } from "@prisma/client";
import { PrismaOrderRepository } from "../PrismaOrderRepository";
import { Order } from "../../../domain/entities/Order";
import { Money } from "../../../domain/value-objects/Money";

describe("PrismaOrderRepository", () => {
  let prisma: PrismaClient;
  let repo: PrismaOrderRepository;

  beforeAll(async () => {
    prisma = new PrismaClient(); // points to test database
    repo = new PrismaOrderRepository(prisma);
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  it("round-trips an order through the database", async () => {
    const order = Order.create("test-1", "cust-1", [
      { productId: "p1", quantity: 3, unitPrice: Money.of(500, "USD") },
    ]);

    await repo.save(order);
    const loaded = await repo.findById("test-1");

    expect(loaded).not.toBeNull();
    expect(loaded!.customerId).toBe("cust-1");
    expect(loaded!.items).toHaveLength(1);
  });
});
```

Integration tests verify the adapter-to-infrastructure boundary. They need a real
database but do not need an HTTP server.

---

## Common Mistakes

### 1. Prisma Types Leaking into the Domain

```typescript
// BAD: Domain entity importing from @prisma/client
import { Order as PrismaOrder } from "@prisma/client";

export class OrderService {
  async process(order: PrismaOrder): Promise<void> { /* ... */ }
}
```

The domain now depends on Prisma's schema. Switch to Drizzle and the domain breaks.
Fix: the adapter maps Prisma types to domain types. The domain declares its own types.

### 2. Express `req`/`res` in Use Cases

```typescript
// BAD: Use case knows about Express
export class PlaceOrderService {
  async execute(req: Request, res: Response): Promise<void> {
    const order = Order.create(req.body.id, req.body.customerId, req.body.items);
    // ...
    res.status(201).json(order);
  }
}
```

This use case cannot be called from a CLI, a queue consumer, or a test without
faking Express objects. Fix: the use case accepts a plain command object and returns
a plain result.

### 3. Anemic Domain Model

```typescript
// BAD: Entity is just a data bag, all logic lives in services
export class Order {
  id: string;
  status: string;
  items: OrderItem[];
}

export class OrderService {
  confirm(order: Order): void {
    if (order.status !== "PENDING") throw new Error("...");
    order.status = "CONFIRMED"; // mutation + logic outside entity
  }
}
```

This is a procedural program wearing OOP clothing. The entity should own its state
transitions (as shown in the domain entities section above).

### 4. Over-Abstracting Simple CRUD

Not every endpoint needs ports and adapters. A straightforward read-only endpoint
that queries a table and returns rows does not benefit from three layers of
abstraction. Apply hexagonal architecture where there is real business logic worth
protecting. For simple CRUD, a thin service calling the ORM directly is fine.

---

## Java Parallel: Spring vs True Hexagonal

Spring's layered architecture looks close to hexagonal:

```
Controller  ->  Service  ->  Repository
(adapter)      (domain?)     (adapter?)
```

But there are key differences:

| Aspect | Typical Spring | True Hexagonal |
|--------|---------------|----------------|
| Service depends on | `JpaRepository<Order, Long>` concrete interface | `OrderRepository` (your own interface) |
| Entity definition | `@Entity` with JPA annotations | Plain class, no ORM annotations |
| Persistence mapping | Lives on the entity itself | Lives in the adapter |
| Framework coupling | `@Service`, `@Transactional` on business logic | Business logic is framework-free |
| Swapping data layer | Rewrite entity + service | Write a new adapter, nothing else changes |

### True Hexagonal in Spring

```java
// Domain port — no Spring annotations
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

// Infrastructure adapter — Spring lives here
@Repository
@Qualifier("jpa")
public class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderRepo springRepo;

    // Maps between JPA entity and domain entity
}

// Domain service — no @Service, no @Transactional
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository orderRepo; // injected via interface
}
```

The effort is similar in both languages. The payoff is the same: the domain core
has zero framework dependencies, and you can test it or swap infrastructure without
touching business logic.

---

## When to Use Hexagonal Architecture

**Good fit**:
- Applications with complex business rules (finance, logistics, e-commerce)
- Systems that need to support multiple entry points (HTTP + CLI + queue consumers)
- Projects where the infrastructure may change (database migration, cloud provider swap)
- Teams practicing domain-driven design

**Overkill**:
- Simple REST wrappers around a database
- Prototypes and throwaway MVPs
- CRUD-heavy applications with minimal business logic
- Single-developer side projects under time pressure

The architecture is a tool, not a religion. Use it where the coupling protection
pays for the indirection cost.

---

## Related

- [Spring Boot Project Structure](../../java/architecture/project-structure.md) — layered, package-by-feature, and hexagonal structure on the Java side.
- [Spring Data Repository Interfaces](../../java/data-repositories/repository-interfaces.md) — a contrasting style where infrastructure repositories are generated by the framework.
- [Prisma Deep Dive](../data-access/prisma-deep-dive.md) — how to keep Prisma inside adapters instead of your domain core.

## References

- Alistair Cockburn, "Hexagonal Architecture" (original 2005 article): https://alistair.cockburn.us/hexagonal-architecture/
- "Ports and Adapters Pattern" on the Netflix Tech Blog
- Martin Fowler, "Patterns of Enterprise Application Architecture" (Repository pattern context)
- NestJS Custom Providers documentation: https://docs.nestjs.com/fundamentals/custom-providers
- Spring Framework Reference, "Using @Qualifier": https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html
- Vaughn Vernon, "Implementing Domain-Driven Design" (Ch. 4, Architecture)
