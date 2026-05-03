---
title: "Go Tour for TypeScript + Java Developers"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, syntax, tour, comparison]
---

# Go Tour for TypeScript + Java Developers

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `syntax` `tour` `comparison`

---

## Table of Contents

1. [Program structure & the file as a unit of compilation](#1-program-structure--the-file-as-a-unit-of-compilation)
2. [Packages, imports, and visibility](#2-packages-imports-and-visibility)
3. [Variables and the basic types](#3-variables-and-the-basic-types)
4. [Control flow: one loop, no parens](#4-control-flow-one-loop-no-parens)
5. [Functions: multiple returns, named returns, variadics](#5-functions-multiple-returns-named-returns-variadics)
6. [`defer`, `panic`, and stack-style cleanup](#6-defer-panic-and-stack-style-cleanup)
7. [`gofmt` is law](#7-gofmt-is-law)
8. [What Go intentionally leaves out](#8-what-go-intentionally-leaves-out)

## Summary

Go's syntax is small on purpose. The Go specification is roughly 90 pages; the C# spec is over 500 and the Java Language Specification is over 700. That smallness is the language's central design move: there is usually exactly one obvious way to do a thing, the formatter is mandatory and non-negotiable, and features that other languages spent years debating (ternaries, exceptions, inheritance, parameterless `if` parens) are simply absent.

Coming from TypeScript and Java, the surprises are mostly omissions, not additions: no classes, no exceptions, no generics until 1.18 (and they're deliberately limited), no implicit conversions, no overloading. Once you accept what is missing, the parts that remain compose into a language you can read end-to-end after a weekend.

This doc is the syntactic tour. The mental-model docs that follow it (interfaces, errors, pointers, slices) are where Go starts to feel different.

## 1. Program structure & the file as a unit of compilation

Every Go file declares a package, then imports, then top-level declarations. Order inside the file does not matter — the compiler resolves names within a package, not within a file.

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "usage: hello NAME")
        os.Exit(1)
    }
    fmt.Printf("hello, %s\n", os.Args[1])
}
```

Side-by-side:

```ts
// TypeScript / Node
import { argv, stderr, exit } from "node:process";

if (argv.length < 3) {
  stderr.write("usage: hello NAME\n");
  exit(1);
}
console.log(`hello, ${argv[2]}`);
```

```java
// Java
public class Hello {
    public static void main(String[] args) {
        if (args.length < 1) {
            System.err.println("usage: hello NAME");
            System.exit(1);
        }
        System.out.printf("hello, %s%n", args[0]);
    }
}
```

A few things to notice up front:
- `package main` plus a `func main()` is what makes a binary. Any other package name produces a library.
- Imports are paths, not names. `"fmt"` is the standard library; a third-party import looks like `"github.com/google/uuid"`.
- There is no `public` / `private` keyword. Visibility is encoded in the **case of the identifier**: `Println` is exported, `println` is not.

## 2. Packages, imports, and visibility

A directory is a package. All `.go` files in the directory must share one `package <name>` line. Files in the same package see each other's identifiers without an import.

```go
// file: greet/greet.go
package greet

import "fmt"

func Hello(name string) string {  // exported (uppercase)
    return formatGreeting("hello", name)
}

func formatGreeting(verb, name string) string {  // unexported (lowercase)
    return fmt.Sprintf("%s, %s", verb, name)
}
```

Consumers import the package by **path**, then refer to its exported identifiers by **package name**:

```go
import "example.com/myapp/greet"

func main() {
    fmt.Println(greet.Hello("Alice"))
}
```

Comparison of visibility models:

| Language | How visibility is expressed |
|---|---|
| Java | `public` / `protected` / package-private (default) / `private` keywords |
| TypeScript | `export` keyword on the declaration; everything else is module-local |
| Go | Identifier case: `UpperCase` is exported, `lowerCase` is package-local. No keyword, no `private` |

Unused imports are a **compile error**, not a warning. Same for unused local variables. This is intentional and one of the most-cited "I felt the language pushing back" experiences for new Go programmers — most editors hide the friction by auto-removing unused imports on save via `goimports`.

## 3. Variables and the basic types

Three ways to declare:

```go
// Explicit type
var count int = 0

// Type inferred
var count = 0

// Short declaration (only inside functions)
count := 0
```

The short form `:=` is what you'll write most. It is **only** valid inside a function body — at package scope you must use `var`.

The basic types:

| Category | Types |
|---|---|
| Boolean | `bool` |
| Integers | `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr` |
| Aliases | `byte` (= `uint8`), `rune` (= `int32`, used for Unicode code points) |
| Floats | `float32`, `float64` |
| Complex | `complex64`, `complex128` |
| Strings | `string` (immutable, UTF-8 bytes) |

`int` is platform-sized — 64 bits on every 64-bit target you care about, 32 bits on 32-bit targets. Use the explicit-width types (`int32`, `int64`) when the size matters for serialization or interop.

Two non-obvious rules:
1. **No implicit numeric conversion.** Adding an `int32` to an `int64` does not compile. You must write `int64(x) + y`. This is the inverse of Java's quiet widening conversions.
2. **Strings are immutable byte sequences, not character arrays.** Indexing a string gives you a `byte`, not a character — `"héllo"[1]` is `0xc3`, the first byte of the UTF-8 encoding of `é`. To iterate runes (code points), use `for _, r := range s`.

```go
s := "héllo"
fmt.Println(len(s))      // 6  (bytes, not characters)
fmt.Println(s[1])        // 195 (a byte: 0xc3)
for i, r := range s {
    fmt.Printf("%d: %c\n", i, r)
}
// 0: h
// 1: é
// 3: l
// 4: l
// 5: o
```

## 4. Control flow: one loop, no parens

Go has exactly one looping construct: `for`. It plays four roles.

```go
// Classic three-clause loop
for i := 0; i < 10; i++ { /* ... */ }

// While loop (just drop the init and post)
for x < 100 { /* ... */ }

// Infinite loop
for { /* ... */ }

// Range over a slice / map / channel / string
for i, v := range items { /* ... */ }
```

`if` and `for` do not take parentheses, but braces are mandatory — even for one-line bodies. This kills the "dangling-else" bug and the goto-fail-style single-line-without-braces bug at the syntax level (the latter being the literal cause of CVE-2014-1266 in Apple's SecureTransport).

`if` accepts an init statement, scoped to the `if`/`else`:

```go
if user, err := lookup(id); err != nil {
    return err
} else {
    use(user)
}
```

`switch` cases do not fall through by default — you opt into fallthrough with the keyword `fallthrough`. This is the inverse of C/Java/TS, where you opt out with `break`. There is no need for `break`.

```go
switch status {
case "active", "trial":         // multiple values per case
    return true
case "suspended":
    return false
default:
    return false
}
```

A `switch` with no expression is equivalent to `switch true` — useful as a chain of conditions:

```go
switch {
case x < 0:
    return "negative"
case x == 0:
    return "zero"
default:
    return "positive"
}
```

## 5. Functions: multiple returns, named returns, variadics

Multiple return values are first-class and ubiquitous — they replace exceptions for error reporting (covered in [Errors as Values](06-errors-as-values.md)).

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("divide by zero")
    }
    return a / b, nil
}

// Caller must consume both, or explicitly discard with _
result, err := divide(10, 2)
result, _ := divide(10, 2)  // discard the error (rare; usually a code smell)
```

Named return values double as documentation and let you write a bare `return`:

```go
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return  // returns x, y
}
```

Variadic parameters use `...T`:

```go
func max(nums ...int) int { /* ... */ }
max(1, 2, 3)
nums := []int{1, 2, 3}
max(nums...)  // spread a slice
```

Functions are first-class values and closures capture variables by reference:

```go
adder := func(x int) func(int) int {
    sum := 0
    return func(y int) int {
        sum += x + y
        return sum
    }
}
```

There is **no function overloading**. You cannot define two functions with the same name and different signatures in the same package. This is the same constraint TypeScript has at runtime (since it compiles to JS) but enforced by the compiler instead of by the runtime model.

## 6. `defer`, `panic`, and stack-style cleanup

`defer` schedules a call to run when the surrounding function returns — in LIFO order. It is Go's substitute for `try/finally` and the foundation of resource cleanup.

```go
func readFile(name string) ([]byte, error) {
    f, err := os.Open(name)
    if err != nil {
        return nil, err
    }
    defer f.Close()  // runs no matter how readFile returns
    return io.ReadAll(f)
}
```

Two non-obvious behaviours:
1. Arguments to a deferred call are **evaluated when the `defer` statement executes**, not when it runs:
   ```go
   x := 1
   defer fmt.Println(x)  // prints 1, even though x is mutated below
   x = 2
   ```
2. Deferred calls run **after** the return value is set but **before** the caller resumes. This means a deferred call can mutate a named return value — useful for error wrapping at function boundaries.

`panic` and `recover` exist but are not the analogue of exceptions. They are reserved for unrecoverable program states (nil dereference, index out of range, contract violations). Idiomatic Go does not use them for control flow. The boundary use case is recovering from panics in goroutines spawned by HTTP handlers, queue consumers, etc., to keep the process alive — covered in [Errors as Values](06-errors-as-values.md).

## 7. `gofmt` is law

There is one canonical formatting for Go source. `gofmt` (or its more aggressive cousin `goimports`, which also manages imports) is run on save by every editor in the ecosystem. The Go community spent zero time arguing about tabs vs spaces or where the brace goes — `gofmt` decided in 2009 (tabs, brace on the same line) and the conversation was over.

Practical implications:
- Code review never discusses formatting.
- Diffs are smaller because everyone's code looks the same.
- `gofmt -d` in CI catches non-canonical formatting as a build failure.

If you come from TS, this replaces the Prettier + ESLint + EditorConfig stack. If you come from Java, this replaces the Checkstyle + spotless argument.

## 8. What Go intentionally leaves out

A short list of features Go does not have, and what to use instead:

| Missing feature | Go's answer |
|---|---|
| Classes & inheritance | Structs + embedding for composition; interfaces for polymorphism. See [Interfaces](05-interfaces-and-structural-typing.md). |
| Exceptions | Multiple returns with an `error` value. See [Errors as Values](06-errors-as-values.md). |
| Constructors | The zero value is required to be useful, plus `New<Type>` factory functions by convention. See [Types & Zero Values](03-types-zero-values-composites.md). |
| Function overloading | One name, one signature. Use different names (`Add` / `AddString`) or variadics. |
| Default arguments | Functional options pattern (covered in Tier 5) or builders. |
| Ternary operator | `if`/`else`. The Go FAQ explicitly defends this — Go's authors found ternaries reliably produced unreadable nested code. |
| Generics (pre-1.18) | Generics arrived in Go 1.18 (March 2022). Pre-1.18 code uses `interface{}` and reflection. |
| Implicit type conversion | Explicit cast: `int64(x)`. |
| Macros / annotations | `go generate` for code generation; struct tags for serialization metadata. |

The omission that surprises new Go programmers most is **classes**. Go has structs, methods on structs (covered in [Pointers, Methods & Receivers](07-pointers-methods-receivers.md)), and interfaces. There is no `extends`, no `implements` keyword, and no class hierarchy. Composition through struct embedding plus structural interface satisfaction does the work that inheritance does in Java — and in many cases does it better, because the coupling is shallower.

## Related

- [Module System & Project Layout](02-module-system-and-layout.md) — how Go programs are packaged
- [Types, Zero Values & Composite Literals](03-types-zero-values-composites.md) — the value model
- [Interfaces & Structural Typing](05-interfaces-and-structural-typing.md) — the polymorphism model that replaces classes
- [Errors as Values](06-errors-as-values.md) — what replaces exceptions
- [TypeScript Compiler — tsconfig Demystified](../../typescript/tooling/tsconfig-demystified.md) — TS compiler config (compare with Go's near-zero config surface)
- [Java Coding Standards](../../java/) — reference for the Java side of the contrasts above

## References

- The Go Programming Language Specification — https://go.dev/ref/spec
- Effective Go — https://go.dev/doc/effective_go
- Go FAQ ("Why does Go not have…?") — https://go.dev/doc/faq
- A Tour of Go — https://go.dev/tour/
- "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite." — Rob Pike, _Go Proverbs_, 2015. https://go-proverbs.github.io/
- Donovan & Kernighan, _The Go Programming Language_, Addison-Wesley, 2015 — Chapter 1 ("Tutorial") covers the same ground in book length.
