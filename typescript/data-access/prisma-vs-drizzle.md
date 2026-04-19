# Prisma vs Drizzle

> Problem frame: When Prisma's abstraction hurts and Drizzle's SQL-closeness helps.

Both Prisma and Drizzle are type-safe data access layers for TypeScript, but they occupy fundamentally different positions on the abstraction spectrum. This guide walks through every dimension that matters when choosing between them or migrating from one to the other.

---

## Philosophy

### Prisma: Schema-First, Abstraction-Forward

Prisma treats the database as an implementation detail. You define models in a `.prisma` DSL, run `prisma generate`, and get a fully typed client. SQL is hidden behind method chains. The target audience is product developers who want to ship fast without thinking in SQL.

**JPA parallel:** Prisma's philosophy mirrors JPA/Hibernate. You define entities (schema), the framework generates queries, and you escape to native queries when needed.

### Drizzle: Code-First, SQL-in-TypeScript

Drizzle's pitch is literal: "If you know SQL, you know Drizzle." Schemas are TypeScript objects. Queries read like SQL translated into method chains. There is no code generation step, no binary dependency, and no custom DSL. The target audience is developers who think in SQL and want TypeScript to verify their queries at compile time.

**JPA parallel:** Drizzle is closer to jOOQ or MyBatis. You write the query; the library types it.

| Dimension | Prisma | Drizzle |
|-----------|--------|---------|
| Schema language | `.prisma` DSL | TypeScript (`pgTable`, `mysqlTable`) |
| Code generation | Required (`prisma generate`) | None (TS inference) |
| Binary dependency | Rust query engine | None |
| SQL proximity | Low (abstracted) | High (SQL-shaped API) |
| Learning curve for SQL users | Medium | Low |
| Learning curve for non-SQL users | Low | Medium |

---

## Schema Definition

The same domain model expressed in both systems: a `User` who has many `Post` entries.

### Prisma Schema

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String   @db.VarChar(255)
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
  createdAt DateTime @default(now())
}
```

After editing the schema, run `npx prisma generate` to produce the typed client.

### Drizzle Schema

```typescript
// src/db/schema.ts
import { pgTable, serial, varchar, text, boolean, integer, timestamp } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  name: varchar('name', { length: 255 }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 255 }).notNull(),
  content: text('content'),
  published: boolean('published').default(false).notNull(),
  authorId: integer('author_id').references(() => users.id).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Relations are declared separately (used by the relational query API)
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}));
```

No generation step. The schema is just TypeScript. Refactoring tools (rename symbol, find references) work out of the box.

---

## Query Builder

### Basic: findMany with where, orderBy, include

**Prisma:**

```typescript
const recentPosts = await prisma.post.findMany({
  where: { published: true },
  orderBy: { createdAt: 'desc' },
  include: { author: true },
  take: 10,
});
// Type: (Post & { author: User })[]
```

**Drizzle (relational query API):**

```typescript
const recentPosts = await db.query.posts.findMany({
  where: eq(posts.published, true),
  orderBy: desc(posts.createdAt),
  with: { author: true },
  limit: 10,
});
// Type inferred from schema + with clause
```

**Drizzle (SQL-like API):**

```typescript
const recentPosts = await db
  .select()
  .from(posts)
  .where(eq(posts.published, true))
  .orderBy(desc(posts.createdAt))
  .innerJoin(users, eq(posts.authorId, users.id))
  .limit(10);
