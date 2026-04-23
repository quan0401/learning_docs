---
title: "Integration & API Testing"
date: 2026-04-23
updated: 2026-04-23
tags: [typescript, testing, vitest, integration, supertest]
---

# Integration & API Testing

| | |
|---|---|
| **Audience** | Intermediate TS/Node backend developer with Java testing background |
| **Stack** | Express + Prisma + PostgreSQL |
| **Prerequisite** | [Vitest Fundamentals](vitest-fundamentals.md) |
| **Problem** | "Test an endpoint end-to-end without mocking the database." |

---

## 1. Integration vs Unit vs E2E — The Testing Spectrum

```
Unit                    Integration                  E2E
Mock everything         Real dependencies,           Full system
Fast, isolated          isolated scope               Slow, realistic
────────────────────────────────────────────────────────────────►
Low confidence/test     HIGH confidence/cost         Highest confidence/test
```

### The Testing Trophy (Not Pyramid)

For **backend services**, the testing trophy replaces the traditional pyramid:

```
        ╱ E2E ╲           ← Few: critical happy paths only
       ╱────────╲
      ╱Integration╲       ← MOST TESTS HERE
     ╱──────────────╲
    ╱   Unit Tests    ╲    ← Pure logic, utilities
   ╱────────────────────╲
  ╱   Static Analysis     ╲ ← TypeScript + ESLint (free)
```

A unit test for `createUser` with a mocked repository proves almost nothing — the real bugs live in the SQL, the Prisma query, the HTTP serialization. An integration test hitting a real database catches all of these.

| Level | Use When | Skip When |
|-------|----------|-----------|
| **Unit** | Pure functions, utilities, type transformers | Thin pass-through to a database |
| **Integration** | Routes, services with DB, middleware chains, auth flows | Logic has no external dependencies |
| **E2E** | Critical journeys (signup, checkout, payment) | Every CRUD variation |

> **Java parallel:** `@WebMvcTest` (unit-ish) vs `@SpringBootTest` + Testcontainers (integration) vs Selenium (e2e). The trophy says: spend most effort on `@SpringBootTest`-style tests.

See [Mocking & Test Doubles](mocking-and-test-doubles.md) for when mocking is the better choice.

---

## 2. Testing Express Routes with Supertest

Key insight: **create the app without `app.listen()`**, pass it to `request()`.

```bash
pnpm add -D supertest @types/supertest
```

### App Factory Pattern

```typescript
// src/app.ts — create app without starting server
export function createApp(deps: { prisma: PrismaClient }) {
  const app = express();
  app.use(express.json());
  app.use('/users', userRouter(deps));
  app.use(errorHandler);
  return app;
}
```

```typescript
// src/server.ts — startup (never imported in tests)
const app = createApp({ prisma });
app.listen(3000);
```

This aligns with [Hexagonal Architecture](../patterns/hexagonal-architecture.md) — the factory accepts dependencies (ports), and tests inject real or fake adapters.

### Route Tests

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import request from 'supertest';
import { createApp } from '../../app';
import { prisma } from '../../lib/prisma';

const app = createApp({ prisma });

describe('POST /users', () => {
  afterEach(() => prisma.user.deleteMany());

  it('creates a user and returns 201', async () => {
    const res = await request(app)
      .post('/users')
      .send({ email: 'test@example.com', name: 'Test User' })
      .expect(201);

    expect(res.body).toMatchObject({
      id: expect.any(String),
      email: 'test@example.com',
    });
    // Verify it actually hit the database
    const dbUser = await prisma.user.findUnique({ where: { email: 'test@example.com' } });
    expect(dbUser).not.toBeNull();
  });

  it('returns 422 for invalid email', async () => {
    await request(app)
      .post('/users')
      .send({ email: 'not-an-email', name: 'Test' })
      .expect(422);
  });
});

