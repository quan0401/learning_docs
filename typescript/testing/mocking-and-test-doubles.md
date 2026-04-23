---
title: "Mocking & Test Doubles in Vitest"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, testing, vitest, mocking]
---

# Mocking & Test Doubles in Vitest

> Test a service that calls Prisma and an external API — without a database or network.

| Field        | Value                                                |
| ------------ | ---------------------------------------------------- |
| Prerequisite | [Vitest Fundamentals](vitest-fundamentals.md)        |
| Audience     | TS/Node backend developer with Java/Mockito background |
| Stack        | Express, Prisma, Vitest                              |

You already know Mockito inside-out: `mock()`, `when().thenReturn()`, `verify()`,
`@MockBean`. You understand *why* you mock, but every test framework spells
it differently. This document maps the entire Mockito mental model onto Vitest
so you can start writing isolated tests for your Express + Prisma services today.

---

## 1. Test Doubles Taxonomy

Before touching APIs, nail the vocabulary. The terminology comes from Gerard
Meszaros's *xUnit Test Patterns* and applies regardless of language.

| Double   | Purpose                                   | Vitest API                     | Mockito Equivalent            |
| -------- | ----------------------------------------- | ------------------------------ | ----------------------------- |
| **Dummy**  | Fills a parameter slot, never called      | `{} as SomeType`              | `mock(SomeType.class)` unused |
| **Stub**   | Returns canned data, no behavior check    | `vi.fn().mockReturnValue(x)`   | `when(m.get()).thenReturn(x)` |
| **Spy**    | Wraps a real object, records calls        | `vi.spyOn(obj, 'method')`     | `Mockito.spy(real)`           |
| **Mock**   | Stub + assertion on how it was called     | `vi.fn()` + `toHaveBeenCalled` | `mock()` + `verify()`        |
| **Fake**   | Working implementation (in-memory DB)     | Hand-written class             | Hand-written class            |

### When to Use Each

- **Dummy**: A logger you must pass but never care about in this test.
- **Stub**: The Prisma call returns a fixed user — you are testing the service logic, not the query.
- **Spy**: You want the real implementation to run but need to verify it was called (e.g., audit logging).
- **Mock**: You need to assert the service called the payment gateway with the correct amount.
- **Fake**: An `InMemoryOrderRepository` that stores data in a `Map` — fast, deterministic, reusable across many tests. See [Hexagonal Architecture](../patterns/hexagonal-architecture.md) for the port that makes this possible.

---

## 2. `vi.fn()` Basics

`vi.fn()` creates a mock function — the Vitest equivalent of `Mockito.mock()` for
a single method. It records every call and lets you control return values.

### Creating and Configuring

```typescript
import { describe, it, expect, vi } from 'vitest';

// Basic mock — returns undefined by default
const greet = vi.fn();

// Stub a return value (like when().thenReturn())
greet.mockReturnValue('hello');
expect(greet()).toBe('hello');

// One-shot return (like when().thenReturn(x).thenReturn(y))
greet.mockReturnValueOnce('first').mockReturnValueOnce('second');
expect(greet()).toBe('first');
expect(greet()).toBe('second');
expect(greet()).toBe('hello'); // falls back to mockReturnValue

// Async stubs (like when().thenReturn(CompletableFuture.completedFuture(x)))
const fetchUser = vi.fn();
fetchUser.mockResolvedValue({ id: 1, name: 'Alice' });
fetchUser.mockRejectedValueOnce(new Error('timeout'));
```

### Assertions

```typescript
const save = vi.fn();
save({ id: 1, name: 'Alice' });

// Was it called? (verify(mock).save(any()))
expect(save).toHaveBeenCalled();

// How many times? (verify(mock, times(1)).save(any()))
expect(save).toHaveBeenCalledTimes(1);

// With what arguments? (verify(mock).save(eq(expected)))
expect(save).toHaveBeenCalledWith({ id: 1, name: 'Alice' });

// Nth call inspection
expect(save).toHaveBeenNthCalledWith(1, { id: 1, name: 'Alice' });
```

### Clearing State: `mockClear` vs `mockReset` vs `mockRestore`

This trips up everyone. The three methods remove progressively more state:

| Method          | Clears call history | Clears return config | Restores original impl |
| --------------- | :-----------------: | :------------------: | :--------------------: |
| `mockClear()`   | yes                 | no                   | no                     |
| `mockReset()`   | yes                 | yes                  | no                     |
| `mockRestore()` | yes                 | yes                  | yes (spies only)       |