// Type: { posts: Post; users: User }[]
```

At this level, both are comparable. The differences emerge with complexity.

### Nested Includes

**Prisma:**

```typescript
const userWithPostsAndComments = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      include: { comments: { include: { author: true } } },
      where: { published: true },
    },
  },
});
```

**Drizzle:**

```typescript
const userWithPostsAndComments = await db.query.users.findFirst({
  where: eq(users.id, 1),
  with: {
    posts: {
      where: eq(posts.published, true),
      with: {
        comments: { with: { author: true } },
      },
    },
  },
});
```

Both handle nested includes well. Prisma's `include` and Drizzle's `with` are structurally similar.

### Raw Joins with Complex Conditions

This is where the gap opens. A query joining posts with a lateral subquery for the latest comment:

**Prisma:** Not expressible with the client API. You need `$queryRaw`:

```typescript
const results = await prisma.$queryRaw`
  SELECT p.*, lc.latest_comment
  FROM "Post" p
  LEFT JOIN LATERAL (
    SELECT c.content AS latest_comment
    FROM "Comment" c
    WHERE c."postId" = p.id
    ORDER BY c."createdAt" DESC
    LIMIT 1
  ) lc ON true
  WHERE p.published = true
`;
// Type: unknown — you must cast manually
```

**Drizzle:** Expressible inline with full type safety:

```typescript
const latestComment = db
  .select({ content: comments.content })
  .from(comments)
  .where(eq(comments.postId, posts.id))
  .orderBy(desc(comments.createdAt))
  .limit(1)
  .as('latest_comment');

const results = await db
  .select({
    post: posts,
    latestComment: latestComment.content,
  })
  .from(posts)
  .leftJoin(latestComment, sql`true`)
  .where(eq(posts.published, true));
// Fully typed
```

### Aggregations: GROUP BY, HAVING

**Prisma:**

```typescript
const authorStats = await prisma.post.groupBy({
  by: ['authorId'],
  _count: { id: true },
  _avg: { viewCount: true },
  having: {
    id: { _count: { gt: 5 } },
  },
});
```

Prisma's `groupBy` works for simple cases but becomes awkward with multiple aggregations or computed columns.

**Drizzle:**

```typescript
const authorStats = await db
  .select({
    authorId: posts.authorId,
    postCount: count(posts.id),
    avgViews: avg(posts.viewCount),
  })
  .from(posts)
  .groupBy(posts.authorId)
  .having(gt(count(posts.id), 5));
```

Reads like SQL. Adding more aggregate functions is straightforward.

### CTEs (Common Table Expressions)

**Prisma:** No CTE support. Use `$queryRaw`.

**Drizzle:**

```typescript
const activeAuthors = db.$with('active_authors').as(
  db.select({ id: users.id, name: users.name })
    .from(users)
    .innerJoin(posts, eq(users.id, posts.authorId))
    .where(gte(posts.createdAt, thirtyDaysAgo))
    .groupBy(users.id, users.name)
);

const result = await db
  .with(activeAuthors)
  .select()
  .from(activeAuthors);
```

### Window Functions

**Prisma:** No support. Raw query only.

**Drizzle:**

```typescript
const rankedPosts = await db
  .select({
    id: posts.id,
    title: posts.title,
    authorId: posts.authorId,
    rank: sql<number>`ROW_NUMBER() OVER (PARTITION BY ${posts.authorId} ORDER BY ${posts.createdAt} DESC)`,
  })
  .from(posts);
```

The `sql` template literal lets you drop into raw SQL for any expression while keeping the rest of the query typed.

### Summary: Query Expressiveness

| Feature | Prisma | Drizzle |
|---------|--------|---------|
| Basic CRUD | Native | Native |
| Nested relations | `include` | `with` |
| Complex JOINs | `$queryRaw` (untyped) | Native (typed) |
| GROUP BY / HAVING | `groupBy` (limited) | Native (typed) |
| CTEs | `$queryRaw` only | `$with` (typed) |
| Window functions | `$queryRaw` only | `sql` template (typed) |
| Subqueries | `$queryRaw` only | Native (typed) |

---

## Type Safety

Both libraries are excellent at type safety, but the mechanisms differ.

### Prisma: Generated Types

Prisma generates types from the `.prisma` schema into `node_modules/.prisma/client`:

```typescript
import { Prisma } from '@prisma/client';

// The full model type
type User = Prisma.UserGetPayload<{}>;

// A type with specific includes
type UserWithPosts = Prisma.UserGetPayload<{
  include: { posts: true };
}>;

