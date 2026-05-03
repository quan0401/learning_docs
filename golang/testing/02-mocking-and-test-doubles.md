---
title: "Mocking & Test Doubles in Go"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, testing, mocking, test-doubles, dependency-injection]
---

# Mocking & Test Doubles in Go

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `testing` `mocking` `test-doubles` `dependency-injection`

---

## Table of Contents

1. [Interfaces are the mocking story](#1-interfaces-are-the-mocking-story)
2. [Hand-rolled fakes — the default](#2-hand-rolled-fakes--the-default)
3. [`httptest` for HTTP servers and clients](#3-httptest-for-http-servers-and-clients)
4. [`gomock` (uber-go/mock)](#4-gomock-uber-gomock)
5. [`testify/mock`](#5-testifymock)
6. [`go-sqlmock` for `database/sql`](#6-go-sqlmock-for-databasesql)
7. [Time, randomness, network — injecting non-determinism](#7-time-randomness-network--injecting-non-determinism)
8. [Avoiding over-mocking](#8-avoiding-over-mocking)
9. [Comparison with Vitest mocks and Mockito](#9-comparison-with-vitest-mocks-and-mockito)

## Summary

Go has no class hierarchy, no reflection-based mock generators built in, and no `@MockBean`-style framework. What it has is **interfaces** — implicit, structural, satisfied by anything that has the right method set. That single feature absorbs nearly all the mocking work that Mockito and Vitest's `vi.mock` do in their respective ecosystems.

The idiomatic order of preference is: **hand-rolled fake first**, **`httptest` for HTTP**, **generated mocks (`uber-go/mock`) when the surface is wide**, and **`testify/mock` if the team is already invested**. `database/sql` has its own niche tool (`go-sqlmock`) but for serious database tests `testcontainers-go` (covered in [03-integration-and-contract-testing.md](03-integration-and-contract-testing.md)) is usually a better return on investment. Across all of them, the rule is the same: mock the boundaries you control, never the libraries you don't.

## 1. Interfaces are the mocking story

Recall from [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) that Go interfaces are satisfied implicitly. A function that takes `io.Reader` will accept `*os.File`, `*bytes.Buffer`, `strings.Reader`, or any test-only struct you write that has a `Read([]byte) (int, error)` method. That's the whole mechanism.

The "accept interfaces, return structs" guideline is what makes this work in practice. If your service depends on a concrete `*PostgresUserRepo`, mocking it requires global module substitution (think `vi.mock` of an entire file). If it depends on a `UserRepo` interface, mocking it is one struct definition.

```go
// Production code — depends on the interface, not the implementation.
type UserRepo interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, u *User) error
}

type UserService struct {
    repo UserRepo
}

func (s *UserService) Promote(ctx context.Context, id string) error {
    u, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("Promote: %w", err)
    }
    u.Role = "admin"
    return s.repo.Save(ctx, u)
}
```

In production: `service := NewUserService(postgresRepo)`. In tests: `service := NewUserService(fakeRepo)`. No framework, no annotation.

The smaller the interface, the easier the mock. The standard library's `io.Reader`, `io.Writer`, and `error` are the canonical examples. When designing your own interfaces, prefer many small ones over one large one.

## 2. Hand-rolled fakes — the default

For most service interfaces — three to ten methods — write the fake by hand. It's faster than configuring a mock framework, the test reads better, and refactors stay safe.

```go
// repo_fake_test.go
type fakeUserRepo struct {
    users map[string]*User

    // Optional: track calls for assertions.
    findCalls int
    saveCalls int

    // Optional: inject failures per-test.
    findErr error
    saveErr error
}

func newFakeUserRepo() *fakeUserRepo {
    return &fakeUserRepo{users: map[string]*User{}}
}

func (f *fakeUserRepo) FindByID(ctx context.Context, id string) (*User, error) {
    f.findCalls++
    if f.findErr != nil {
        return nil, f.findErr
    }
    u, ok := f.users[id]
    if !ok {
        return nil, ErrUserNotFound
    }
    return u, nil
}

func (f *fakeUserRepo) Save(ctx context.Context, u *User) error {
    f.saveCalls++
    if f.saveErr != nil {
        return f.saveErr
    }
    f.users[u.ID] = u
    return nil
}
```

A test using it:

```go
func TestPromote(t *testing.T) {
    repo := newFakeUserRepo()
    repo.users["u1"] = &User{ID: "u1", Role: "viewer"}

    svc := NewUserService(repo)
    if err := svc.Promote(context.Background(), "u1"); err != nil {
        t.Fatalf("Promote: %v", err)
    }

    if got := repo.users["u1"].Role; got != "admin" {
        t.Errorf("role = %q; want %q", got, "admin")
    }
    if repo.saveCalls != 1 {
        t.Errorf("Save called %d times; want 1", repo.saveCalls)
    }
}
```

Three properties of this style:

- **No magic.** What the fake does is what you wrote. There's no DSL to misread, no expectation matcher firing in unexpected order.
- **Reusable across tests.** A well-designed fake (often called an in-memory implementation) can power dozens of tests in the package. Compare to mock frameworks where each test re-stubs the same expectations.
- **Compiles or breaks loudly.** Add a method to `UserRepo` and every fake breaks at compile time. With reflective mocks you'd discover the gap at runtime.

The downside: for a 30-method interface (some legacy SDKs do this), hand-rolling is tedious. That's the case for code generation — see §4.

## 3. `httptest` for HTTP servers and clients

The `net/http/httptest` package is the right tool whenever your test crosses an HTTP boundary. Two main pieces: `httptest.NewServer` (a real HTTP server bound to localhost on a free port) and `httptest.NewRecorder` (a fake `http.ResponseWriter` for testing handlers in isolation).

### Stubbing an upstream HTTP API

```go
func TestPaymentClient_Charge(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/v1/charge" {
            t.Errorf("path = %q; want /v1/charge", r.URL.Path)
        }
        if r.Method != http.MethodPost {
            t.Errorf("method = %q; want POST", r.Method)
        }
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        _, _ = w.Write([]byte(`{"id":"ch_123","status":"ok"}`))
    }))
    t.Cleanup(srv.Close)

    client := NewPaymentClient(srv.URL, srv.Client())  // srv.Client() is preconfigured
    got, err := client.Charge(context.Background(), 1000, "USD")
    if err != nil {
        t.Fatalf("Charge: %v", err)
    }
    if got.ID != "ch_123" {
        t.Errorf("ID = %q; want ch_123", got.ID)
    }
}
```

`srv.URL` gives you a real `http://127.0.0.1:PORT/` URL — pass it to your client as the base URL. `srv.Client()` returns an `*http.Client` that trusts the test server's TLS cert (when using `httptest.NewTLSServer`) and uses standard timeouts.

This is the closest Go gets to MSW (Mock Service Worker): a real network round-trip against a server you control. URL typos are caught, headers are exercised, the `*http.Client` is the production one. It's slower than module-level mocking but far more realistic.

### Testing a handler with `httptest.NewRecorder`

When you want to test a handler without a real server:

```go
func TestUserHandler_GET(t *testing.T) {
    h := NewUserHandler(repoStub)
    req := httptest.NewRequest(http.MethodGet, "/users/u1", nil)
    rr := httptest.NewRecorder()

    h.ServeHTTP(rr, req)

    res := rr.Result()
    if res.StatusCode != http.StatusOK {
        t.Errorf("status = %d; want 200", res.StatusCode)
    }
    var body User
    if err := json.NewDecoder(res.Body).Decode(&body); err != nil {
        t.Fatalf("decode: %v", err)
    }
    if body.ID != "u1" {
        t.Errorf("body.ID = %q; want u1", body.ID)
    }
}
```

`httptest.NewRequest` builds an `*http.Request` shaped like a server-side incoming request. `httptest.NewRecorder` is a `*ResponseRecorder` whose `Code`, `Body`, and `HeaderMap` you can inspect after the handler runs. `rr.Result()` produces the `*http.Response` form, including headers.

## 4. `gomock` (uber-go/mock)

When the interface is wide, frequently changing, and used by many tests, code generation pays off. The original `golang/mock` was archived on June 27, 2023; the maintained fork is **`uber-go/mock`** at module path `go.uber.org/mock`.

Install:

```bash
go install go.uber.org/mock/mockgen@latest
```

Generate a mock — a `go:generate` directive next to the interface is the conventional placement:

```go
// repo.go
//go:generate mockgen -destination=mocks/mock_user_repo.go -package=mocks . UserRepo
type UserRepo interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, u *User) error
}
```

```bash
go generate ./...
```

The generated `mocks/mock_user_repo.go` provides a `MockUserRepo` plus an `EXPECT()` builder. Use it in tests:

```go
import (
    "go.uber.org/mock/gomock"
    "example.com/internal/user/mocks"
)

func TestPromote_gomock(t *testing.T) {
    ctrl := gomock.NewController(t)
    repo := mocks.NewMockUserRepo(ctrl)

    repo.EXPECT().
        FindByID(gomock.Any(), "u1").
        Return(&User{ID: "u1", Role: "viewer"}, nil)

    repo.EXPECT().
        Save(gomock.Any(), gomock.AssignableToTypeOf(&User{})).
        DoAndReturn(func(_ context.Context, u *User) error {
            if u.Role != "admin" {
                t.Errorf("Save received role %q; want admin", u.Role)
            }
            return nil
        })

    svc := NewUserService(repo)
    if err := svc.Promote(context.Background(), "u1"); err != nil {
        t.Fatalf("Promote: %v", err)
    }
}
```

`gomock.NewController(t)` registers a test-finalizer that fails the test if expectations aren't met. The `EXPECT()` builder supports argument matchers (`gomock.Any`, `gomock.Eq`, `gomock.AssignableToTypeOf`), call counts (`.Times(1)`, `.AnyTimes()`), and ordering (`gomock.InOrder`).

When it's worth the tooling cost:

- The interface has ten or more methods.
- Multiple packages mock the same interface — one generated mock serves all.
- Strict expectation ordering is essential to the test (e.g., a state machine).

When it isn't:

- Three-method `UserRepo`-style interfaces — the hand-rolled fake is shorter than the `EXPECT()` chain.
- Tests where only return values matter — gomock's verify-every-call overhead becomes friction, not signal.

## 5. `testify/mock`

`github.com/stretchr/testify/mock` is the alternative — runtime-reflective, no code generation. Same idea as Mockito's `mock(Foo.class)`.

```go
import "github.com/stretchr/testify/mock"

type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) FindByID(ctx context.Context, id string) (*User, error) {
    args := m.Called(ctx, id)
    if u := args.Get(0); u != nil {
        return u.(*User), args.Error(1)
    }
    return nil, args.Error(1)
}

func (m *MockUserRepo) Save(ctx context.Context, u *User) error {
    return m.Called(ctx, u).Error(0)
}
```

Test:

```go
func TestPromote_testify(t *testing.T) {
    repo := &MockUserRepo{}
    repo.On("FindByID", mock.Anything, "u1").
        Return(&User{ID: "u1", Role: "viewer"}, nil)
    repo.On("Save", mock.Anything, mock.AnythingOfType("*User")).
        Return(nil)

    svc := NewUserService(repo)
    err := svc.Promote(context.Background(), "u1")

    require.NoError(t, err)
    repo.AssertExpectations(t)
}
```

Tradeoffs vs gomock:

| Aspect | gomock | testify/mock |
|---|---|---|
| Approach | code generation | runtime reflection |
| Setup | `mockgen` + `go:generate` | manual struct embedding |
| Type safety | full — compiler checks | partial — `.Get(0).(*User)` cast |
| Argument matching | rich (`Any`, `Eq`, custom) | rich (`Anything`, `MatchedBy`) |
| Expectation ordering | first-class (`InOrder`) | manual sequencing |
| Failure messages | precise call comparison | useful but less structured |
| Project preference | net-new Go projects | teams already on testify |

Both are fine. The Go community is split roughly down the middle. The decisive factor is usually whether the team already uses `testify/assert` and `testify/require` for assertions — if so, `testify/mock` saves a dependency.

## 6. `go-sqlmock` for `database/sql`

`github.com/DATA-DOG/go-sqlmock` plugs into Go's `database/sql` driver registry, so your code uses `*sql.DB` exactly as in production while the driver is replaced by a mock that matches queries against pre-registered expectations.

```go
import "github.com/DATA-DOG/go-sqlmock"

func TestUserRepo_FindByID(t *testing.T) {
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatalf("sqlmock.New: %v", err)
    }
    t.Cleanup(func() { db.Close() })

    rows := sqlmock.NewRows([]string{"id", "email", "role"}).
        AddRow("u1", "alice@example.com", "viewer")

    mock.ExpectQuery(`SELECT id, email, role FROM users WHERE id = \$1`).
        WithArgs("u1").
        WillReturnRows(rows)

    repo := NewUserRepo(db)
    got, err := repo.FindByID(context.Background(), "u1")
    if err != nil {
        t.Fatalf("FindByID: %v", err)
    }
    if got.Email != "alice@example.com" {
        t.Errorf("email = %q", got.Email)
    }

    if err := mock.ExpectationsWereMet(); err != nil {
        t.Errorf("expectations: %v", err)
    }
}
```

The default query matcher is regex; query strings are escaped (`\$1` for Postgres placeholders). `ExpectExec` is the equivalent for INSERT/UPDATE/DELETE; `ExpectBegin`/`ExpectCommit`/`ExpectRollback` cover transactions.

**When sqlmock is the right tool:** narrow unit tests of repository methods where you want to assert the exact SQL shape without a database. **When it isn't:** integration tests of multi-statement business logic, anything that exercises real constraints, triggers, or query plans. For those use `testcontainers-go` against a real PostgreSQL — that doc lives in [03-integration-and-contract-testing.md](03-integration-and-contract-testing.md).

A common middle path: sqlmock for CRUD-shape tests (fast, no Docker), testcontainers for the harder query and migration tests (slow, real semantics). Both can coexist with `//go:build integration` separating the slow set.

## 7. Time, randomness, network — injecting non-determinism

Every flaky test in Go can usually be traced to one of three things: `time.Now()`, `rand.Int*`, or actual network calls. The fix is the same in each case: depend on an interface, inject a deterministic implementation in tests.

### Time

```go
type Clock interface {
    Now() time.Time
}

type realClock struct{}
func (realClock) Now() time.Time { return time.Now() }

type fakeClock struct{ now time.Time }
func (f *fakeClock) Now() time.Time { return f.now }
func (f *fakeClock) Advance(d time.Duration) { f.now = f.now.Add(d) }

type SubscriptionService struct {
    clock Clock
    // ...
}

func (s *SubscriptionService) IsExpired(sub Subscription) bool {
    return s.clock.Now().After(sub.ExpiresAt)
}
```

In tests:

```go
clk := &fakeClock{now: time.Date(2026, 5, 3, 10, 0, 0, 0, time.UTC)}
svc := &SubscriptionService{clock: clk}
sub := Subscription{ExpiresAt: clk.now.Add(time.Hour)}

if svc.IsExpired(sub) {
    t.Error("not expired yet")
}
clk.Advance(2 * time.Hour)
if !svc.IsExpired(sub) {
    t.Error("should be expired")
}
```

### Randomness

`math/rand/v2` (Go 1.22+) takes a `*rand.Rand` you can seed deterministically:

```go
type IDGenerator struct {
    rng *rand.Rand
}

// In production: rand.New(rand.NewPCG(uint64(time.Now().UnixNano()), 0))
// In tests:      rand.New(rand.NewPCG(42, 0))
```

Or pass an `IDProvider` interface and stub it.

### Network

Use `httptest.NewServer` (§3) so you have a real loopback server, or accept an `*http.Client` parameter and stub its `Transport`. Avoid mocking package-level `http.DefaultClient` — it's global, leaks between tests, and can't be parallelized.

The pattern across all three: the production constructor takes the dependency as a parameter; tests pass a fake. No globals, no monkey patching.

## 8. Avoiding over-mocking

Mocking is a tool; like any tool, it's possible to use it past the point of return. Symptoms of over-mocking:

**Mocking your own code.** If `service_test.go` mocks `service.calculate` to test `service.process`, the test verifies the mock, not the code. Inline the dependency or restructure so the test exercises real behavior.

**Mocking what you don't own.** Mocking the PostgreSQL driver, the AWS SDK, or `gorilla/mux` directly couples your tests to the library's internals. When the library changes, your mocks silently diverge from reality. The fix: define an interface you control (`UserRepo`, `S3Bucket`, `Router`), and mock that.

**Mock-heavy test setup.** When a test has 40 lines of `EXPECT().X().Return(...)` before any actual code runs, the test is documenting the implementation, not the behavior. Replace with a fake that holds state, or rewrite the system under test to have fewer collaborators.

**No real test of integration.** If every test mocks every collaborator, nothing tests the wiring. Pair unit tests (heavy mocking) with at least a few integration tests where real components meet (no mocking of internals; testcontainers for external systems). The "test pyramid" applies here.

A useful heuristic: **mock at architectural seams, not arbitrary boundaries.** Your domain calling a database is a seam. Your domain calling its own helper isn't.

## 9. Comparison with Vitest mocks and Mockito

| Concept | Vitest | Mockito (Java) | Go |
|---|---|---|---|
| Replace a single function | `vi.fn()` | `mock(Foo.class)` | hand-rolled struct or generated mock |
| Stub return value | `.mockReturnValue(x)` | `when(m.f()).thenReturn(x)` | assign to a field on the fake; or `EXPECT().f().Return(x)` |
| Verify call | `expect(fn).toHaveBeenCalledWith(...)` | `verify(m).f(eq(...))` | counter field; or `EXPECT().f(...).Times(1)` |
| Replace whole module | `vi.mock('./repo')` | `@MockBean` (Spring) | inject an interface (no module-level mocking idiomatic) |
| Spy on real | `vi.spyOn(obj, 'method')` | `Mockito.spy(real)` | wrap the real type in a struct that records and forwards |
| Time control | `vi.useFakeTimers()` | mocked clock or PowerMock | inject a `Clock` interface |
| HTTP stubbing | MSW or `vi.fn` on `fetch` | WireMock or RestAssured | `httptest.NewServer` |
| Frameworky setup | `vi.mock` is hoisted | `@ExtendWith(MockitoExtension.class)` | constructor injection — no extension needed |

Three observations from sitting between TS and Java backgrounds and writing Go:

**Module-level mocking has no Go idiom.** `vi.mock('../lib/prisma')` rewrites the import graph; Java's `@MockBean` swaps the bean in the application context. Go has no application context and no module substitution at the test level. The equivalent is structural: pass dependencies through constructors, mock the interface in the constructor's parameter. If you find yourself wishing for `vi.mock` in Go, the production code has a hidden coupling — refactor for explicit dependencies.

**The mock library matters less.** Switching from Mockito to JMockit is a project. Switching from `gomock` to `testify/mock` to hand-rolled fakes is a per-test choice. Pick whatever's clearest for the case at hand and don't over-invest in one.

**`httptest` is the unsung hero.** The closest thing in TS-land is MSW, which is a third-party library with its own setup. In Go it's standard library, two lines, no config. Reach for it whenever the test crosses an HTTP boundary.

## Related

- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — why mocking is mostly free in Go
- [Errors as Values](../fundamentals/06-errors-as-values.md) — error stubbing in fakes
- [Testing Package Fundamentals](01-testing-package-fundamentals.md) — table-driven tests using these doubles
- [Integration & Contract Testing](03-integration-and-contract-testing.md) — when to skip mocking entirely
- [Goroutines and Scheduler](../concurrency/01-goroutines-and-scheduler.md) — testing concurrent code with `-race`
- [`sync` Package and Memory Model](../concurrency/03-sync-package-and-memory-model.md) — race detector context
- [Mocking & Test Doubles in Vitest](../../typescript/testing/mocking-and-test-doubles.md) — the TS counterpart for cross-reference
- [Vitest Fundamentals](../../typescript/testing/vitest-fundamentals.md) — TS mocking baseline

## References

- `net/http/httptest` — https://pkg.go.dev/net/http/httptest
- `uber-go/mock` (the maintained gomock fork) — https://github.com/uber-go/mock
- Original `golang/mock` (archived 2023-06-27) — https://github.com/golang/mock
- `mockgen` usage — https://pkg.go.dev/go.uber.org/mock/mockgen
- `stretchr/testify` — https://github.com/stretchr/testify
- `DATA-DOG/go-sqlmock` — https://github.com/DATA-DOG/go-sqlmock
- "Test Doubles" taxonomy (Meszaros) — http://xunitpatterns.com/Test%20Double.html
- Go FAQ, "Why does Go not have…" — https://go.dev/doc/faq