describe('GET /users/:id', () => {
  it('returns 200 for existing user', async () => {
    const user = await prisma.user.create({ data: { email: 'a@b.com', name: 'A' } });
    const res = await request(app).get(`/users/${user.id}`).expect(200);
    expect(res.body.email).toBe('a@b.com');
  });

  it('returns 404 for non-existent user', async () => {
    await request(app).get('/users/00000000-0000-0000-0000-000000000000').expect(404);
  });
});
```

### Testing Authentication

```typescript
describe('GET /users/me', () => {
  it('returns 401 without token', async () => {
    await request(app).get('/users/me').expect(401);
  });

  it('returns user with valid token', async () => {
    const token = await createTestToken({ userId: testUser.id });
    const res = await request(app)
      .get('/users/me')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
    expect(res.body.id).toBe(testUser.id);
  });

  it('returns 403 for insufficient permissions', async () => {
    const token = await createTestToken({ userId: testUser.id, role: 'USER' });
    await request(app)
      .delete('/users/other-id')
      .set('Authorization', `Bearer ${token}`)
      .expect(403);
  });
});
```

> **Java parallel:** `request(app).get(path).set('Authorization', ...)` maps to `mockMvc.perform(get(path).header("Authorization", ...))`. The `.expect(200)` chain parallels `.andExpect(status().isOk())`.

For framework-specific testing (Fastify's `inject()`, Hono's test client), see [Express vs Fastify vs Hono vs Elysia](../frameworks/express-vs-alternatives.md).

---

## 3. Database Testing Strategies

### Strategy A: Real Database with Testcontainers (Recommended)

Same library you know from Java, different language binding.

```bash
pnpm add -D @testcontainers/postgresql testcontainers
```

```typescript
// test/setup/testcontainers.ts
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { execSync } from 'child_process';

let container: StartedPostgreSqlContainer;
let prisma: PrismaClient;

export async function setupTestDatabase(): Promise<PrismaClient> {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('testdb')
    .withUsername('test')
    .withPassword('test')
    .start();

  const databaseUrl = container.getConnectionUri();
  execSync(`DATABASE_URL="${databaseUrl}" npx prisma migrate deploy`, {
    env: { ...process.env, DATABASE_URL: databaseUrl },
  });

  prisma = new PrismaClient({ datasources: { db: { url: databaseUrl } } });
  await prisma.$connect();
  return prisma;
}

export async function teardownTestDatabase(): Promise<void> {
  await prisma.$disconnect();
  await container.stop();
}
```

```typescript
// test/integration/user.integration.test.ts
let app: Express.Application;

beforeAll(async () => {
  const testPrisma = await setupTestDatabase();
  app = createApp({ prisma: testPrisma });
}, 60_000); // Container startup timeout

afterAll(() => teardownTestDatabase());
afterEach(() => prisma.user.deleteMany());
```

> **Java parallel:** Identical to `@Testcontainers` + `@Container` + `@DynamicPropertySource`. Container lifecycle is the same; only wiring syntax differs.

For Prisma migration details, see [Prisma Deep Dive](../data-access/prisma-deep-dive.md).

### Strategy B: Real Database with Test Schema

Skip container overhead when Postgres is already running:

```typescript
// test/setup/global-setup.ts
export async function setup() {
  execSync(`DATABASE_URL="${TEST_DB_URL}" npx prisma migrate reset --force`, {
    env: { ...process.env, DATABASE_URL: TEST_DB_URL },
  });
}
```

Faster startup (~1-2s vs ~5-15s), but requires a running Postgres instance and careful parallelism configuration.

### Strategy C: In-Memory / SQLite

```prisma
// prisma/schema.test.prisma
datasource db {
  provider = "sqlite"
  url      = "file:./test.db"
}
```

**Limitation:** No `Json`, no arrays, no `@db.Uuid`, no `ILIKE`. Only viable for simple CRUD with scalar columns.

> **Java parallel:** This is `@DataJpaTest` with H2. Same tradeoff — fast but dangerous feature gaps. Most teams eventually move to Testcontainers.

### Comparison

| | Testcontainers | Test Schema | SQLite |
|---|---|---|---|
| **Feature parity** | Exact | Exact | Partial |
| **Startup** | ~5-15s | ~1-2s | Instant |
| **Docker needed** | Yes | No | No |
| **Confidence** | Highest | High | Medium |
| **Best for** | CI / final verification | Local dev | Prototype only |

---

## 4. Fixture Patterns

### Factory Functions

```typescript
// test/factories/user.factory.ts
import { faker } from '@faker-js/faker';