// Input types for create/update
type CreateUserInput = Prisma.UserCreateInput;
```

The generated types are precise. The `include`/`select` payload types are computed at the type level, so the return type changes based on what you include.

### Drizzle: Inferred Types

Drizzle infers types from the TypeScript schema definition:

```typescript
import { InferSelectModel, InferInsertModel } from 'drizzle-orm';
// or use the table-level shortcuts:
type User = typeof users.$inferSelect;        // SELECT result shape
type NewUser = typeof users.$inferInsert;      // INSERT input shape

// Partial select
const result = await db
  .select({ id: users.id, email: users.email })
  .from(users);
// Type: { id: number; email: string }[]  — automatically narrowed
```

### Where They Differ

**Prisma's relational queries** produce deeply nested types that exactly match the `include` structure. This is powerful but requires understanding Prisma's `Payload` utility types for extraction.

**Drizzle's relational queries** (the `with` API) similarly produce nested types. However, the SQL-like API returns flat join results by default:

```typescript
// Drizzle join returns a flat structure
const result = await db
  .select()
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id));
// Type: { posts: Post; users: User }[]  — namespaced by table
```

Both approaches are fully type-safe. Prisma requires a generation step after every schema change. Drizzle reflects changes instantly since the schema is TypeScript.

---

## Migrations

### Prisma Migration Workflow

```bash
# 1. Edit schema.prisma

# 2. Generate migration SQL from the schema diff
npx prisma migrate dev --name add_view_count
# Creates prisma/migrations/<timestamp>_add_view_count/migration.sql
# Applies the migration to the dev database
# Runs prisma generate automatically

# 3. In production
npx prisma migrate deploy
# Applies pending migrations in order
```

Key characteristics:
- **Shadow database**: Prisma creates a temporary database to diff your schema against. Requires `CREATE DATABASE` privileges in development.
- **Migration files**: Auto-generated SQL. You can edit them before applying.
- **Drift detection**: Warns if the database schema has drifted from the migration history.
- **Baselining**: `prisma migrate resolve` for existing databases.

### Drizzle Migration Workflow

```bash
# 1. Edit your TypeScript schema files

# 2. Generate migration SQL
npx drizzle-kit generate
# Creates drizzle/<timestamp>_migration.sql

# 3. Apply migrations
npx drizzle-kit migrate
# Runs pending migrations against the database

# For prototyping (no migration files, direct push):
npx drizzle-kit push
# Syncs schema directly to DB — like prisma db push
```

Key characteristics:
- **No shadow database**: Drizzle compares the TS schema against the current database state directly using introspection.
- **Migration files**: Auto-generated SQL. Editable before applying.
- **`push` for prototyping**: Fast iteration without migration files, similar to `prisma db push`.
- **Simpler setup**: No extra database privileges needed beyond the target database.

### Side-by-Side Comparison

| Aspect | Prisma | Drizzle |
|--------|--------|---------|
| Schema source | `.prisma` file | `.ts` files |
| Diff mechanism | Shadow database | Database introspection |
| Generate SQL | `prisma migrate dev` | `drizzle-kit generate` |
| Apply migrations | `prisma migrate deploy` | `drizzle-kit migrate` |
| Quick prototyping | `prisma db push` | `drizzle-kit push` |
| Extra DB privileges | Yes (shadow DB) | No |
| Migration directory | `prisma/migrations/` | `drizzle/` (configurable) |

---

## Performance

### Prisma's Architecture

Prisma runs a Rust-based **query engine** as a sidecar binary. Your TypeScript code sends queries to this engine via an internal protocol, and the engine translates them to SQL, manages the connection pool, and returns results.

```
Your TS code → Prisma Client (JS) → Query Engine (Rust binary) → PostgreSQL
```

Implications:
- **Connection pooling**: Built into the engine. Configurable via `connection_limit` in the URL.
- **Query batching**: Prisma automatically batches `findUnique` calls within the same tick via its DataLoader-like mechanism.
- **Cold start cost**: The binary must be loaded. Adds ~100-300ms to the first invocation in serverless.
- **Memory overhead**: The Rust binary consumes its own memory separate from the Node process.

### Drizzle's Architecture

Drizzle is a thin TypeScript layer that generates SQL strings and passes them directly to the underlying driver (`pg`, `postgres.js`, `better-sqlite3`, etc.).

```
Your TS code → Drizzle (TS query builder) → Driver (pg / postgres.js) → PostgreSQL
```

Implications:
- **No binary**: Zero cold start overhead beyond normal module loading.
- **Connection pooling**: Handled by the driver or an external pooler (PgBouncer, Neon pooler).
- **Performance**: Essentially raw driver performance with the overhead of building SQL strings (negligible).
- **Memory**: Only the Node.js process.

### When Does It Matter?

For most traditional server applications (Express/Fastify on a VM or container), Prisma's overhead is negligible. The query engine is loaded once and stays resident.

It matters in:
- **Serverless functions**: Cold starts are frequent. The binary adds latency and bundle size.
- **Edge runtimes**: Many edge environments cannot run native binaries at all.
- **High-throughput services**: At thousands of queries/second, the IPC overhead between Node and the Rust engine adds up.

---

## Edge and Serverless

### Prisma's Serverless Story

Prisma cannot run the Rust query engine on edge runtimes (Cloudflare Workers, Vercel Edge). Their solutions:

- **Prisma Accelerate**: A managed proxy service. Your edge function sends queries to Prisma's infrastructure, which runs the query engine and forwards to your database. Adds a network hop.
- **Prisma Data Proxy**: Self-hosted alternative for the same pattern.
- **Driver Adapters (Preview)**: Allows Prisma to use JS-based drivers (like `@neondatabase/serverless`) instead of the Rust engine. Reduces the gap but still carries the Prisma client weight.

```typescript
// Prisma on edge with Accelerate
import { PrismaClient } from '@prisma/client/edge';
import { withAccelerate } from '@prisma/extension-accelerate';