In practice, use `vi.restoreAllMocks()` in a global `afterEach` (configure in
`vitest.config.ts` with `mockReset: true`) so every test starts clean. This is
the equivalent of Mockito's `@ExtendWith(MockitoExtension.class)` which resets
mocks between tests automatically.

---

## 3. `vi.spyOn()`

A spy wraps a real method. The original implementation still runs unless you
override it. Think `Mockito.spy(realObject)`.

### Spy on `console.log`

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';

afterEach(() => {
  vi.restoreAllMocks();
});

it('logs a warning when balance is low', () => {
  const logSpy = vi.spyOn(console, 'warn');

  checkBalance({ balance: 5, threshold: 100 });

  expect(logSpy).toHaveBeenCalledWith(
    expect.stringContaining('low balance'),
  );
});
```

The real `console.warn` still fires (you'll see the output). If you want silence,
chain `.mockImplementation(() => {})`.

### Spy on a Service Method

```typescript
import { PaymentService } from './payment.service';

it('calls the audit log after payment', async () => {
  const service = new PaymentService(/* deps */);
  const auditSpy = vi.spyOn(service, 'writeAuditLog');

  await service.processPayment({ amount: 50, currency: 'USD' });

  // Real writeAuditLog ran — we just verified it was called
  expect(auditSpy).toHaveBeenCalledWith(
    expect.objectContaining({ amount: 50 }),
  );
});
```

### Override with `mockImplementation`

```typescript
const spy = vi.spyOn(service, 'sendEmail');
spy.mockImplementation(async () => {
  // Replace the real email sender in tests
  return { sent: true, messageId: 'fake-123' };
});
```

This is `doReturn(x).when(spy).method()` in Mockito.

---

## 4. Module Mocking with `vi.mock()`

Module mocking replaces an entire import. This is the Vitest equivalent of
Spring's `@MockBean` replacing a bean in the application context — except it
operates at the module level.

### Hoisting Behavior

`vi.mock()` calls are **hoisted to the top of the file** before any imports.
This is the single most surprising behavior for newcomers:

```typescript
// This runs AFTER vi.mock, even though it appears first in source
import { prisma } from '../lib/prisma';

// This is hoisted to the top at compile time
vi.mock('../lib/prisma');
```

The practical consequence: you cannot use variables declared above `vi.mock()`
inside its factory function. If you need runtime values, use `vi.hoisted()`:

```typescript
const { mockFindUnique } = vi.hoisted(() => ({
  mockFindUnique: vi.fn(),
}));

vi.mock('../lib/prisma', () => ({
  prisma: {
    user: { findUnique: mockFindUnique },
  },
}));
```

### Auto-Mocking vs Manual Factory

```typescript
// Auto-mock: every export becomes vi.fn(), returns undefined
vi.mock('../lib/prisma');

// Manual factory: you control the shape
vi.mock('../lib/prisma', () => ({
  prisma: {
    user: {
      findUnique: vi.fn(),
      create: vi.fn(),
    },
  },
}));
```

Auto-mocking is fragile for complex objects like PrismaClient. Prefer the manual
factory or the dedicated pattern in the next section.

---

## 5. Mocking Prisma Specifically

[Prisma Deep Dive](../data-access/prisma-deep-dive.md) covers the client itself.
Here we focus on isolating it in tests.

### The Singleton Mock Pattern

Create a shared mock module that every test file imports:

```typescript
// src/__mocks__/prisma.ts
import { vi } from 'vitest';
import type { PrismaClient } from '@prisma/client';
import { mockDeep, type DeepMockProxy } from 'vitest-mock-extended';

const prismaMock = mockDeep<PrismaClient>();

vi.mock('../lib/prisma', () => ({
  prisma: prismaMock,
}));

export { prismaMock };
```

`vitest-mock-extended` gives you `mockDeep` — it recursively mocks every nested
property (`prisma.user.findUnique`, `prisma.$transaction`, etc.). Without it,
you would need to manually stub every model and method.

### Manual Approach (No Extra Library)

If you prefer zero dependencies:

```typescript
// test/helpers/prisma-mock.ts
import { vi } from 'vitest';

