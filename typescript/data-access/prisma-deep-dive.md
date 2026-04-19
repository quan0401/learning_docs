# Prisma Deep Dive

> Problem frame: Handle a complex multi-table transaction with optimistic locking.

Prisma is a type-safe ORM for Node.js and TypeScript that generates a fully typed client from a declarative schema. This guide covers schema design through production deployment patterns, with JPA parallels throughout for developers crossing between the Java and TypeScript ecosystems.

---

## Schema Design Patterns

### Relations

Prisma models map directly to database tables. Relations are declared with `@relation` and always require a foreign key field on the owning side.

#### One-to-One

```prisma
model User {
  id      String  @id @default(uuid())
  profile Profile?
}

model Profile {
  id     String @id @default(uuid())
  bio    String
  user   User   @relation(fields: [userId], references: [id])
  userId String @unique // @unique enforces 1:1
}
```

The `@unique` on `userId` is what makes this one-to-one rather than one-to-many.

#### One-to-Many

```prisma
model Author {
  id    String @id @default(cuid())
  name  String
  posts Post[]
}

model Post {
  id       String @id @default(cuid())
  title    String
  author   Author @relation(fields: [authorId], references: [id])
  authorId String
}
```

#### Many-to-Many (Implicit)

```prisma
model Post {
  id         String     @id @default(cuid())
  title      String
  categories Category[]
}

model Category {
  id    String @id @default(cuid())
  name  String
  posts Post[]
}
```

Prisma creates a hidden join table `_CategoryToPost`. You cannot add columns to it.

#### Many-to-Many (Explicit Join Table)

Use this when the join table carries its own data:

```prisma
model Post {
  id   String       @id @default(cuid())
  tags PostTag[]
}

model Tag {
  id    String    @id @default(cuid())
  name  String
  posts PostTag[]
}

model PostTag {
  post      Post     @relation(fields: [postId], references: [id])
  postId    String
  tag       Tag      @relation(fields: [tagId], references: [id])
  tagId     String
  assignedAt DateTime @default(now())
  assignedBy String?

  @@id([postId, tagId])
}
```

#### Self-Relations

```prisma
model Employee {
  id        String     @id @default(uuid())
  name      String
  manager   Employee?  @relation("ManagerReports", fields: [managerId], references: [id])
  managerId String?
  reports   Employee[] @relation("ManagerReports")
}
```

### ID Strategies

| Strategy | Declaration | When to Use |
|----------|-------------|-------------|
| UUID v4 | `@default(uuid())` | Distributed systems, no ordering needed |
| CUID | `@default(cuid())` | URL-safe, sortable, good default choice |
| Auto-increment | `@default(autoincrement())` | Internal-only IDs, legacy compat |

CUIDs are collision-resistant and roughly sortable by creation time. UUIDs are the standard for cross-service identifiers. Auto-increment leaks ordering information and couples insert order to identity.

### Composite Unique Constraints and Indexes

```prisma
model Enrollment {
  id        String   @id @default(cuid())
  studentId String
  courseId   String
  semester  String
  grade     Float?

  student Student @relation(fields: [studentId], references: [id])
  course  Course  @relation(fields: [courseId], references: [id])

  @@unique([studentId, courseId, semester])
  @@index([courseId, semester])
}
```

`@@unique` creates a unique constraint and an index. `@@index` creates an index without a uniqueness constraint. Add `@@index` on columns you filter or sort by frequently.

### Enums

```prisma
enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

model Order {
  id     String      @id @default(cuid())
  status OrderStatus @default(PENDING)
}
```

Enums map to native database enums on PostgreSQL and to strings on MySQL/SQLite.

### Database-Level Naming with `@map` / `@@map`

Prisma conventions use PascalCase models and camelCase fields. The database might use snake_case. Map between them without changing your TypeScript API:

```prisma
model UserAccount {
  id        String   @id @default(uuid()) @map("user_account_id")
  firstName String   @map("first_name")
  lastName  String   @map("last_name")
  createdAt DateTime @default(now()) @map("created_at")

  @@map("user_accounts")
}
```