const prisma = new PrismaClient().$extends(withAccelerate());
```

### Drizzle's Serverless Story

Drizzle works natively with any JavaScript driver. No proxy needed.

```typescript
// Drizzle on Cloudflare Workers with Neon
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

export default {
  async fetch(request: Request, env: Env) {
    const sql = neon(env.DATABASE_URL);
    const db = drizzle(sql, { schema });

    const users = await db.query.users.findMany();
    return Response.json(users);
  },
};
```

```typescript
// Drizzle on Vercel Edge with PlanetScale
import { drizzle } from 'drizzle-orm/planetscale-serverless';
import { Client } from '@planetscale/database';

const client = new Client({ url: process.env.DATABASE_URL });
const db = drizzle(client);
```

**Supported drivers out of the box**: `pg`, `postgres.js`, `@neondatabase/serverless`, `@planetscale/database`, `better-sqlite3`, `@libsql/client` (Turso), `mysql2`, `bun:sqlite`, and more.

---

## Developer Experience

### Prisma

- **Prisma Studio**: Built-in GUI for browsing and editing data (`npx prisma studio`). Runs locally on port 5555.
- **IntelliSense**: Exceptional. The generated client provides autocomplete for every field, relation, filter operator, and nested operation.
- **Schema as documentation**: The `.prisma` file is a readable, centralized view of your entire data model. Non-developers can read it.
- **Error messages**: Clear and specific. Validation errors point to the exact schema line.
- **Ecosystem**: Prisma Pulse (real-time), Prisma Optimize (query analysis), large community.

### Drizzle

- **Drizzle Studio**: Web-based data browser (`npx drizzle-kit studio`). Newer than Prisma Studio but actively developed.
- **IntelliSense**: Excellent. Since the schema is TypeScript, standard TS tooling (rename, go-to-definition, find references) works without any special extension.
- **Refactoring**: Renaming a column in the schema is a TypeScript rename. Every query referencing that column gets updated. Prisma requires regenerating the client after schema changes.
- **Bundle size**: Drizzle core is ~50KB. Prisma's client + engine is significantly larger.
- **SQL debugging**: Drizzle can log the exact SQL it generates. You always know what query hits the database because the API mirrors SQL.

### IDE Experience Comparison

| Feature | Prisma | Drizzle |
|---------|--------|---------|
| Go to definition (schema → query) | Via generated types | Native TS |
| Rename refactoring | Requires regenerate | Instant |
| Schema file format | Custom `.prisma` | `.ts` |
| VS Code extension | Prisma extension (syntax, format) | None needed |
| Data browser | Prisma Studio (mature) | Drizzle Studio (newer) |
| Bundle size (approx) | ~15MB (with engine) | ~50KB |

---

## When to Choose Which

### Choose Prisma When

- **Rapid prototyping**: The schema DSL and auto-generated migrations get you from zero to CRUD faster than anything else.
- **Team with mixed SQL experience**: Prisma abstracts SQL. Junior developers can be productive without writing a single JOIN.
- **You need strong migration tooling**: Prisma's migration system with drift detection, baselining, and shadow database is more mature.
- **Your queries are mostly CRUD**: If 90% of your queries are `findMany`, `create`, `update`, `delete` with basic filters and includes, Prisma's abstraction costs you nothing.
- **You value the ecosystem**: Prisma Studio, Prisma Accelerate, Prisma Optimize, and the large community.

### Choose Drizzle When

- **Complex queries are the norm**: If your application needs CTEs, window functions, complex JOINs, or subqueries, Drizzle avoids the constant escape to `$queryRaw`.
- **Your team thinks in SQL**: Drizzle rewards SQL knowledge. The API is a 1:1 mapping. No new query language to learn.
- **Serverless or edge-first**: No binary dependency means zero cold start overhead and compatibility with every edge runtime.
- **You want no binary dependency**: Some teams have constraints around native binaries in their deployment pipeline.
- **Bundle size matters**: Drizzle's ~50KB footprint vs Prisma's ~15MB matters for serverless and edge.
- **You want standard TS tooling**: Schema-as-TypeScript means rename, find references, and go-to-definition work without plugins.

### Both Are Valid When

- Switching cost is low (early project, small team).
- Pick based on your team's SQL comfort level.
- Neither is wrong for a typical CRUD-heavy web application.

---

## Migration from Prisma to Drizzle

### Step 1: Introspect the Existing Database

Drizzle can generate a TypeScript schema from your current database, regardless of how it was created:

```bash
npx drizzle-kit introspect
```

This reads your database and produces `.ts` schema files. Review the output — it maps tables, columns, indexes, and foreign keys but may need manual adjustment for relation declarations.

### Step 2: Configure Drizzle

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

### Step 3: Gradual Migration

You do not need to switch everything at once. Both ORMs can coexist against the same database:

```typescript
// src/db/index.ts
import { PrismaClient } from '@prisma/client';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

