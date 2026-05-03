---
title: "The `testing` Package — Fundamentals"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, testing, table-driven, subtests, benchmarks, fuzzing]
---

# The `testing` Package — Fundamentals

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `testing` `table-driven` `subtests` `benchmarks` `fuzzing`

---

## Table of Contents

1. [Philosophy: tests are just Go code](#1-philosophy-tests-are-just-go-code)
2. [Test discovery and the `go test` command](#2-test-discovery-and-the-go-test-command)
3. [Table-driven tests — the canonical idiom](#3-table-driven-tests--the-canonical-idiom)
4. [Subtests with `t.Run` and `t.Parallel`](#4-subtests-with-trun-and-tparallel)
5. [Helpers, cleanup, temp dirs](#5-helpers-cleanup-temp-dirs)
6. [Benchmarks](#6-benchmarks)
7. [Example tests (godoc-runnable)](#7-example-tests-godoc-runnable)
8. [Fuzz testing](#8-fuzz-testing)
9. [Coverage](#9-coverage)
10. [Comparison with Vitest and JUnit 5](#10-comparison-with-vitest-and-junit-5)

## Summary

Go ships its test framework in the standard library. There is no `package.json` `devDependencies` ritual, no `pom.xml` plugin, no separate runner binary — `go test` is part of the toolchain, and any file ending in `_test.go` becomes a test. The `testing` package supplies three top-level types: `*testing.T` (unit tests), `*testing.B` (benchmarks), and `*testing.F` (fuzz tests). Discovery is by naming convention: `TestXxx`, `BenchmarkXxx`, `FuzzXxx`, `ExampleXxx`. There are no annotations, no decorators, no descriptors.

This minimalism shapes the idioms. Without a parameterized-test annotation, Go developers reach for **table-driven tests**: a slice of structs, a `for` loop, and `t.Run` for subtests. Without a heavyweight assertion library, you write `if got != want { t.Errorf(...) }` directly. Without lifecycle annotations, you use `t.Cleanup` and `t.TempDir` inside the test body. The result is tests that read like Go code rather than DSL incantations — and that's the point.

## 1. Philosophy: tests are just Go code

The Go FAQ puts it bluntly: tests live alongside production code in the same package, written in the same language, with no special framework. A test file is any file whose name ends in `_test.go`; the Go toolchain compiles those files **only** when running `go test`, and excludes them from the production binary.

Two placement options:

- **Same package** (`foo_test.go` in package `foo`) — tests can reach unexported identifiers. Use this for unit tests of internal helpers.
- **External test package** (`foo_test.go` declaring `package foo_test`) — tests see only the exported API. Forces you to consume the package as a user would, which catches API-design mistakes.

A typical layout:

```text
internal/billing/
├── invoice.go
├── invoice_test.go        // package billing — internal test
├── invoice_export_test.go // package billing_test — black-box test
└── testdata/
    └── invoice-fixture.json
```

The `testdata/` directory is treated specially: the Go toolchain ignores it when compiling, so you can put any fixture content there without worrying about it being parsed as Go.

A minimal test:

```go
package billing

import "testing"

func TestRoundCents(t *testing.T) {
    got := RoundCents(1.235)
    want := 1.24
    if got != want {
        t.Errorf("RoundCents(1.235) = %v; want %v", got, want)
    }
}
```

That's it. No `describe`, no `@Test`, no imports beyond `testing`.

## 2. Test discovery and the `go test` command

`go test` walks the import graph, compiles the package plus its `_test.go` files into a test binary, and runs it. The discovery rule is simple: any function with the signature `func TestXxx(t *testing.T)` (where `Xxx` starts with an uppercase letter) is a test.

Common invocations:

```bash
# Run tests in the current package
go test

# Run tests in this package and all subpackages (the most common form)
go test ./...

# Verbose — print PASS/FAIL for every test, plus t.Log output
go test -v ./...

# Run a single test by name (regex matched against TestXxx)
go test -run TestRoundCents ./internal/billing

# Run a subtest within a test (slash-separated)
go test -run TestRoundCents/negative_amount ./internal/billing

# Force re-run, bypassing the build cache
go test -count=1 ./...

# Race detector — costs 5-10x slowdown but catches data races
go test -race ./...

# Short mode — tests opt out of long work via testing.Short()
go test -short ./...

# Fail any test that takes longer than 30 seconds
go test -timeout 30s ./...
```

`go test` caches successful test results keyed by the inputs (source files, environment, and command line). A second invocation with no changes prints `(cached)` and exits instantly. To bypass: `-count=1`, or modify the package, or run `go clean -testcache`.

`testing.Short()` returns `true` when `-short` is passed. Use it to skip slow tests in CI's fast-feedback lane:

```go
func TestImportsLargeDataset(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping in -short mode")
    }
    // expensive setup...
}
```

## 3. Table-driven tests — the canonical idiom

When you'd reach for `@ParameterizedTest` in JUnit or `it.each` in Vitest, in Go you write a table. A slice of struct literals, each describing one case, iterated in a `for` loop.

```go
func TestRoundCents(t *testing.T) {
    tests := []struct {
        name  string
        input float64
        want  float64
    }{
        {"round half up", 1.235, 1.24},
        {"round half down", 1.234, 1.23},
        {"already two decimals", 1.20, 1.20},
        {"negative", -0.005, -0.01},
        {"zero", 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := RoundCents(tt.input); got != tt.want {
                t.Errorf("RoundCents(%v) = %v; want %v", tt.input, got, tt.want)
            }
        })
    }
}
```

Why this form dominates idiomatic Go testing:

- **One source of truth** for the cases — adding a row is one line.
- **Each row becomes a subtest** via `t.Run(tt.name, ...)`. The output reads `TestRoundCents/round_half_up`, and you can target a single case with `-run TestRoundCents/zero`.
- **No DSL** to learn. It's a `for` loop, a struct, and the standard library.
- **The compiler checks every field** — if you change the struct, every row gets a compile error until updated.

A useful refinement: include both a "happy path" group and an "error" group, and check `err`:

```go
tests := []struct {
    name    string
    input   string
    want    Invoice
    wantErr bool
}{
    {"valid", `{"amount": 100}`, Invoice{Amount: 100}, false},
    {"missing field", `{}`, Invoice{}, true},
    {"malformed json", `{`, Invoice{}, true},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Parse(tt.input)
        if (err != nil) != tt.wantErr {
            t.Fatalf("Parse() err = %v; wantErr = %v", err, tt.wantErr)
        }
        if !tt.wantErr && got != tt.want {
            t.Errorf("Parse() = %+v; want %+v", got, tt.want)
        }
    })
}
```

For error-shape assertions beyond a boolean, use `errors.Is` and `errors.As` (see [Errors as Values §2–3](../fundamentals/06-errors-as-values.md)).

## 4. Subtests with `t.Run` and `t.Parallel`

`t.Run(name, fn)` creates a subtest. The subtest gets its own `*testing.T`, its own pass/fail status, and a fully-qualified name (`Parent/Child`). Failures in one subtest don't stop sibling subtests.

`t.Parallel()` signals "this test (or subtest) can run concurrently with other parallel tests." The runner pauses the calling test until all non-parallel tests in the same package finish, then runs all parallel ones together.

```go
func TestParseAddress(t *testing.T) {
    t.Parallel()  // this whole test can run in parallel with siblings

    cases := []struct {
        name, input string
    }{
        {"with country", "1 Main St, NY, USA"},
        {"without country", "1 Main St, NY"},
        {"po box", "PO Box 123"},
    }

    for _, tc := range cases {
        tc := tc  // capture for parallel — pre-Go 1.22 only
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()  // each subtest also runs in parallel
            _, err := ParseAddress(tc.input)
            if err != nil {
                t.Errorf("ParseAddress(%q) err = %v", tc.input, err)
            }
        })
    }
}
```

**The loop-variable gotcha (pre-Go 1.22):** prior to Go 1.22 the loop variable `tc` was reused across iterations, so `t.Parallel()` subtests captured the same final value. The classic fix was `tc := tc` to create a fresh per-iteration binding. Go 1.22 changed loop semantics — each iteration now gets its own variable — and the `tc := tc` shadow is no longer needed. If you target Go 1.22+ and your `go.mod` declares it, drop the shadow.

The `-parallel n` flag bounds parallelism (default: `GOMAXPROCS`).

## 5. Helpers, cleanup, temp dirs

### `t.Helper()`

Marks the calling function as a test helper so that failure messages report the caller's file and line, not the helper's:

```go
func mustParse(t *testing.T, raw string) Invoice {
    t.Helper()
    inv, err := Parse(raw)
    if err != nil {
        t.Fatalf("Parse(%q): %v", raw, err)
    }
    return inv
}

func TestInvoice(t *testing.T) {
    inv := mustParse(t, `{"amount": 100}`)  // failure points here, not inside mustParse
    // ...
}
```

Without `t.Helper()`, `t.Fatalf` would point at the `t.Fatalf` line inside `mustParse` — useless for debugging.

### `t.Cleanup(fn)`

Registers a function to run when the test (and its subtests) finishes, in LIFO order. Replaces the older pattern of `defer` plus exported helper functions:

```go
func TestUserRepo(t *testing.T) {
    db := openTestDB(t)
    t.Cleanup(func() { db.Close() })

    // The cleanup runs even if subsequent code panics or t.Fatal fires.
}
```

`t.Cleanup` is preferable to `defer` inside the test for shared setup helpers — the helper itself can register cleanup:

```go
func newTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("postgres", testDSN)
    if err != nil {
        t.Fatalf("open: %v", err)
    }
    t.Cleanup(func() { db.Close() })
    return db
}
```

### `t.TempDir()`

Creates a fresh temporary directory and registers cleanup that removes it. Each call returns a new directory; nothing leaks between tests.

```go
func TestWriteConfig(t *testing.T) {
    dir := t.TempDir()
    path := filepath.Join(dir, "config.toml")
    if err := WriteConfig(path, defaultConfig); err != nil {
        t.Fatalf("WriteConfig: %v", err)
    }
    // No teardown needed — t.TempDir handles it.
}
```

### `t.Setenv(key, val)`

Sets an environment variable for the duration of the test, then restores it. Implicitly disables `t.Parallel` (env vars are global).

## 6. Benchmarks

Functions of the shape `func BenchmarkXxx(b *testing.B)` are benchmarks. The runner calls them with `b.N` set to a small number and increases it until the per-iteration timing is statistically stable.

```go
func BenchmarkRoundCents(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = RoundCents(1.2345)
    }
}
```

Run with:

```bash
go test -bench=. -benchmem ./internal/billing
```

Output:

```text
BenchmarkRoundCents-10    523817286    2.291 ns/op    0 B/op    0 allocs/op
```

Columns: name, iterations completed, ns per op, bytes allocated per op (with `-benchmem`), allocations per op.

`b.ResetTimer()` discards setup time — call it after expensive setup so it doesn't pollute the measurement:

```go
func BenchmarkParseLargeJSON(b *testing.B) {
    data, err := os.ReadFile("testdata/large.json")
    if err != nil {
        b.Fatal(err)
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = Parse(data)
    }
}
```

`b.ReportAllocs()` forces allocation reporting even without `-benchmem`. `b.Loop()` (Go 1.24+) is a newer alternative to the `for i := 0; i < b.N; i++` form that handles timer reset and dead-store elimination automatically.

Sub-benchmarks via `b.Run` work like `t.Run`:

```go
func BenchmarkParse(b *testing.B) {
    sizes := []int{100, 1_000, 10_000}
    for _, n := range sizes {
        b.Run(fmt.Sprintf("size=%d", n), func(b *testing.B) {
            data := makePayload(n)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                _, _ = Parse(data)
            }
        })
    }
}
```

For statistical comparison between two implementations, capture results to files and compare with `benchstat`:

```bash
go test -bench=. -count=10 -run=^$ ./... > old.txt
# make changes
go test -bench=. -count=10 -run=^$ ./... > new.txt
benchstat old.txt new.txt
```

## 7. Example tests (godoc-runnable)

Functions named `ExampleXxx` are compiled, executed, and have their stdout captured and compared against an `// Output:` comment. They serve as both documentation (rendered on pkg.go.dev with a "Run" button) and tests.

```go
func ExampleRoundCents() {
    fmt.Println(RoundCents(1.235))
    // Output: 1.24
}

func ExampleInvoice_Total() {
    inv := Invoice{Items: []Item{{Price: 100}, {Price: 50}}}
    fmt.Println(inv.Total())
    // Output: 150
}
```

Naming associates the example with a documented identifier:

| Function name | Documents |
|---|---|
| `ExamplePackage` | the package itself |
| `ExampleFoo` | function `Foo` |
| `ExampleBar_Qux` | method `Qux` on type `Bar` |
| `ExampleFoo_negativeInput` | suffixed variant for multiple examples |

`// Unordered output:` ignores line ordering — useful for map iteration. Examples without an `Output:` comment compile but don't execute, useful for examples that touch the network.

## 8. Fuzz testing

Fuzzing was added in Go 1.18 (the proposal is at golang/go#44551). A fuzz test is a function `func FuzzXxx(f *testing.F)` that registers seed inputs with `f.Add` and a target function with `f.Fuzz`. The runner generates inputs that mutate the seeds, runs the target against each, and saves any input that triggers a failure.

```go
func FuzzParseAddress(f *testing.F) {
    f.Add("1 Main St, NY, USA")
    f.Add("PO Box 123")
    f.Add("")  // edge case

    f.Fuzz(func(t *testing.T, in string) {
        addr, err := ParseAddress(in)
        if err != nil {
            return  // a parse error is fine; we want crashes
        }
        // Round-trip property: formatting then re-parsing should be idempotent.
        again, err := ParseAddress(addr.String())
        if err != nil {
            t.Errorf("re-parse failed: %v (original=%q)", err, in)
        }
        if again != addr {
            t.Errorf("round-trip mismatch: %v != %v", again, addr)
        }
    })
}
```

Modes:

```bash
# Default: run the fuzz target only on the seed corpus (acts as a unit test).
go test ./...

# Active fuzzing: generate inputs continuously until a failure or Ctrl+C.
go test -fuzz=FuzzParseAddress -fuzztime=30s ./...
```

When fuzzing finds a failing input it writes the bytes to `testdata/fuzz/FuzzParseAddress/<hash>` and prints a `go test -run=FuzzParseAddress/<hash>` reproducer. The failing input becomes part of the regression corpus on subsequent commits — anyone running `go test` will hit it until it's fixed.

The Go security team's fuzzing landing page (https://go.dev/security/fuzz/) explains the mutation engine and corpus management. Fuzz testing is most valuable on parsers, decoders, and any function consuming untrusted bytes.

**Caveat from the Go 1.18 release notes:** the fuzz cache (`$GOCACHE/fuzz/`) is unbounded — long fuzzing runs can grow to multi-GB. Run `go clean -fuzzcache` to reclaim space.

## 9. Coverage

Coverage instrumentation is built in:

```bash
# Print summary coverage for the package
go test -cover ./...

# Write a profile and inspect it
go test -coverprofile=cover.out ./...
go tool cover -func=cover.out                 # per-function table
go tool cover -html=cover.out                 # opens an HTML viewer
go tool cover -html=cover.out -o coverage.html

# Combine multiple packages in one report
go test -coverprofile=cover.out -coverpkg=./... ./...
```

The `-coverpkg=./...` flag is important: by default `-coverprofile` only counts code in the package being tested. `-coverpkg=./...` measures every package the test exercises, which is what most teams want.

The HTML view colors each line green (covered), red (not covered), or grey (not instrumented). For CI, parse `go tool cover -func=cover.out`'s last line to get the total percentage, or convert the profile to lcov for tools like Codecov.

There's no built-in threshold-fail like Vitest's `coverage.thresholds`. Either parse the percentage in CI yourself, or use a third-party wrapper.

## 10. Comparison with Vitest and JUnit 5

| Concern | Vitest | JUnit 5 | Go `testing` |
|---|---|---|---|
| Where it lives | `devDependencies` | Maven/Gradle plugin | standard library, in toolchain |
| Test discovery | `**/*.test.ts` glob | `@Test` annotation | `TestXxx` in `*_test.go` |
| Grouping | `describe` blocks | `@Nested` classes | `t.Run` subtests |
| Parameterization | `it.each([...])` | `@ParameterizedTest` + sources | table-driven loop |
| Setup/teardown | `beforeEach`/`afterEach` | `@BeforeEach`/`@AfterEach` | `t.Cleanup` registered in test body |
| Parallelism | per file (workers) | configurable | `t.Parallel` per test/subtest |
| Assertions | `expect().toBe/toEqual` | `assertEquals`, AssertJ | plain `if got != want` + `t.Errorf` |
| Mocks | `vi.fn`, `vi.mock` | Mockito | hand-rolled fakes / gomock / testify |
| Benchmarks | not built in | JMH (separate) | `BenchmarkXxx` first-class |
| Fuzzing | not built in | Jazzer (separate) | `FuzzXxx` first-class (1.18+) |
| Examples in docs | doc comments | not standard | `ExampleXxx` runs and is shown on godoc |
| Coverage | `@vitest/coverage-v8` | JaCoCo | `-cover` / `go tool cover` |
| Coverage thresholds | yes, in config | yes, in plugin | not built in |
| Race detector | n/a | n/a | `-race` flag (built in) |

Three things stand out for someone coming from Vitest or JUnit:

**No assertion library by default.** The Go core team has consistently declined to add one. The argument: a failing `if got != want` already tells you what you need (`got = X; want = Y`), and chained-fluent matchers add a vocabulary to learn without making the test more correct. Many teams adopt `testify/assert` anyway — see [02-mocking-and-test-doubles.md](02-mocking-and-test-doubles.md).

**Setup is data, not annotations.** Where JUnit uses `@BeforeEach` and Vitest uses `beforeEach`, idiomatic Go uses helpers called from the test body, with `t.Cleanup` registered inline. The benefit is locality: you can read a single test top-to-bottom and see exactly what it sets up. The cost is a little more typing.

**Race detector and benchmarks are built-in superpowers.** `go test -race` finds data races at runtime — there is no equivalent shipping with Vitest or JUnit. Benchmarks live alongside tests and use the same toolchain. These integrate concurrency testing (see [Goroutines and Scheduler](../concurrency/01-goroutines-and-scheduler.md) and the [sync package memory model](../concurrency/03-sync-package-and-memory-model.md)) into the everyday test flow.

## Related

- [Errors as Values](../fundamentals/06-errors-as-values.md) — error assertion patterns in tests
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — why interfaces enable mocking
- [Mocking & Test Doubles in Go](02-mocking-and-test-doubles.md) — the next doc in this tier
- [Integration & Contract Testing](03-integration-and-contract-testing.md) — testcontainers, golden files, contracts
- [Goroutines and Scheduler](../concurrency/01-goroutines-and-scheduler.md) — context for the `-race` flag
- [`sync` Package and Memory Model](../concurrency/03-sync-package-and-memory-model.md) — race detector grounding
- [Vitest Fundamentals](../../typescript/testing/vitest-fundamentals.md) — the TS counterpart
- [Mocking & Test Doubles in Vitest](../../typescript/testing/mocking-and-test-doubles.md) — TS doubles, for cross-reference

## References

- `testing` package — https://pkg.go.dev/testing
- `cmd/go` — Testing flags — https://pkg.go.dev/cmd/go#hdr-Testing_flags
- `cmd/cover` — https://pkg.go.dev/cmd/cover
- "Testable Examples in Go" (Go blog) — https://go.dev/blog/examples
- "Go Fuzzing" (security landing page) — https://go.dev/security/fuzz/
- Go 1.18 release notes (fuzzing GA) — https://go.dev/doc/go1.18
- Fuzzing proposal — https://github.com/golang/go/issues/44551
- Go 1.22 loop variable scoping change — https://go.dev/blog/loopvar-preview
- `benchstat` — https://pkg.go.dev/golang.org/x/perf/cmd/benchstat