export function createPrismaMock() {
  return {
    user: {
      findUnique: vi.fn(),
      findMany: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    },
    order: {
      findUnique: vi.fn(),
      create: vi.fn(),
    },
    $transaction: vi.fn((fn) => fn()),
    $disconnect: vi.fn(),
  };
}
```

### Full Example: Testing a UserService

```typescript
// src/services/user.service.ts
import { prisma } from '../lib/prisma';

export interface CreateUserDto {
  readonly email: string;
  readonly name: string;
}

export class UserService {
  async findById(id: string) {
    const user = await prisma.user.findUnique({ where: { id } });
    if (!user) {
      return { ok: false, error: 'USER_NOT_FOUND' } as const;
    }
    return { ok: true, data: user } as const;
  }

  async create(dto: CreateUserDto) {
    return prisma.user.create({ data: dto });
  }
}
```

```typescript
// src/services/__tests__/user.service.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { prismaMock } from '../../__mocks__/prisma';
import { UserService } from '../user.service';

describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    service = new UserService();
  });

  describe('findById', () => {
    it('returns user when found', async () => {
      // Arrange — like when().thenReturn() in Mockito
      const fakeUser = { id: '1', email: 'a@b.com', name: 'Alice' };
      prismaMock.user.findUnique.mockResolvedValue(fakeUser);

      // Act
      const result = await service.findById('1');

      // Assert — Result type from Error Handling Architecture
      expect(result).toEqual({ ok: true, data: fakeUser });
      expect(prismaMock.user.findUnique).toHaveBeenCalledWith({
        where: { id: '1' },
      });
    });

    it('returns error when not found', async () => {
      prismaMock.user.findUnique.mockResolvedValue(null);

      const result = await service.findById('999');

      // See: Error Handling Architecture for the Result pattern
      expect(result).toEqual({ ok: false, error: 'USER_NOT_FOUND' });
    });
  });

  describe('create', () => {
    it('delegates to prisma.user.create', async () => {
      const dto = { email: 'b@c.com', name: 'Bob' };
      const created = { id: '2', ...dto };
      prismaMock.user.create.mockResolvedValue(created);

      const result = await service.create(dto);

      expect(result).toEqual(created);
      expect(prismaMock.user.create).toHaveBeenCalledWith({
        data: dto,
      });
    });
  });
});
```

The `Result` type return (`{ ok, data } | { ok, error }`) comes from the
[Error Handling Architecture](../patterns/error-handling-architecture.md). Testing
these discriminated unions is straightforward — assert `ok` first, then narrow.

### Mocking `$transaction`

```typescript
it('rolls back on failure inside transaction', async () => {
  prismaMock.$transaction.mockImplementation(async (fn) => {
    // Simulate the transaction callback receiving the prisma client
    return fn(prismaMock);
  });

  prismaMock.order.create.mockRejectedValue(new Error('constraint'));

  await expect(service.placeOrder(dto)).rejects.toThrow('constraint');
});
```

---

## 6. Mocking External APIs

### Mock `fetch` Directly

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

describe('PaymentGateway', () => {
  beforeEach(() => {
    global.fetch = vi.fn();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('returns success on 200', async () => {
    const mockResponse = {
      ok: true,
      json: () => Promise.resolve({ transactionId: 'tx-123' }),
    };
    vi.mocked(fetch).mockResolvedValue(mockResponse as Response);

    const result = await gateway.charge({ amount: 1000, currency: 'USD' });

    expect(result).toEqual({ ok: true, data: { transactionId: 'tx-123' } });
    expect(fetch).toHaveBeenCalledWith(
      'https://api.payment.com/charge',
      expect.objectContaining({ method: 'POST' }),
    );
  });

  it('returns error on network failure', async () => {
    vi.mocked(fetch).mockRejectedValue(new Error('ECONNREFUSED'));

    const result = await gateway.charge({ amount: 1000, currency: 'USD' });

    expect(result).toEqual({
      ok: false,
      error: expect.stringContaining('ECONNREFUSED'),
    });
  });
});
```

### Mock `axios`

```typescript
vi.mock('axios');
import axios from 'axios';

vi.mocked(axios.post).mockResolvedValue({
  status: 200,
  data: { transactionId: 'tx-456' },
});
```

### MSW — Mock Service Worker

`vi.mock()` replaces code at the module level. MSW intercepts HTTP requests at
the network level — closer to reality, catches URL typos, and works for
integration tests where you boot a real Express server.