// Keep Prisma for existing code
export const prisma = new PrismaClient();

// Add Drizzle for new code and complex queries
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
```

Migration strategy:
1. **New features**: Write with Drizzle.
2. **Complex queries**: Migrate `$queryRaw` calls to typed Drizzle queries first — these benefit most.
3. **Simple CRUD**: Migrate last (or not at all if Prisma is working fine).
4. **Remove Prisma**: Once all queries are migrated, remove `@prisma/client`, the `.prisma` schema, and the Rust engine.

### Step 4: Migration Ownership

Decide who owns migrations going forward. During the transition period, use Drizzle for new migrations:

```bash
# Stop using prisma migrate
# Start using drizzle-kit
npx drizzle-kit generate
npx drizzle-kit migrate
```

Keep the Prisma migration history in the repo for reference, but do not run new Prisma migrations.

---

## References

- [Prisma Documentation](https://www.prisma.io/docs)
- [Drizzle ORM Documentation](https://orm.drizzle.team/docs/overview)
- [Drizzle Kit CLI Reference](https://orm.drizzle.team/kit-docs/overview)
- [Prisma vs Drizzle — Drizzle Team Comparison](https://orm.drizzle.team/docs/prisma)
- [Prisma Client API Reference](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference)
- [Drizzle Relational Queries](https://orm.drizzle.team/docs/rqb)
- [Prisma Edge Functions](https://www.prisma.io/docs/guides/deployment/edge/overview)
- Related: [Prisma Deep Dive](prisma-deep-dive.md)