export function buildUserData(overrides: Partial<UserInput> = {}): UserInput {
  return {
    email: faker.internet.email(),
    name: faker.person.fullName(),
    role: 'USER',
    ...overrides,
  };
}

export async function createTestUser(prisma: PrismaClient, overrides = {}): Promise<User> {
  return prisma.user.create({ data: buildUserData(overrides) });
}
```

### Builder Pattern for Complex Entities

```typescript
export class TestOrderBuilder {
  private data: Partial<OrderCreateInput> = {};
  private items: OrderItemInput[] = [];

  constructor(private readonly prisma: PrismaClient) {}

  withUser(userId: string): this { this.data.userId = userId; return this; }
  withStatus(status: OrderStatus): this { this.data.status = status; return this; }
  withItem(product: string, qty: number, price: number): this {
    this.items.push({ product, quantity: qty, unitPrice: price });
    return this;
  }

  async build(): Promise<Order> {
    const userId = this.data.userId ?? (await createTestUser(this.prisma)).id;
    return this.prisma.order.create({
      data: {
        userId,
        status: this.data.status ?? 'PENDING',
        total: this.items.reduce((sum, i) => sum + i.quantity * i.unitPrice, 0),
        items: { create: this.items },
      },
      include: { items: true },
    });
  }
}

// Usage:
const order = await new TestOrderBuilder(prisma)
  .withUser(testUser.id)
  .withItem('Widget', 2, 29.99)
  .build();
```

### Cleanup

```typescript
// Option 1: deleteMany (simple, most common in Node)
afterEach(async () => {
  await prisma.orderItem.deleteMany(); // children first
  await prisma.order.deleteMany();
  await prisma.user.deleteMany();
});
```

Transaction rollback is trickier with Prisma than JPA's `@Transactional` automatic rollback. The `deleteMany` approach is standard in the Node ecosystem.

> **Java parallel:** Factory functions = `@Sql` + `TestEntityManager`. Builder pattern is identical. `deleteMany` cleanup parallels `@DirtiesContext`.

---

## 5. Testing Middleware

### Auth Middleware in Isolation

```typescript
function createMockReqRes(headers: Record<string, string> = {}) {
  const req = { headers, user: undefined } as unknown as Request;
  const res = { status: vi.fn().mockReturnThis(), json: vi.fn().mockReturnThis() } as unknown as Response;
  const next = vi.fn() as NextFunction;
  return { req, res, next };
}