```typescript
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('https://api.payment.com/charge', () => {
    return HttpResponse.json({ transactionId: 'tx-789' });
  }),

  http.get('https://api.payment.com/status/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, status: 'completed' });
  }),
];
```

```typescript
// test/setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './mocks/handlers';

export const server = setupServer(...handlers);
```

```typescript
// vitest.config.ts — wire up msw
export default defineConfig({
  test: {
    setupFiles: ['./test/setup-msw.ts'],
  },
});
```

```typescript
// test/setup-msw.ts
import { beforeAll, afterAll, afterEach } from 'vitest';
import { server } from './setup';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

**When to use which:**

| Approach         | Best for                                       |
| ---------------- | ---------------------------------------------- |
| `vi.mock()`      | Unit tests, fast, module-level replacement     |
| `vi.fn()` on fetch | Simple HTTP stubs, no extra dependency        |
| **MSW**          | Integration tests, shared handler sets, realistic HTTP behavior |

---

## 7. Dependency Injection for Testability

Mocking modules with `vi.mock()` works but tightly couples tests to file paths.
The cleaner approach — and the one you already know from Spring — is constructor
injection through interfaces.

See [Hexagonal Architecture](../patterns/hexagonal-architecture.md) for the full
ports & adapters pattern. Here is the testing payoff:

### Port (Interface)

```typescript
// src/domain/ports/order-repository.port.ts
export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<Order>;
}
```

This is the port. [DDD Tactical Patterns](../patterns/ddd-tactical-patterns.md)
covers why the domain defines the interface, not the infrastructure.

### Production Adapter

```typescript
// src/infrastructure/prisma-order.repository.ts
import { prisma } from '../lib/prisma';
import type { OrderRepository } from '../domain/ports/order-repository.port';

export class PrismaOrderRepository implements OrderRepository {
  async findById(id: string) {
    return prisma.order.findUnique({ where: { id } });
  }

  async save(order: Order) {
    return prisma.order.create({ data: order });
  }
}
```

### Fake for Tests

```typescript
// src/infrastructure/__fakes__/in-memory-order.repository.ts
import type { OrderRepository } from '../../domain/ports/order-repository.port';

export class InMemoryOrderRepository implements OrderRepository {
  private readonly store = new Map<string, Order>();

  async findById(id: string) {
    return this.store.get(id) ?? null;
  }

  async save(order: Order) {
    const saved = { ...order, id: order.id ?? crypto.randomUUID() };
    this.store.set(saved.id, saved);
    return saved;
  }
}
```

### The Test

```typescript
describe('OrderService', () => {
  it('calculates total with discount', async () => {
    const repo = new InMemoryOrderRepository();
    const service = new OrderService(repo); // constructor injection

    const order = await service.placeOrder({
      items: [{ sku: 'A', price: 100, qty: 2 }],
      discountCode: 'SAVE10',
    });

    expect(order.total).toBe(180); // 200 - 10%
  });
});
```

No `vi.mock()`. No module path coupling. The test reads like a specification.

### Spring Parallel

| Spring                              | TypeScript + Vitest                        |
| ----------------------------------- | ------------------------------------------ |
| `@MockBean OrderRepository repo`    | `const repo = new InMemoryOrderRepository()` |
| `@Autowired` constructor injection  | Constructor parameter                      |
| `when(repo.findById(x)).thenReturn` | Fake implements the interface directly     |
| Application context wiring          | Manual `new Service(dep1, dep2)`           |

The TypeScript version is more explicit and has zero framework magic. You trade
some verbosity for total transparency — every dependency is visible in the
constructor.

---

## 8. Common Mocking Mistakes

### Over-Mocking

```typescript
// BAD — mocking the thing you're testing
vi.spyOn(service, 'calculateTotal').mockReturnValue(100);
expect(service.calculateTotal(items)).toBe(100); // Tests nothing
```

If you mock the function under test, the test verifies your mock, not your code.
Mock dependencies, not the subject.

### Mocking Implementation Details

```typescript
// FRAGILE — breaks if you rename the internal method
expect(service['_internalValidate']).toHaveBeenCalled();
```

Test behavior (what comes out), not mechanics (what methods ran internally). If
a refactor changes internals but preserves behavior, tests should still pass.

### Forgetting to Restore Mocks

```typescript
// Test A
vi.spyOn(Date, 'now').mockReturnValue(1000);