The generated client still uses `prisma.userAccount.findMany()` with `firstName`, but the underlying SQL targets `user_accounts.first_name`.

---

## Relation Strategies

### Implicit vs Explicit Many-to-Many

Use **implicit** when the relationship is a pure association with no metadata. Use **explicit** when you need extra fields on the join (timestamps, ordering, roles, permissions).

### Cascade Deletes

```prisma
model User {
  id    String @id @default(cuid())
  posts Post[]
}

model Post {
  id       String  @id @default(cuid())
  author   User    @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId String
  comments Comment[]
}

model Comment {
  id     String @id @default(cuid())
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String
}
```

Deleting a `User` cascades to their `Post`s, which cascades to `Comment`s. Other `onDelete` options: `SetNull`, `Restrict`, `NoAction`, `SetDefault`.

### Relation Loading: include vs select

```typescript
// include: fetch the relation alongside the parent
const user = await prisma.user.findUnique({
  where: { id: userId },
  include: {
    posts: true,        // all post fields
    profile: true,
  },
});
// user.posts is Post[]

// select: pick specific fields, including from relations
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: {
    name: true,
    posts: {
      select: { id: true, title: true },
      orderBy: { createdAt: "desc" },
      take: 10,
    },
  },
});
// user.posts is { id: string; title: string }[]
```

**Key rule**: `include` and `select` cannot be combined at the same level. Use `select` when you want to restrict the parent fields returned.

### The N+1 Problem

Prisma does NOT lazy-load relations. By default, relations are simply absent from the result. This means you cannot accidentally trigger N+1 queries the way JPA can with lazy loading. However, you can create an N+1 yourself:

```typescript
// BAD: N+1 — one query per user
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id },
  });
}

// GOOD: single query with include
const users = await prisma.user.findMany({
  include: { posts: true },
});
```

### Relation Load Strategy (Prisma 5.x)

By default, Prisma uses separate queries for each `include` level (similar to JPA's `FetchType.LAZY` with manual fetch). The `relationLoadStrategy: "join"` preview feature collapses nested includes into a single SQL JOIN:

```typescript
const users = await prisma.user.findMany({
  relationLoadStrategy: "join", // preview feature
  include: {
    posts: {
      include: { comments: true },
    },
  },
});
```

Enable in schema:

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["relationJoins"]
}
```

This can reduce round-trips but may return more data. Profile both strategies for your workload.

---

## Transactions

### Sequential (Batch) Transactions

Pass an array of operations. They execute sequentially in a single database transaction:

```typescript
const [order, payment, inventory] = await prisma.$transaction([
  prisma.order.create({
    data: { userId, total: 99.99, status: "CONFIRMED" },
  }),
  prisma.payment.create({
    data: { userId, amount: 99.99, method: "CARD" },
  }),
  prisma.inventoryItem.update({
    where: { sku: "WIDGET-42" },
    data: { quantity: { decrement: 1 } },
  }),
]);
```

All succeed or all roll back. You cannot use the result of one operation as input to the next — use interactive transactions for that.

### Interactive Transactions

```typescript
const result = await prisma.$transaction(async (tx) => {
  const account = await tx.account.findUniqueOrThrow({
    where: { id: fromAccountId },
  });

  if (account.balance < amount) {
    throw new Error("Insufficient funds");
  }

  const debit = await tx.account.update({
    where: { id: fromAccountId },
    data: { balance: { decrement: amount } },
  });

  const credit = await tx.account.update({
    where: { id: toAccountId },
    data: { balance: { increment: amount } },
  });

  return { debit, credit };
});
```

The `tx` parameter is a transactional PrismaClient. Any thrown error triggers a rollback.

### Timeout and Max Wait

```typescript
await prisma.$transaction(
  async (tx) => {
    // long-running work
  },
  {
    maxWait: 5000,    // ms to wait for a connection from the pool
    timeout: 10000,   // ms before the transaction is rolled back
  }
);
```

Default `maxWait` is 2000ms, default `timeout` is 5000ms.

### Isolation Levels

```typescript
await prisma.$transaction(
  async (tx) => {
    // reads see a consistent snapshot
  },
  {
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  }
);
```

Available levels: `ReadUncommitted`, `ReadCommitted`, `RepeatableRead`, `Serializable`. PostgreSQL default is `ReadCommitted`.

### Optimistic Locking Pattern

Prisma does not have built-in `@Version` like JPA. Implement it with a version field and conditional update:

```prisma
model Product {
  id      String @id @default(cuid())
  name    String
  price   Float
  version Int    @default(0)
}
```

```typescript
async function updateProductPrice(
  productId: string,
  newPrice: number,
  expectedVersion: number
): Promise<Product> {
  const updated = await prisma.product.updateMany({
    where: {
      id: productId,
      version: expectedVersion, // optimistic lock check
    },
    data: {
      price: newPrice,
      version: { increment: 1 },
    },
  });

  if (updated.count === 0) {
    throw new Error(
      `Optimistic lock failure: Product ${productId} was modified by another transaction`
    );
  }

  return prisma.product.findUniqueOrThrow({ where: { id: productId } });
}
```

We use `updateMany` because `update` throws if the `where` matches zero rows (it only accepts unique fields). `updateMany` with a compound `where` returns `{ count: 0 }` instead, which lets us detect the version mismatch.

### Retry Logic for Deadlocks

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const isPrismaDeadlock =
        error instanceof Prisma.PrismaClientKnownRequestError &&
        error.code === "P2034"; // transaction conflict / deadlock

      if (!isPrismaDeadlock || attempt === maxRetries) {
        throw error;
      }

      const backoff = Math.min(100 * Math.pow(2, attempt), 2000);
      await new Promise((resolve) => setTimeout(resolve, backoff));
    }
  }
  throw new Error("Unreachable");
}

// Usage
const result = await withRetry(() =>
  prisma.$transaction(async (tx) => {
    // transaction body
  })
);
```

