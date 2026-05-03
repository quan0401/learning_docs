---
title: "Integration & Contract Testing"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, testing, integration, testcontainers, contract-testing]
---

# Integration & Contract Testing

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `testing` `integration` `testcontainers` `contract-testing`

---

## Table of Contents

1. [The integration vs unit boundary](#1-the-integration-vs-unit-boundary)
2. [`testcontainers-go` — real dependencies in Docker](#2-testcontainers-go--real-dependencies-in-docker)
3. [End-to-end HTTP routes with `httptest.Server`](#3-end-to-end-http-routes-with-httptestserver)
4. [Database setup and teardown patterns](#4-database-setup-and-teardown-patterns)
5. [Golden files](#5-golden-files)
6. [Contract testing with Pact](#6-contract-testing-with-pact)
7. [CI considerations](#7-ci-considerations)
8. [Build tags for integration tests](#8-build-tags-for-integration-tests)

## Summary

Unit tests verify a single function or struct in isolation; integration tests verify that multiple components — your code plus a real database, message broker, or downstream service — agree about how to talk to each other. The Go community lands closer to the integration end of the test pyramid than the JS world does, because the tools make it cheap: `testcontainers-go` boots a real PostgreSQL in three seconds, `httptest.NewServer` is in the standard library, and `go test` happily runs tests in parallel.

This doc covers the integration-test toolkit (testcontainers, golden files, build tags), the boundary between unit and integration suites, and the contract-testing layer that complements both. The recurring theme: prefer real dependencies over elaborate mocks, gate slow tests behind build tags, and let CI run them on a separate lane.

## 1. The integration vs unit boundary

There is no universally agreed definition; pragmatic working definitions:

| Layer | What it tests | Speed | Dependencies |
|---|---|---|---|
| Unit | One function or struct, in isolation | <10ms | none — everything mocked or fake |
| Integration | Your code + one real external system | 50ms-2s | testcontainers (DB, Redis), httptest server |
| End-to-end | Whole service, including HTTP boundary | 1-30s | full app stack, real config |
| Contract | Service A's expectations vs B's behavior | per-pair | shared contract artifact |

In a typical Go service, you'll have:

- 80% unit tests in package-local `*_test.go` files (using fakes from [02-mocking-and-test-doubles.md](02-mocking-and-test-doubles.md))
- 15% integration tests, often in a separate `internal/.../integration_test.go` with a build tag
- 5% end-to-end tests that wire the whole binary

The boundary that matters: **a unit test never calls Docker, never opens a TCP port, and never blocks on I/O it didn't mock.** Once you cross any of those lines, you're in integration territory and you should know it (gate it, isolate it, run it on its own CI lane).

## 2. `testcontainers-go` — real dependencies in Docker

`github.com/testcontainers/testcontainers-go` boots disposable Docker containers from Go test code. The library has 70+ pre-built modules (postgres, redis, mysql, kafka, mongodb, localstack, neo4j, vault, …) and a generic `ContainerRequest` for anything else.

The argument for using it over a long-lived dev database: each test (or test package) gets a fresh container, so tests can't leak schema changes into one another. Setup is ~3 seconds for Postgres on a warm Docker daemon, which is faster than chasing flakes from a shared instance.

### Postgres example with the dedicated module

```go
package userrepo_test

import (
    "context"
    "database/sql"
    "testing"

    _ "github.com/jackc/pgx/v5/stdlib"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

func newPostgres(t *testing.T) *sql.DB {
    t.Helper()
    ctx := context.Background()

    pgC, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("appdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2),
        ),
    )
    if err != nil {
        t.Fatalf("postgres.Run: %v", err)
    }
    t.Cleanup(func() {
        _ = testcontainers.TerminateContainer(pgC)
    })

    dsn, err := pgC.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("ConnectionString: %v", err)
    }

    db, err := sql.Open("pgx", dsn)
    if err != nil {
        t.Fatalf("sql.Open: %v", err)
    }
    t.Cleanup(func() { db.Close() })

    if err := runMigrations(db); err != nil {
        t.Fatalf("migrations: %v", err)
    }
    return db
}

func TestUserRepo_FindByID_integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in -short mode")
    }
    db := newPostgres(t)
    repo := NewUserRepo(db)

    if err := seed(db, &User{ID: "u1", Email: "alice@example.com"}); err != nil {
        t.Fatalf("seed: %v", err)
    }
    got, err := repo.FindByID(context.Background(), "u1")
    if err != nil {
        t.Fatalf("FindByID: %v", err)
    }
    if got.Email != "alice@example.com" {
        t.Errorf("email = %q", got.Email)
    }
}
```

Notes:

- `postgres.Run` is the modern entry point (replacing the older `postgres.RunContainer`). It handles the image, waits for readiness, and returns a `*postgres.PostgresContainer`.
- The wait strategy `wait.ForLog(...).WithOccurrence(2)` is the canonical Postgres readiness check — Postgres logs the message twice during startup (once when the cluster is initialized, once when it's actually accepting connections).
- `t.Cleanup(testcontainers.TerminateContainer)` is preferable to `defer` because it runs even on `t.Fatal`.

### Sharing a container across tests in a package

Booting Postgres for every single test is overkill. The common pattern is one container per test package, set up in `TestMain`:

```go
var sharedDB *sql.DB

func TestMain(m *testing.M) {
    ctx := context.Background()
    pgC, err := postgres.Run(ctx, "postgres:16-alpine", /* opts */)
    if err != nil {
        log.Fatalf("postgres: %v", err)
    }
    defer testcontainers.TerminateContainer(pgC)

    dsn, _ := pgC.ConnectionString(ctx, "sslmode=disable")
    sharedDB, _ = sql.Open("pgx", dsn)
    defer sharedDB.Close()

    if err := runMigrations(sharedDB); err != nil {
        log.Fatalf("migrate: %v", err)
    }

    os.Exit(m.Run())
}
```

Tests in the package then use `sharedDB` and rely on per-test isolation (next section).

## 3. End-to-end HTTP routes with `httptest.Server`

`httptest.Server` (covered for client-side stubbing in [02-mocking-and-test-doubles.md §3](02-mocking-and-test-doubles.md)) also serves the reverse direction: starting your own router on a real port to exercise the full request pipeline.

```go
func TestAPIServer_e2e(t *testing.T) {
    db := newPostgres(t)
    api := NewAPI(db, /* deps */)

    srv := httptest.NewServer(api.Router())  // your real chi/echo/std-lib router
    t.Cleanup(srv.Close)

    body := strings.NewReader(`{"email":"new@example.com"}`)
    res, err := srv.Client().Post(srv.URL+"/users", "application/json", body)
    if err != nil {
        t.Fatalf("POST: %v", err)
    }
    defer res.Body.Close()

    if res.StatusCode != http.StatusCreated {
        t.Errorf("status = %d; want 201", res.StatusCode)
    }

    // Verify side effect against the real DB.
    var count int
    _ = db.QueryRow(`SELECT COUNT(*) FROM users WHERE email = $1`, "new@example.com").Scan(&count)
    if count != 1 {
        t.Errorf("user count = %d; want 1", count)
    }
}
```

This single test exercises HTTP parsing, your router, the handler, the service layer, the repository, and the database — the entire vertical slice. Each component is the production one. When this passes, you have high confidence the wiring is right.

## 4. Database setup and teardown patterns

Three patterns, in order of preference for most projects:

### Pattern A: Transactional rollback per test

Each test runs inside a transaction that's rolled back at the end. Fast (no schema reset), strong isolation, but doesn't work if your code itself manages transactions.

```go
func newTx(t *testing.T, db *sql.DB) *sql.Tx {
    t.Helper()
    tx, err := db.BeginTx(context.Background(), nil)
    if err != nil {
        t.Fatalf("begin: %v", err)
    }
    t.Cleanup(func() { _ = tx.Rollback() })
    return tx
}

func TestRepoInsert(t *testing.T) {
    tx := newTx(t, sharedDB)
    repo := NewUserRepo(tx)  // repo accepts an interface that *sql.Tx satisfies
    // ... test body ...
}
```

The catch: your repository interfaces must accept `*sql.Tx` (or the common `interface { ExecContext(...); QueryRowContext(...) }`), not just `*sql.DB`. This is good design anyway — it lets the service layer compose multiple repository calls into one transaction.

### Pattern B: Truncate tables per test

Reset known tables before each test. Slower than rollback, but works when the code under test owns its own transactions.

```go
func resetDB(t *testing.T, db *sql.DB) {
    t.Helper()
    _, err := db.Exec(`TRUNCATE users, orders, payments RESTART IDENTITY CASCADE`)
    if err != nil {
        t.Fatalf("truncate: %v", err)
    }
}
```

`RESTART IDENTITY` resets sequences so generated IDs are predictable. `CASCADE` follows foreign keys.

### Pattern C: Fresh schema per test

Drop and recreate the schema between tests. Strongest isolation, but slow — only worth it for tests that exercise migrations themselves, or when subtle schema state matters.

```go
func freshSchema(t *testing.T, db *sql.DB) {
    t.Helper()
    _, _ = db.Exec(`DROP SCHEMA public CASCADE; CREATE SCHEMA public;`)
    if err := runMigrations(db); err != nil {
        t.Fatalf("migrate: %v", err)
    }
}
```

In practice, most teams pick A or B for application tests and keep C for migration tests.

## 5. Golden files

Golden file testing — also called snapshot testing in the JS world — compares output against a checked-in reference file. Most useful when the output is large or structured (rendered HTML, formatted JSON, generated code) where writing an inline assertion would be tedious and the diff against a known-good is far more readable.

The convention: store goldens under `testdata/`, accept a `-update` flag to regenerate them.

```go
var update = flag.Bool("update", false, "update golden files")

func TestRenderInvoice_golden(t *testing.T) {
    inv := Invoice{ID: "inv-1", Items: []Item{{SKU: "A", Price: 100}}}
    got, err := RenderInvoice(inv)
    if err != nil {
        t.Fatalf("Render: %v", err)
    }

    goldenPath := filepath.Join("testdata", "invoice.golden.html")

    if *update {
        if err := os.WriteFile(goldenPath, got, 0644); err != nil {
            t.Fatalf("write golden: %v", err)
        }
        return
    }

    want, err := os.ReadFile(goldenPath)
    if err != nil {
        t.Fatalf("read golden: %v (run with -update to create)", err)
    }
    if !bytes.Equal(got, want) {
        t.Errorf("output mismatch:\n--- got ---\n%s\n--- want ---\n%s", got, want)
    }
}
```

Run normally:

```bash
go test ./...
```

Regenerate when intentionally changing output:

```bash
go test -update ./...
git diff testdata/  # review what changed
```

When goldens are worth it: large, structured, deterministic outputs that change rarely. When they aren't: small outputs (just write the assertion inline), non-deterministic content (timestamps, UUIDs), or anything where the failure message would be more useful than a diff.

A common pitfall: tests pass after `-update` even if the new output is wrong. Always **review the diff** before committing.

## 6. Contract testing with Pact

When two services depend on each other, integration tests on either side can pass while the actual interaction breaks at runtime — the consumer assumes a JSON shape the provider doesn't produce, or vice versa. Contract testing closes that gap by capturing the consumer's expectations as a checked-in artifact (the "pact"), then verifying the provider satisfies it.

`pact-go` v2 is the Go binding for the Pact framework.

### Consumer side

```go
import (
    "github.com/pact-foundation/pact-go/v2/consumer"
    "github.com/pact-foundation/pact-go/v2/matchers"
)

func TestUserClient_Pact(t *testing.T) {
    mockProvider, err := consumer.NewV2Pact(consumer.MockHTTPProviderConfig{
        Consumer: "BillingService",
        Provider: "UserService",
    })
    if err != nil {
        t.Fatal(err)
    }

    err = mockProvider.
        AddInteraction().
        Given("user u1 exists").
        UponReceiving("a request for user u1").
        WithRequest("GET", "/users/u1").
        WillRespondWith(200, func(b *consumer.V2ResponseBuilder) {
            b.Header("Content-Type", matchers.S("application/json"))
            b.JSONBody(matchers.MapMatcher{
                "id":    matchers.S("u1"),
                "email": matchers.Like("alice@example.com"),
            })
        }).
        ExecuteTest(t, func(srv consumer.MockServerConfig) error {
            client := NewUserClient(fmt.Sprintf("http://%s:%d", srv.Host, srv.Port))
            user, err := client.Get(context.Background(), "u1")
            if err != nil {
                return err
            }
            if user.ID != "u1" {
                return fmt.Errorf("got id %q", user.ID)
            }
            return nil
        })
    if err != nil {
        t.Fatal(err)
    }
}
```

This run produces `pacts/billingservice-userservice.json`. Push it to a Pact Broker (or check it into a shared repo).

### Provider side

The provider service runs `pact-go verify` against the contract, which replays the recorded interactions against a live instance and confirms each expected response.

```go
import "github.com/pact-foundation/pact-go/v2/provider"

func TestProvider_Pact(t *testing.T) {
    srv := startTestServer(t)  // boot the real HTTP service
    err := provider.NewVerifier().VerifyProvider(t, provider.VerifyRequest{
        ProviderBaseURL: srv.URL,
        BrokerURL:       os.Getenv("PACT_BROKER_URL"),
        Provider:        "UserService",
        StateHandlers: provider.StateHandlers{
            "user u1 exists": func(setup bool, state provider.ProviderState) (provider.ProviderStateResponse, error) {
                if setup {
                    seed(testDB, &User{ID: "u1", Email: "alice@example.com"})
                }
                return nil, nil
            },
        },
    })
    if err != nil {
        t.Fatal(err)
    }
}
```

When contract testing is worth it: independently-deployed services owned by different teams. When it isn't: a monorepo where consumer and provider deploy together — a regular integration test gives the same coverage with less ceremony.

`pact-go` v2 requires CGo and platform-specific Pact FFI binaries (run `pact-go install` once after `go install`).

## 7. CI considerations

### Parallelism

`go test` parallelizes at three levels: across packages by default (`-p N`), within a package via `t.Parallel()`, and `-parallel N` bounds the latter. For integration tests touching shared resources, parallelism is usually a foot-gun — set `-p 1` for the integration suite or use per-test isolation (each test gets its own container).

```bash
# Unit tests: full parallelism
go test -short -race ./...

# Integration tests: serialized, with a longer timeout
go test -tags=integration -p 1 -timeout=10m ./...
```

### Docker-in-Docker (DinD)

Most CI runners ship with Docker. GitHub Actions exposes the host Docker daemon at `/var/run/docker.sock`; testcontainers-go picks it up automatically. On GitLab and Jenkins agents that use DinD, set `TESTCONTAINERS_HOST_OVERRIDE` if container-to-container networking has hostname issues.

### Caching

`go test` caches results keyed by source files, environment variables read via `os.Getenv` (the toolchain tracks them), and command-line flags. Integration tests that depend on Docker state can break the cache — testcontainers' container IDs vary per run, but those don't affect `go test`'s cache because they aren't read until inside `m.Run`. If your CI re-runs unchanged tests, that's the cache working as intended; pass `-count=1` to bypass.

### Image pinning

Pin testcontainers images by tag (`postgres:16-alpine`) or, better, by digest in CI:

```go
postgres.Run(ctx, "postgres@sha256:abc123...")
```

A floating tag means a tag-update on Docker Hub can break tests overnight. Digest pinning makes the container input deterministic.

## 8. Build tags for integration tests

Slow tests should be opt-in. Go's `//go:build` directive (replacing the older `// +build`) gates a file behind a tag.

```go
//go:build integration

package userrepo_test

// ... Postgres-dependent tests ...
```

The blank line after the directive is required. Files with this header compile and run **only** when `-tags=integration` is passed:

```bash
# Default — skips integration files
go test ./...

# Include integration tests
go test -tags=integration ./...

# Combine multiple tags
go test -tags="integration e2e" ./...
```

Common tag conventions across Go projects:

| Tag | Typical meaning |
|---|---|
| `integration` | needs Docker / external deps |
| `e2e` | full-stack, longer-running |
| `slow` | takes >1s; excluded from fast feedback |
| `manual` | only run when explicitly invoked |

A complementary pattern: use `testing.Short()` inside integration tests so `go test -short ./...` skips them even without tag gating. Pick one mechanism per project; mixing both confuses contributors.

CI typically runs unit tests on every commit and integration tests on a separate workflow that gates merging to main:

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    steps:
      - run: go test -race -short -count=1 ./...

  integration:
    services:
      docker: { image: docker:dind }
    steps:
      - run: go test -tags=integration -p 1 -timeout=10m -count=1 ./...
```

## Related

- [Testing Package Fundamentals](01-testing-package-fundamentals.md) — `t.Cleanup`, `TestMain`, `testing.Short`
- [Mocking & Test Doubles in Go](02-mocking-and-test-doubles.md) — `httptest`, `sqlmock`, the unit-test side
- [Errors as Values](../fundamentals/06-errors-as-values.md) — error assertions across boundaries
- [Goroutines and Scheduler](../concurrency/01-goroutines-and-scheduler.md) — goroutine leak detection in integration tests
- [`sync` Package and Memory Model](../concurrency/03-sync-package-and-memory-model.md) — race detector on integration suites
- [Integration & API Testing in TS](../../typescript/testing/integration-and-api-testing.md) — Supertest + Testcontainers on the TS side
- [Mocking & Test Doubles in Vitest](../../typescript/testing/mocking-and-test-doubles.md) — TS counterpart
- Java side: JUnit 5 + Testcontainers + REST Assured under `../../java/`

## References

- `testcontainers-go` site — https://golang.testcontainers.org/
- `testcontainers-go` repository — https://github.com/testcontainers/testcontainers-go
- `net/http/httptest` — https://pkg.go.dev/net/http/httptest
- `pact-go` v2 — https://github.com/pact-foundation/pact-go
- Pact specification — https://docs.pact.io/
- `cmd/go` build constraints — https://pkg.go.dev/cmd/go#hdr-Build_constraints
- `cmd/go` test caching — https://pkg.go.dev/cmd/go#hdr-Test_packages
- "Test Pyramid" (Cohn) — https://martinfowler.com/articles/practical-test-pyramid.html
- Postgres `TRUNCATE` semantics — https://www.postgresql.org/docs/current/sql-truncate.html