// Test B — still gets 1000 if mocks aren't restored!
```

Always configure `restoreAllMocks: true` in `vitest.config.ts`:

```typescript
export default defineConfig({
  test: {
    restoreAllMocks: true,
  },
});
```

### Mocking What You Don't Own

Mock what you *own* (your repository interface), not what you *don't own*
(Prisma's internal query engine). If Prisma changes its internal API, your
mocks silently diverge from reality.

The fix: define a port interface you control ([Hexagonal Architecture](../patterns/hexagonal-architecture.md)),
mock that interface, and let integration tests verify the real Prisma adapter.

---

## 9. Mockito to Vitest Cheat Sheet

A quick-reference for the muscle memory translation:

| Concept                  | Mockito (Java)                              | Vitest (TypeScript)                           |
| ------------------------ | ------------------------------------------- | --------------------------------------------- |
| Create a mock            | `Mockito.mock(UserRepo.class)`              | `vi.fn()` or `mockDeep<UserRepo>()`           |
| Stub return value        | `when(repo.find(id)).thenReturn(user)`      | `repo.find.mockResolvedValue(user)`           |
| Stub throw               | `when(repo.find(id)).thenThrow(new Ex())`   | `repo.find.mockRejectedValue(new Error())`    |
| Verify called            | `verify(repo).find(id)`                     | `expect(repo.find).toHaveBeenCalledWith(id)`  |
| Verify call count        | `verify(repo, times(2)).find(any())`        | `expect(repo.find).toHaveBeenCalledTimes(2)`  |
| Verify never called      | `verify(repo, never()).delete(any())`       | `expect(repo.delete).not.toHaveBeenCalled()`  |
| Argument capture         | `ArgumentCaptor<Order>`                     | `repo.save.mock.calls[0][0]`                  |
| Spy (real + recording)   | `Mockito.spy(realService)`                  | `vi.spyOn(realService, 'method')`             |
| Reset                    | `Mockito.reset(mock)`                       | `mock.mockReset()`                            |
| Mock an injected bean    | `@MockBean` in test context                 | `vi.mock('./module')` or constructor injection |
| Ordered verification     | `InOrder inOrder = inOrder(a, b)`           | Check `mock.calls` array order manually       |
| Argument matchers        | `any()`, `eq()`, `argThat()`                | `expect.any(String)`, `expect.objectContaining()` |

### Argument Capture Deep Dive

Mockito's `ArgumentCaptor` is explicit. In Vitest, reach into `mock.calls`:

```typescript
const save = vi.fn();
save({ id: '1', total: 180, currency: 'USD' });

// Capture the first argument of the first call
const captured = save.mock.calls[0][0];
expect(captured.total).toBe(180);
expect(captured.currency).toBe('USD');
```

For multiple calls:

```typescript
const allOrders = save.mock.calls.map(([order]) => order);
expect(allOrders).toHaveLength(3);
expect(allOrders.every((o) => o.currency === 'USD')).toBe(true);
```

---

## Related

- [Vitest Fundamentals](vitest-fundamentals.md) — test runner setup, assertions, lifecycle hooks
- [Integration & API Testing](integration-and-api-testing.md) — supertest, database fixtures, end-to-end flows
- [Hexagonal Architecture](../patterns/hexagonal-architecture.md) — ports & adapters that make DI-based mocking natural
- [DDD Tactical Patterns](../patterns/ddd-tactical-patterns.md) — repository interface pattern enables mock repositories
- [Prisma Deep Dive](../data-access/prisma-deep-dive.md) — the PrismaClient we mock throughout this doc
- [Error Handling Architecture](../patterns/error-handling-architecture.md) — the `Result` type tested in section 5

---

## References

- [Vitest — Mocking](https://vitest.dev/guide/mocking.html) — official vi.fn, vi.mock, vi.spyOn docs
- [Vitest — vi API](https://vitest.dev/api/vi.html) — complete vi utility reference
- [vitest-mock-extended](https://github.com/eratio08/vitest-mock-extended) — `mockDeep` for Prisma and other complex types
- [MSW — Getting Started](https://mswjs.io/docs/getting-started) — Mock Service Worker setup guide
- [MSW — Node Integration](https://mswjs.io/docs/integrations/node) — server-side request interception for Vitest
- [Prisma — Testing Guide](https://www.prisma.io/docs/guides/testing/unit-testing) — official Prisma unit testing patterns
- [xUnit Test Patterns](http://xunitpatterns.com/Test%20Double.html) — Meszaros's original test double taxonomy