---

## Raw Queries

### Tagged Template Literals (Safe)

```typescript
const email = "user@example.com";

// SELECT — returns typed results
const users = await prisma.$queryRaw<
  Array<{ id: string; name: string }>
>`SELECT id, name FROM "User" WHERE email = ${email}`;

// INSERT/UPDATE/DELETE — returns affected row count
const count = await prisma.$executeRaw`
  UPDATE "User" SET "lastLoginAt" = NOW() WHERE email = ${email}
`;
```

The tagged template literal parameterizes values automatically — `${email}` becomes a prepared statement parameter, not string interpolation.

### Dynamic SQL (Unsafe)

When you need to build column names or table names dynamically:

```typescript
import { Prisma } from "@prisma/client";

function buildDynamicQuery(
  tableName: string,
  sortColumn: string,
  limit: number
) {
  // Validate against allowlist to prevent SQL injection
  const allowedTables = ["User", "Post", "Comment"] as const;
  const allowedColumns = ["createdAt", "updatedAt", "name"] as const;

  if (!allowedTables.includes(tableName as any)) {
    throw new Error(`Invalid table: ${tableName}`);
  }
  if (!allowedColumns.includes(sortColumn as any)) {
    throw new Error(`Invalid sort column: ${sortColumn}`);
  }

  return prisma.$queryRawUnsafe(
    `SELECT * FROM "${tableName}" ORDER BY "${sortColumn}" DESC LIMIT $1`,
    limit
  );
}
```

Always validate dynamic identifiers against an allowlist. `$queryRawUnsafe` does NOT auto-parameterize — use `$1`, `$2` placeholders and pass values as additional arguments.

### When Raw Queries Are Necessary

- Window functions (`ROW_NUMBER`, `RANK`, `LAG`/`LEAD`)
- Common Table Expressions (recursive CTEs for tree traversal)
- Complex aggregations that exceed `groupBy` capabilities
- Database-specific features (PostgreSQL `LATERAL JOIN`, `JSONB` operators)
- `EXPLAIN ANALYZE` for query plan inspection
- Full-text search with `ts_vector` / `ts_query`

---

## Connection Pooling

### Built-in Pool

Prisma Client maintains its own connection pool. Configure via the connection string:

```
DATABASE_URL="postgresql://user:pass@localhost:5432/mydb?connection_limit=10&pool_timeout=10"
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `connection_limit` | `num_cpus * 2 + 1` | Max connections in the pool |
| `pool_timeout` | 10 (seconds) | Time to wait for a free connection |

For serverless environments (AWS Lambda, Vercel Functions), keep `connection_limit` low (1-3) to avoid exhausting database connections across cold starts.

### PgBouncer Integration

When using PgBouncer in transaction mode, Prisma needs specific settings:

```
DATABASE_URL="postgresql://user:pass@pgbouncer-host:6432/mydb?pgbouncer=true&connection_limit=10"
```

The `pgbouncer=true` flag tells Prisma to avoid prepared statements (PgBouncer in transaction mode does not support them).

### Prisma Accelerate (Serverless/Edge)

Prisma Accelerate is a managed connection pooler and global cache:

```typescript
import { PrismaClient } from "@prisma/client/edge";
import { withAccelerate } from "@prisma/extension-accelerate";

const prisma = new PrismaClient().$extends(withAccelerate());

const users = await prisma.user.findMany({
  cacheStrategy: {
    ttl: 60,   // cache for 60 seconds
    swr: 120,  // serve stale for 120 seconds while revalidating
  },
});
```

### Multi-Tenant Connection Switching

Override the datasource URL at runtime for per-tenant databases:

```typescript
function getTenantClient(tenantDatabaseUrl: string): PrismaClient {
  return new PrismaClient({
    datasourceUrl: tenantDatabaseUrl,
  });
}

// Usage
const tenantDb = getTenantClient(tenant.databaseUrl);
const users = await tenantDb.user.findMany();
```

Pool client instances per tenant rather than creating a new one per request. A `Map<string, PrismaClient>` with cleanup logic works well.

---

## Query Optimization

### EXPLAIN ANALYZE

```typescript
const plan = await prisma.$queryRaw<Array<{ "QUERY PLAN": string }>>`
  EXPLAIN ANALYZE
  SELECT p.* FROM "Post" p
  JOIN "User" u ON p."authorId" = u.id
  WHERE u.email = ${email}
  ORDER BY p."createdAt" DESC
  LIMIT 20
`;

for (const row of plan) {
  console.log(row["QUERY PLAN"]);
}
```

### Select Only What You Need

```typescript
// BAD: fetches all 30 columns
const users = await prisma.user.findMany();

// GOOD: fetches only 3 columns
const users = await prisma.user.findMany({
  select: { id: true, name: true, email: true },
});
```

This matters most when tables have large text/JSON columns.

### Batch Operations

```typescript
// createMany: single INSERT with multiple rows
await prisma.user.createMany({
  data: [
    { name: "Alice", email: "alice@example.com" },
    { name: "Bob", email: "bob@example.com" },
  ],
  skipDuplicates: true,
});

// updateMany: single UPDATE with WHERE clause
await prisma.post.updateMany({
  where: { authorId: userId, status: "DRAFT" },
  data: { status: "ARCHIVED" },
});

// deleteMany: single DELETE with WHERE clause
await prisma.session.deleteMany({
  where: { expiresAt: { lt: new Date() } },
});
```

`createMany` does not return the created records (PostgreSQL limitation on multi-row INSERT ... RETURNING). Use `createManyAndReturn` (Prisma 5.14+) if you need them.

### findUnique vs findFirst

| Method | Constraint | Performance |
|--------|-----------|-------------|
| `findUnique` | Requires `@id` or `@unique` field | Uses index seek, returns at most one row |
| `findFirst` | Any `where` clause | May scan, respects `orderBy`, returns first match |

Always prefer `findUnique` when querying by a unique field — the database can short-circuit.

### Cursor-Based vs Offset Pagination

```typescript
// Offset: simple but degrades on deep pages (OFFSET 10000)
const page = await prisma.post.findMany({
  skip: (pageNumber - 1) * pageSize,
  take: pageSize,
  orderBy: { createdAt: "desc" },
});

// Cursor: stable performance regardless of depth
const page = await prisma.post.findMany({
  take: pageSize,
  skip: 1,           // skip the cursor row itself
  cursor: { id: lastSeenId },
  orderBy: { createdAt: "desc" },
});
```

Cursor-based pagination uses a `WHERE id > ?` under the hood, which is an index seek. Offset pagination uses `OFFSET N`, which still scans and discards N rows.

---

## PrismaClient Extension API

### Overview

`$extends` lets you add custom methods, computed fields, query middleware, and result transformations without modifying the generated client.

### Soft Delete Extension

```typescript
const prisma = new PrismaClient().$extends({
  name: "softDelete",
  query: {
    $allModels: {
      async findMany({ model, operation, args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
      async findFirst({ model, operation, args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
      async delete({ model, operation, args, query }) {
        // Convert delete to soft-delete
        return (prisma as any)[model].update({
          ...args,
          data: { deletedAt: new Date() },
        });
      },
      async deleteMany({ model, operation, args, query }) {
        return (prisma as any)[model].updateMany({
          ...args,
          data: { deletedAt: new Date() },
        });
      },
    },
  },
});
```

Every `findMany` and `findFirst` now auto-filters out soft-deleted rows. `delete` becomes an `update` that sets `deletedAt`.

### Computed Fields

```typescript
const prisma = new PrismaClient().$extends({
  name: "computedFields",
  result: {
    user: {
      fullName: {
        needs: { firstName: true, lastName: true },
        compute(user) {
          return `${user.firstName} ${user.lastName}`;
        },
      },
    },
  },
});

const user = await prisma.user.findUnique({ where: { id: "..." } });
console.log(user.fullName); // "John Doe" — computed at read time
```

### Audit Trail Extension

```typescript
const prisma = new PrismaClient().$extends({
  name: "auditTrail",
  query: {
    $allModels: {
      async create({ model, operation, args, query }) {
        const result = await query(args);
        await logAuditEvent(model, "CREATE", null, result);
        return result;
      },
      async update({ model, operation, args, query }) {
        const before = await (prisma as any)[model].findUnique({
          where: args.where,
        });
        const result = await query(args);
        await logAuditEvent(model, "UPDATE", before, result);
        return result;
      },
      async delete({ model, operation, args, query }) {
        const before = await (prisma as any)[model].findUnique({
          where: args.where,
        });
        const result = await query(args);
        await logAuditEvent(model, "DELETE", before, null);
        return result;
      },
    },
  },
});

async function logAuditEvent(
  model: string,
  action: string,
  before: unknown,
  after: unknown
) {
  await prisma.auditLog.create({
    data: {
      model,
      action,
      before: before ? JSON.stringify(before) : null,
      after: after ? JSON.stringify(after) : null,
      timestamp: new Date(),
    },
  });
}
```

---

## Migrations

### Core Commands

| Command | Purpose | Environment |
|---------|---------|-------------|
| `prisma migrate dev` | Generate + apply migration, sync client | Development |
| `prisma migrate deploy` | Apply pending migrations | Production / CI |
| `prisma db push` | Push schema directly (no migration file) | Prototyping |
| `prisma migrate reset` | Drop DB + re-apply all migrations | Development only |

### Migration Workflow

```bash
# 1. Edit schema.prisma
# 2. Generate migration SQL
npx prisma migrate dev --name add_order_status_enum

# 3. Review the generated SQL in prisma/migrations/<timestamp>_add_order_status_enum/migration.sql
# 4. Edit if needed (data migrations, backfills)
# 5. Commit the migration file to version control
```

### Data Migrations

Prisma generates DDL only. For data transformations, edit the generated SQL:

```sql
-- prisma/migrations/20250301120000_add_full_name/migration.sql

-- 1. Add column (Prisma generated this)
ALTER TABLE "User" ADD COLUMN "fullName" TEXT;

-- 2. Backfill data (you add this manually)
UPDATE "User" SET "fullName" = "firstName" || ' ' || "lastName";

-- 3. Make non-nullable after backfill (you add this manually)
ALTER TABLE "User" ALTER COLUMN "fullName" SET NOT NULL;
```

### Shadow Database

`prisma migrate dev` uses a temporary "shadow database" to detect schema drift:

1. Creates a shadow database
2. Applies all existing migrations to it
3. Diffs the shadow against your schema.prisma
4. Generates a new migration from the diff
5. Drops the shadow database

This requires `CREATE DATABASE` permissions. In restricted environments, configure a separate shadow database URL:

```prisma
datasource db {
  provider          = "postgresql"
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

### `db push` for Prototyping

`prisma db push` applies schema changes directly without creating migration files. Use it during early development when the schema is unstable. Switch to `migrate dev` once the schema stabilizes and you need reproducible migrations.

---

## Java JPA Parallel

| Prisma | JPA / Spring Data |
|--------|-------------------|
| `schema.prisma` model | `@Entity` class with `@Table`, `@Column` |
| `@id @default(cuid())` | `@Id @GeneratedValue(strategy = UUID)` |
| `@relation(fields: [...], references: [...])` | `@ManyToOne`, `@OneToMany(mappedBy = "...")` |
| `@@unique([a, b])` | `@Table(uniqueConstraints = @UniqueConstraint(...))` |
| `@@index([col])` | `@Table(indexes = @Index(...))` or Flyway migration |
| `@map("col_name")` / `@@map("table_name")` | `@Column(name = "col_name")` / `@Table(name = "table_name")` |
| `include: { posts: true }` | `FetchType.EAGER` or `JOIN FETCH` in JPQL |
| No include (default) | `FetchType.LAZY` (loaded on access, risks N+1) |
| `select: { id: true, name: true }` | DTO projection with `SELECT new Dto(...)` or interface projection |
| `prisma.$transaction(async (tx) => {...})` | `@Transactional` on service method |
| `prisma.$transaction([...])` (batch) | Multiple repository calls inside `@Transactional` |
| Isolation level in `$transaction` options | `@Transactional(isolation = Isolation.SERIALIZABLE)` |
| Version field + `updateMany` WHERE check | `@Version` field (auto-managed by JPA) |
| `prisma migrate dev` / `prisma migrate deploy` | Flyway `migrate` / Liquibase `update` |
| `prisma db push` | Hibernate `ddl-auto=update` (prototyping) |
| `$extends` (soft delete, audit) | `@SQLDelete`, `@Where`, `@EntityListeners` / Envers |
| Prisma Client (generated) | Spring Data Repository (interface proxy) |
| `prisma.$queryRaw` | `@Query(nativeQuery = true)` / `JdbcTemplate` |

### Key Differences

**Lazy loading**: JPA entities lazy-load relations by default, which can trigger N+1 queries silently when accessing a collection property. Prisma never loads relations unless you explicitly `include` or `select` them. This is safer but requires you to think about data needs upfront.

**Unit of work**: JPA's `EntityManager` tracks dirty entities and flushes changes at transaction commit (automatic dirty checking). Prisma has no managed entity state — every write is an explicit `create`, `update`, or `delete` call. There is no implicit persistence.

**Optimistic locking**: JPA handles this transparently with `@Version` — the framework checks and increments the version on every `merge`/`save`. Prisma requires you to implement the pattern manually with `updateMany` + version field check.

**Schema management**: Flyway/Liquibase give you full control over SQL migration files, changelogs, and rollback scripts. Prisma generates migration SQL from schema diffs, which is more automated but less flexible for complex data migrations.

---

## References

- [Prisma Documentation](https://www.prisma.io/docs) -- official reference for schema, client API, and migrations
- [Prisma Client API Reference](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference) -- exhaustive method signatures and options
- [Prisma Schema Reference](https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference) -- all schema attributes and functions
- [Prisma Migrate](https://www.prisma.io/docs/orm/prisma-migrate) -- migration workflow and concepts
- [Prisma Client Extensions](https://www.prisma.io/docs/orm/prisma-client/client-extensions) -- `$extends` API documentation
- [Prisma Error Codes](https://www.prisma.io/docs/reference/api-reference/error-reference) -- P2034 (deadlock), P2025 (not found), etc.
- [Connection Management](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-management) -- pool sizing and serverless guidance
- [Prisma Accelerate](https://www.prisma.io/docs/accelerate) -- managed connection pool and edge caching