describe('requireAuth', () => {
  it('calls next() with valid token', async () => {
    const { req, res, next } = createMockReqRes({ authorization: `Bearer ${validToken}` });
    await requireAuth(req, res, next);
    expect(next).toHaveBeenCalledOnce();
  });

  it('returns 401 without header', async () => {
    const { req, res, next } = createMockReqRes();
    await requireAuth(req, res, next);
    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });
});
```

### Error Handler

Test the error middleware from [Error Handling Architecture](../patterns/error-handling-architecture.md) by invoking it directly:

```typescript
describe('errorHandler', () => {
  it('maps NotFoundError to 404', () => {
    const res = mockRes();
    errorHandler(new NotFoundError('User not found'), {} as Request, res, vi.fn());
    expect(res.status).toHaveBeenCalledWith(404);
  });

  it('maps unknown errors to 500 without leaking details', () => {
    const res = mockRes();
    errorHandler(new Error('db pool exhausted'), {} as Request, res, vi.fn());
    expect(res.status).toHaveBeenCalledWith(500);
    expect(res.json).toHaveBeenCalledWith(
      expect.objectContaining({ error: { code: 'INTERNAL_ERROR', message: 'Internal server error' } }),
    );
  });
});
```

See [Mocking & Test Doubles](mocking-and-test-doubles.md) for more middleware mocking techniques.

---

## 6. Test Organization

### File Layout

```
# Recommended: separate directory for mixed unit/integration
src/routes/user.router.ts
test/
├── unit/middleware/auth.middleware.test.ts
├── integration/routes/user.routes.test.ts
└── setup/testcontainers.ts
```

### Vitest Workspace

Run unit and integration tests with different configs:

```typescript
// vitest.workspace.ts
export default defineWorkspace([
  {
    test: {
      name: 'unit',
      include: ['src/**/*.test.ts'],
    },
  },
  {
    test: {
      name: 'integration',
      include: ['test/integration/**/*.test.ts'],
      globalSetup: ['test/setup/global-setup.ts'],
      pool: 'forks',
      testTimeout: 30_000,
      hookTimeout: 60_000,
    },
  },
]);
```

```bash
pnpm vitest --project unit          # Fast, no Docker
pnpm vitest --project integration   # Needs Docker
pnpm vitest                         # All
```

> **Java parallel:** Mirrors Maven Surefire (`*Test.java`) vs Failsafe (`*IT.java`). The workspace config is the equivalent of separate plugin configurations.

Health check endpoints from [Node.js in Kubernetes](../production/nodejs-in-kubernetes.md) are excellent integration test candidates.

---

## 7. CI/CD Integration

### GitHub Actions Workflow

```yaml
name: Test
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm vitest --project unit --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: unit-coverage, path: coverage/ }

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      - run: pnpm vitest --project integration --coverage
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
```

**Key practices:**
- Separate jobs for unit (fast feedback) and integration (slower, needs Docker)
- `--frozen-lockfile` prevents accidental dependency changes
- Health checks on Postgres ensure readiness before tests
- Alternatively, use Testcontainers in CI (no `services` block needed — Docker is available on `ubuntu-latest` by default)

---

## 8. Java Spring Boot Parallel — Quick Reference

| Concept | Spring Boot / Java | Express + Vitest / TypeScript |
|---|---|---|
| **HTTP testing** | `MockMvc` / `WebTestClient` | `supertest` + `request(app)` |
| **Full app test** | `@SpringBootTest` | `createApp({ prisma })` with real DB |
| **Slice test** | `@WebMvcTest`, `@DataJpaTest` | Unit test with mocked deps |
| **Container DB** | `@Testcontainers` + `@Container` | `@testcontainers/postgresql` |
| **In-memory DB** | H2 | SQLite with Prisma |
| **Fixtures** | `@Sql`, `TestEntityManager` | Factory functions, `faker.js` |
| **Cleanup** | `@Transactional` rollback | `deleteMany` in `afterEach` |
| **Test split** | Surefire / Failsafe | Vitest workspace projects |
| **Auth testing** | `@WithMockUser` | Custom `createTestToken()` |

### Key Mindset Shifts

1. **No annotation magic.** You wire container setup yourself — but understand every piece.
2. **No `@Transactional` rollback.** Explicit `deleteMany` is more honest about real commit behavior.
3. **App factory replaces DI container.** `createApp(deps)` is manual but transparent.
4. **Vitest is faster.** Watch mode with HMR makes feedback tighter than Maven/Gradle cycles.

---

## Related

- [Vitest Fundamentals](vitest-fundamentals.md) — test runner setup, assertions, lifecycle hooks
- [Mocking & Test Doubles](mocking-and-test-doubles.md) — when and how to mock instead of integrate
- [Prisma Deep Dive](../data-access/prisma-deep-dive.md) — migrations, connection pooling, transaction API
- [Hexagonal Architecture](../patterns/hexagonal-architecture.md) — adapter pattern for clean integration tests
- [Error Handling Architecture](../patterns/error-handling-architecture.md) — error types that map to HTTP status codes
- [Express vs Fastify vs Hono vs Elysia](../frameworks/express-vs-alternatives.md) — framework-specific testing APIs
- [Node.js in Kubernetes](../production/nodejs-in-kubernetes.md) — health check endpoints worth integration testing

---

## References

- [Supertest](https://github.com/ladislav-zezula/supertest) — HTTP assertions for Node.js
- [Testcontainers for Node.js](https://node.testcontainers.org/) — Docker container management for tests
- [Faker.js](https://fakerjs.dev/) — realistic test data generation
- [Vitest Workspace](https://vitest.dev/guide/workspace.html) — multi-project test configuration
- [Prisma Testing Guide](https://www.prisma.io/docs/orm/prisma-client/testing) — official testing docs
- [Kent C. Dodds — Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests) — the testing trophy
