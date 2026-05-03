---
title: "`database/sql` and the Go Data-Access Stack"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, data-access, database-sql, postgres, connection-pooling]
---

# `database/sql` and the Go Data-Access Stack

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `data-access` `database-sql` `postgres` `connection-pooling`

---

## Table of Contents

1. [The `database/sql` philosophy](#1-the-databasesql-philosophy)
2. [Drivers and the underscore import](#2-drivers-and-the-underscore-import)
3. [The connection pool](#3-the-connection-pool)
4. [Always use the `Context` variants](#4-always-use-the-context-variants)
5. [Scanning rows and `NULL` handling](#5-scanning-rows-and-null-handling)
6. [Prepared statements](#6-prepared-statements)
7. [Transactions and the deferred-rollback idiom](#7-transactions-and-the-deferred-rollback-idiom)
8. [`sql.ErrNoRows` is a sentinel](#8-sqlerrnorows-is-a-sentinel)
9. [`sqlx` — extension, not ORM](#9-sqlx--extension-not-orm)
10. [`pgx` — bypass `database/sql` for Postgres](#10-pgx--bypass-databasesql-for-postgres)
11. [Migrations: golang-migrate, goose, atlas](#11-migrations-golang-migrate-goose-atlas)

## Summary

Go's standard library ships a single, deliberately small data-access package: `database/sql`. It is **not an ORM**. It does not generate SQL, marshal structs, manage schema, or hide the connection pool. It is a thin abstraction over driver-implemented connections that gives you a pool, transaction management, prepared statements, and a `Rows`/`Row` cursor. Everything else — query building, struct scanning, migrations — lives in third-party libraries that compose on top.

Coming from Prisma (TypeScript) or JPA/Hibernate (Java), the model is jarringly explicit. There is no schema file that becomes Go types. There is no `EntityManager` tracking dirty state. Every query is a string, every scan is a manual destination list, every transaction is a `Begin/Commit/Rollback` you wrote by hand. The payoff is that nothing happens that you didn't write, the SQL you see is the SQL the database runs, and the cost model is obvious.

This doc covers the standard library core (sections 1–8), the most common helper layer `sqlx` (§9), the Postgres-specific driver `pgx` that many production services run instead of `database/sql` (§10), and the migration tooling that fills the schema-management gap (§11).

## 1. The `database/sql` philosophy

`database/sql` is an interface package. It defines `*sql.DB`, `*sql.Tx`, `*sql.Rows`, `*sql.Row`, `*sql.Stmt`, and a few error sentinels. The actual wire protocol — how to talk Postgres, MySQL, SQLite — lives in third-party drivers that satisfy the `database/sql/driver` interfaces.

A few deliberate non-features, all of which Prisma and JPA do for you:

- **No schema definition.** There is no Go-side declaration of tables. Your schema lives in the database; your migration tool keeps it in sync.
- **No struct mapping.** `Scan` takes pointers to destinations in column order. You write that list. (See §9 for `sqlx.StructScan`, which adds reflection-based mapping on top.)
- **No query builder.** Queries are strings with `?` or `$1` placeholders.
- **No relation traversal.** No `user.Posts` lazy loading; you write the JOIN or the second query.
- **No identity map / dirty tracking.** Every UPDATE is a SQL statement you constructed.

What it **does** give you:

- A connection pool with configurable limits and lifetimes.
- Context-aware methods (`QueryContext`, `ExecContext`) that propagate cancellation to the driver.
- Transaction management with isolation-level support.
- Prepared statement caching at the connection level.
- A driver registry so you can swap Postgres drivers without changing call sites.

The closest TypeScript analogue is the `pg` node-postgres driver — a low-level pool plus query interface, which Prisma and Drizzle wrap. The closest Java analogue is JDBC, which Hibernate sits on top of. Go just stops at the `database/sql` layer for many production services and bolts on `sqlc` or `sqlx` rather than reaching for a full ORM.

## 2. Drivers and the underscore import

A driver registers itself with `database/sql` in its `init()` function. To activate a driver in your binary, you import its package for the side effect:

```go
import (
    "database/sql"

    _ "github.com/jackc/pgx/v5/stdlib"  // registers "pgx"
    // or:
    // _ "github.com/lib/pq"            // registers "postgres"
    // _ "github.com/mattn/go-sqlite3"  // registers "sqlite3"
)

func main() {
    db, err := sql.Open("pgx", "postgres://user:pass@localhost:5432/mydb")
    // ...
}
```

The leading `_` is the **blank identifier import**. It tells Go: "I'm not naming any symbol from this package, but run its `init()` function." The `init()` calls `sql.Register("pgx", &Driver{})` so that `sql.Open("pgx", ...)` knows what to do.

`sql.Open` does **not actually open a connection**. It validates the driver name and DSN syntax, then returns a `*sql.DB` that is a lazy handle to a connection pool. The first connection happens on the first query. Always call `db.Ping()` (or `PingContext`) at startup to fail fast on bad config:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
if err := db.PingContext(ctx); err != nil {
    log.Fatalf("db ping: %v", err)
}
```

**Driver choice for Postgres** (the Go ecosystem has two):

| Driver | Status | Notes |
|---|---|---|
| `lib/pq` | Older, still widely used | Pure-Go, stable, but development is slow. Many shops are migrating off it. |
| `pgx/v5/stdlib` | Active, recommended | Same author also maintains `pgx`'s native API. Better Postgres feature support, automatic prepared-statement caching, `LISTEN`/`NOTIFY`, `COPY` protocol. |

If you are starting a new service on Postgres in 2026, use `pgx` — either via the `database/sql` adapter (`pgx/v5/stdlib`) or its native API (§10).

## 3. The connection pool

`*sql.DB` is a pool, not a connection. Every `Query`, `Exec`, or `Begin` checks out a connection, runs the work, and returns it. The pool has four knobs, and the defaults are wrong for production:

```go
db.SetMaxOpenConns(25)                  // hard cap on open connections
db.SetMaxIdleConns(25)                  // pool keeps this many warm
db.SetConnMaxLifetime(5 * time.Minute)  // close & rotate after this age
db.SetConnMaxIdleTime(2 * time.Minute)  // close after idle this long
```

Defaults if you set nothing:

| Setting | Default | Why it bites |
|---|---|---|
| `MaxOpenConns` | **0 = unlimited** | A traffic spike opens 5,000 connections and Postgres crashes. |
| `MaxIdleConns` | 2 | Way too few — every request churns connection setup. |
| `ConnMaxLifetime` | 0 = forever | Long-lived connections accumulate planner memory and miss DNS changes (load balancer reroutes). |
| `ConnMaxIdleTime` | 0 = forever | Same problem; idle connections stick around through deploys. |

**Sizing rule of thumb.** Postgres does not parallelise within a connection. The right `MaxOpenConns` is roughly:

```text
MaxOpenConns ≈ (avg request concurrency on this instance) × (avg queries per request)
```

…capped at whatever fraction of the database's `max_connections` budget your service is allowed. If you run 10 instances and Postgres allows 200 connections, give each instance about 20 (leave headroom for migrations, admin, and a connection pooler like PgBouncer).

`MaxIdleConns ≤ MaxOpenConns`. If `MaxIdleConns` is 0 (or unset and lower than the open cap), connections close immediately when idle and you pay TLS handshake on every checkout. Keep them equal, or close to it.

`ConnMaxLifetime` of 5–30 minutes is standard. It lets the pool gradually rotate so that DNS / cluster topology changes propagate.

`ConnMaxIdleTime` complements lifetime — close connections that nobody is using, so a quiet service does not pin connections it does not need.

The Prisma equivalent is the `connection_limit` URL parameter (single knob, no idle-time control). The JPA / HikariCP equivalents are `maximumPoolSize`, `minimumIdle`, `maxLifetime`, `idleTimeout` — same four concepts, same defaults-are-wrong story.

## 4. Always use the `Context` variants

`database/sql` exposes both `Query/Exec/QueryRow` and `QueryContext/ExecContext/QueryRowContext`. Use the `Context` versions exclusively. Without a context, the query has no cancellation path: if the client disconnects or the request deadline hits, the goroutine will still wait for Postgres to return.

```go
func (r *UserRepo) GetByID(ctx context.Context, id string) (*User, error) {
    const q = `SELECT id, email, created_at FROM users WHERE id = $1`

    var u User
    err := r.db.QueryRowContext(ctx, q, id).Scan(&u.ID, &u.Email, &u.CreatedAt)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("getByID: %w", err)
    }
    return &u, nil
}
```

When the context is cancelled mid-query:

- The driver issues a cancellation message to the database (Postgres receives a `CancelRequest`).
- The connection is returned to the pool.
- `QueryContext` returns `context.Canceled` or `context.DeadlineExceeded` (use `errors.Is` to detect).

Callers must propagate the request `ctx` all the way down. See [Context Propagation](../concurrency/04-context-propagation.md) for the full pattern — context is the single most important thing in any Go service that talks to a database.

## 5. Scanning rows and `NULL` handling

`Scan` takes pointers to destinations in **column order** — not by name. The slot count and types must match the SELECT. Mismatches are runtime errors, not compile-time errors, which is the biggest single ergonomic gap versus Prisma.

```go
rows, err := db.QueryContext(ctx, `
    SELECT id, email, last_login_at FROM users WHERE active = true
`)
if err != nil {
    return nil, err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Email, &u.LastLoginAt); err != nil {
        return nil, err
    }
    users = append(users, u)
}
if err := rows.Err(); err != nil {  // check for iteration errors
    return nil, err
}
```

Three rules that catch everyone:

1. **`defer rows.Close()` always.** Forgetting this leaks the connection back to the pool. The `for rows.Next()` loop closes automatically only when it iterates to completion; an early return, panic, or break leaks.
2. **Check `rows.Err()` after the loop.** `Next()` returning `false` can mean either "done" or "errored mid-iteration". `rows.Err()` distinguishes them.
3. **`Scan` types must accommodate `NULL`.** A `string` cannot hold `NULL`; the driver will error.

For nullable columns, two options:

```go
// Option A: sql.Null* wrappers (stdlib)
var lastLogin sql.NullTime
err := row.Scan(&u.ID, &u.Email, &lastLogin)
if lastLogin.Valid {
    u.LastLoginAt = &lastLogin.Time
}

// Option B: pointer destinations
var lastLoginAt *time.Time
err := row.Scan(&u.ID, &u.Email, &lastLoginAt)
// lastLoginAt is nil if NULL
```

The standard library wrappers: `sql.NullString`, `sql.NullInt64`, `sql.NullInt32`, `sql.NullInt16`, `sql.NullByte`, `sql.NullFloat64`, `sql.NullBool`, `sql.NullTime`. Each has a `.Valid bool` field and a typed value field.

`pgx` adds richer types via the `pgtype` package: `pgtype.Numeric` for arbitrary-precision decimals, `pgtype.UUID`, `pgtype.JSONB`, `pgtype.Interval`, `pgtype.Range[T]`. With the `pgx` native API (§10), Postgres types round-trip with full fidelity.

## 6. Prepared statements

A prepared statement parses and plans a query once, then executes it many times with different parameters. Two ways to use them:

**Implicit (recommended for most cases).** When you call `db.QueryContext` with placeholders, the driver may prepare the statement under the hood, run it, and either cache or discard the prepared plan.

- `pgx` automatically prepares and caches statements per connection (with an LRU). After the first call, subsequent calls reuse the plan.
- `lib/pq` does **not** auto-cache — every call prepares and discards.

**Explicit.** You call `db.PrepareContext`, get a `*sql.Stmt`, and reuse it.

```go
stmt, err := db.PrepareContext(ctx, `SELECT id, email FROM users WHERE id = $1`)
if err != nil {
    return err
}
defer stmt.Close()

for _, id := range ids {
    var u User
    if err := stmt.QueryRowContext(ctx, id).Scan(&u.ID, &u.Email); err != nil {
        return err
    }
    process(u)
}
```

**When prepared statements help:**
- Hot queries called thousands of times with different parameters.
- Saves parse + plan time on each call (in Postgres, this is non-trivial for complex queries).

**When they hurt:**
- One-shot queries — preparation cost exceeds execution cost.
- **PgBouncer in transaction mode.** PgBouncer multiplexes one Postgres connection across many client connections per transaction; prepared statements are tied to a Postgres backend and break under transaction-mode pooling. Disable auto-prepare in `pgx` (`config.DefaultQueryExecMode = pgx.QueryExecModeExec`) or use session-mode pooling.

The Prisma equivalent: Prisma uses prepared statements automatically and exposes `pgbouncer=true` in the URL to disable them when needed. Same trade-off, different surface.

## 7. Transactions and the deferred-rollback idiom

```go
func transfer(ctx context.Context, db *sql.DB, from, to string, amount int) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        return fmt.Errorf("begin: %w", err)
    }
    defer tx.Rollback()  // safe no-op if Commit succeeded

    if _, err := tx.ExecContext(ctx,
        `UPDATE accounts SET balance = balance - $1 WHERE id = $2`,
        amount, from,
    ); err != nil {
        return fmt.Errorf("debit: %w", err)
    }

    if _, err := tx.ExecContext(ctx,
        `UPDATE accounts SET balance = balance + $1 WHERE id = $2`,
        amount, to,
    ); err != nil {
        return fmt.Errorf("credit: %w", err)
    }

    return tx.Commit()
}
```

The **deferred-rollback idiom** is the standard pattern. After `Commit` succeeds, the transaction is finished and the deferred `Rollback` is a no-op (it returns `sql.ErrTxDone` which we ignore). On any error path before `Commit`, the deferred `Rollback` cleans up. You never have to remember to call `Rollback` on error branches — the `defer` does it.

**Isolation levels** are passed via `sql.TxOptions`:

| Level | Constant |
|---|---|
| Read Committed (Postgres default) | `sql.LevelReadCommitted` |
| Repeatable Read | `sql.LevelRepeatableRead` |
| Serializable | `sql.LevelSerializable` |

Postgres `Serializable` is true SSI (Serializable Snapshot Isolation) — it can fail at commit with `40001 serialization_failure`. The application must retry. This is the Go equivalent of Prisma's `Prisma.PrismaClientKnownRequestError` with code `P2034`.

```go
for attempt := 0; attempt < 3; attempt++ {
    err := transfer(ctx, db, from, to, amount)
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) && pgErr.Code == "40001" {
        time.Sleep(backoff(attempt))
        continue
    }
    return err
}
```

`tx` has the same `QueryContext` / `ExecContext` / `QueryRowContext` methods as `*sql.DB`. Pass `*sql.Tx` (or an interface that both satisfy) to repository functions when they participate in a transaction.

## 8. `sql.ErrNoRows` is a sentinel

`QueryRowContext` returns at most one row. If the SELECT returned zero rows, `Scan` returns `sql.ErrNoRows`. This is a **sentinel error** — see [Errors as Values §2](../fundamentals/06-errors-as-values.md#2-sentinel-errors-and-errorsis) for the broader pattern.

```go
err := db.QueryRowContext(ctx, q, id).Scan(&u.ID, &u.Email)
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrUserNotFound  // map to a domain error
}
if err != nil {
    return nil, fmt.Errorf("getUser: %w", err)
}
```

Use `errors.Is`, not `==`. As soon as a wrapper layer does `fmt.Errorf("...: %w", err)`, equality breaks but `Is` keeps working.

The Prisma equivalent: `findUnique` returns `null` for missing rows, `findUniqueOrThrow` throws `P2025`. JPA's `findById` returns `Optional<T>`, `getReferenceById` throws `EntityNotFoundException`. Go's "missing row is an error you check for explicitly" is in between: more friction than `null`, less surprise than an exception bubbling through layers.

## 9. `sqlx` — extension, not ORM

`jmoiron/sqlx` is the most common quality-of-life layer. It does not replace `database/sql`; it embeds it. A `*sqlx.DB` **is a** `*sql.DB`, plus extra methods.

What it adds:

- **`StructScan` / `Get` / `Select`.** Scan a row into a struct by column name (with `db:"col_name"` tags) instead of positional `Scan`.
- **`NamedExec` / `NamedQuery`.** Use `:name` placeholders bound from a struct or map.
- **`In` query expansion.** Expand `WHERE id IN (?)` against a slice into `WHERE id IN (?, ?, ?)` with the right number of placeholders.

```go
type User struct {
    ID        string    `db:"id"`
    Email     string    `db:"email"`
    CreatedAt time.Time `db:"created_at"`
}

var users []User
err := dbx.SelectContext(ctx, &users,
    `SELECT id, email, created_at FROM users WHERE active = $1`, true,
)
// users is fully populated by reflection over `db` tags
```

What it does **not** add:

- No query builder. You still write SQL strings.
- No migrations, no schema management.
- No relations, no eager loading, no identity map.
- No code generation.

`sqlx` is a sweet spot for teams that want struct scanning without buying into the cost model of a full ORM. The reflection cost is real but small (reflection happens once per query shape, not per row in any tight loop). For services where every microsecond counts, see `sqlc` (next doc) which generates the scan code at compile time.

## 10. `pgx` — bypass `database/sql` for Postgres

`jackc/pgx` is a Postgres driver and toolkit. It can be used in two modes:

**As a `database/sql` driver.** Import `_ "github.com/jackc/pgx/v5/stdlib"` and call `sql.Open("pgx", dsn)`. You get the standard `*sql.DB` API with pgx's wire implementation.

**As a native API.** Import `github.com/jackc/pgx/v5` directly. You get `*pgx.Conn` and `*pgxpool.Pool` with a Postgres-specific API.

```go
import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
)

pool, err := pgxpool.New(ctx, "postgres://user:pass@localhost:5432/mydb")
if err != nil {
    return err
}
defer pool.Close()

var u User
err = pool.QueryRow(ctx, `SELECT id, email FROM users WHERE id = $1`, id).
    Scan(&u.ID, &u.Email)
```

Why pick the native API over the `database/sql` adapter:

- **Real Postgres types.** `pgtype.UUID`, `pgtype.JSONB`, `pgtype.Numeric`, range types, arrays — round-trip without manual conversion.
- **Lower per-call overhead.** No `database/sql` pool adapter; `pgxpool` is a Postgres-aware pool.
- **Better protocol features.** `COPY FROM` / `COPY TO` for bulk loads, `LISTEN`/`NOTIFY` for pub/sub, batched queries (`pgx.Batch`).
- **Automatic statement caching.** Per-connection LRU of prepared statements (transparent to your code).

Why stay on the `database/sql` adapter:

- You want to swap drivers later without changing call sites.
- Library you depend on (e.g., `sqlx`) takes `*sql.DB`.
- Team familiarity with the standard interface.

| | `lib/pq` | `pgx` (database/sql adapter) | `pgx` (native) |
|---|---|---|---|
| Driver name | `"postgres"` | `"pgx"` | n/a |
| Auto-prepare | No | Yes | Yes |
| Postgres types | basic | basic | full (`pgtype`) |
| `COPY` / `LISTEN` | partial | partial | full |
| Active development | slow | active | active |
| Recommendation | legacy systems only | new code, want stdlib API | new code, max perf and features |

Many production services in 2026 run native `pgx` with `sqlc`-generated query code. That stack gets you compile-time-checked SQL and full Postgres type fidelity with no runtime reflection — a different position on the abstraction spectrum than Prisma.

## 11. Migrations: golang-migrate, goose, atlas

`database/sql` has no opinion on schema. Three Go-ecosystem tools fill the gap:

### golang-migrate

The most widely adopted. Each migration is a numbered pair of SQL files:

```text
migrations/
├── 0001_create_users.up.sql
├── 0001_create_users.down.sql
├── 0002_add_email_index.up.sql
└── 0002_add_email_index.down.sql
```

```bash
migrate -path ./migrations -database "postgres://..." up
migrate -path ./migrations -database "postgres://..." down 1
```

Strengths: simple model, large database support (Postgres, MySQL, SQLite, Spanner, MongoDB, ClickHouse, more). Weaknesses: SQL-only by default; data migrations that need application logic must be written as raw SQL.

### pressly/goose

Numbered migrations, but supports both SQL and Go:

```go
// 20260503120000_backfill_user_status.go
package migrations

import (
    "context"
    "database/sql"
    "github.com/pressly/goose/v3"
)

func init() {
    goose.AddMigrationContext(upBackfill, downBackfill)
}

func upBackfill(ctx context.Context, tx *sql.Tx) error {
    _, err := tx.ExecContext(ctx,
        `UPDATE users SET status = 'active' WHERE last_login_at > NOW() - INTERVAL '30 days'`)
    return err
}

func downBackfill(ctx context.Context, tx *sql.Tx) error { return nil }
```

Strengths: Go migrations let you do anything (call APIs, multi-step transformations). Embedded migrations via `embed.FS`. Weaknesses: smaller database matrix than golang-migrate.

### Atlas

Schema-as-code, declarative or versioned. You write the desired schema in HCL (or import from an ORM, or introspect the database), and Atlas computes the diff:

```hcl
table "users" {
  schema = schema.public
  column "id" {
    type = uuid
    null = false
  }
  column "email" {
    type = text
    null = false
  }
  primary_key { columns = [column.id] }
  index "users_email_uniq" {
    unique  = true
    columns = [column.email]
  }
}
```

```bash
atlas migrate diff add_email_index --to "file://schema.hcl" --dev-url "..."
atlas migrate apply --url "postgres://..."
```

Strengths: declarative model (Terraform-like), strong CI integration, can lint migrations for unsafe operations (e.g., dropping a column with a NOT NULL constraint), supports many databases, integrates with Ent. Weaknesses: more concepts to learn; the HCL schema language is one more thing to keep in your head.

| Tool | Model | Migration format | Best for |
|---|---|---|---|
| golang-migrate | Versioned | SQL up/down pairs | Most teams; broad DB support |
| goose | Versioned | SQL or Go | Teams that need data migrations with app logic |
| Atlas | Declarative or versioned | HCL or SQL | Teams that want schema diffing and CI lint gates |

The TypeScript / Java parallels:

| Concern | Go | TypeScript | Java |
|---|---|---|---|
| Versioned SQL migrations | golang-migrate, goose | Drizzle Kit, Knex | Flyway |
| Declarative schema diff | Atlas | Prisma Migrate | Liquibase (changelog), Atlas |
| Schema in code | Atlas HCL or Ent | Prisma DSL | JPA `@Entity` (with Hibernate `ddl-auto`) |

A pragmatic combo for a new Go + Postgres service in 2026: `pgx` (native) + `sqlc` for queries + `golang-migrate` or `Atlas` for schema. No ORM, no reflection on the hot path, all SQL visible.

## Related

- [Errors as Values](../fundamentals/06-errors-as-values.md) — `sql.ErrNoRows` as a sentinel, `errors.Is` over `==`
- [Context Propagation](../concurrency/04-context-propagation.md) — why `QueryContext` matters, request cancellation reaching the database
- [sqlc vs GORM vs Ent](02-sqlc-gorm-ent.md) — the next layer up: code generation versus runtime reflection ORMs
- [Prisma Deep Dive](../../typescript/data-access/prisma-deep-dive.md) — the TypeScript ORM Go's stack is intentionally not
- [Prisma vs Drizzle](../../typescript/data-access/prisma-vs-drizzle.md) — abstraction-spectrum framing for data-access tools
- [Database INDEX](../../database/INDEX.md) — Postgres internals, indexing, transactions

## References

- `database/sql` package documentation — https://pkg.go.dev/database/sql
- `database/sql/driver` interfaces — https://pkg.go.dev/database/sql/driver
- `pgx` (jackc) — https://github.com/jackc/pgx
- `pgxpool` package docs — https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool
- `lib/pq` — https://github.com/lib/pq
- `sqlx` (jmoiron) — https://github.com/jmoiron/sqlx
- `golang-migrate/migrate` — https://github.com/golang-migrate/migrate
- `pressly/goose` — https://github.com/pressly/goose
- Atlas — https://atlasgo.io/
- Postgres `max_connections` and pooling — https://www.postgresql.org/docs/current/runtime-config-connection.html
- Postgres SSI (`40001` serialization_failure) — https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE
