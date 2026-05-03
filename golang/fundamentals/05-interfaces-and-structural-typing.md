---
title: "Interfaces & Structural Typing"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, interfaces, polymorphism, structural-typing, typed-nil]
---

# Interfaces & Structural Typing

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `interfaces` `polymorphism` `structural-typing` `typed-nil`

---

## Table of Contents

1. [What an interface is and how it's satisfied](#1-what-an-interface-is-and-how-its-satisfied)
2. [Comparing nominal, structural, and Go's implicit-structural model](#2-comparing-nominal-structural-and-gos-implicit-structural-model)
3. [Interface values: the (type, value) pair](#3-interface-values-the-type-value-pair)
4. [Type assertions and type switches](#4-type-assertions-and-type-switches)
5. [`any`, the empty interface, and when to use it](#5-any-the-empty-interface-and-when-to-use-it)
6. [Accept interfaces, return structs](#6-accept-interfaces-return-structs)
7. [The typed-nil-interface pitfall](#7-the-typed-nil-interface-pitfall)
8. [Small interfaces: the standard library lesson](#8-small-interfaces-the-standard-library-lesson)

## Summary

Go's interfaces are the polymorphism mechanism that replaces inheritance. They are **structural** (a type satisfies an interface by having the right methods) and **implicit** (no `implements` keyword — the satisfaction relation is inferred by the compiler at the call site). This is the inverse of Java, where interfaces are nominal and explicit, and the same shape as TypeScript's structural types — but Go applies it only to interfaces, not to all types.

The model has two important consequences. First, interfaces are usually defined where they are **used**, not where the implementing type lives — flipping the dependency direction Java programmers expect. Second, interface values carry both a type tag and a value, which produces the infamous "typed nil" trap.

## 1. What an interface is and how it's satisfied

An interface is a set of method signatures:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

A type satisfies the interface if it has methods with matching signatures. **There is no `implements` keyword.** The satisfaction is checked by the compiler whenever a concrete value is assigned to an interface variable:

```go
type FileReader struct { /* ... */ }
func (f *FileReader) Read(p []byte) (n int, err error) { /* ... */ }

var r Reader = &FileReader{}  // *FileReader satisfies Reader; the compiler verifies
```

If `*FileReader` did not have `Read` with that exact signature, the assignment would be a compile error. The error would be at the `var r Reader = ...` line, not at the type definition — there is nothing to check at the type definition because there's no claim being made there.

You can check satisfaction explicitly with a compile-time assertion:

```go
var _ Reader = (*FileReader)(nil)  // compile fails if *FileReader doesn't satisfy Reader
```

This idiom is common in libraries that want to fail at compile time instead of when a consumer tries to use the type.

## 2. Comparing nominal, structural, and Go's implicit-structural model

| | TypeScript | Java | Go |
|---|---|---|---|
| Type identity | structural (everywhere) | nominal | nominal for non-interface types, structural for interfaces |
| Interface satisfaction | structural, implicit | nominal, explicit (`implements`) | structural, implicit |
| Multiple interfaces per type | yes | yes (`implements A, B, C`) | yes (just have the methods) |
| Self-documenting `implements`? | no built-in | yes — `implements` keyword | no — but compile-time assertion idiom (§1) is common |

Java requires the implementing class to declare its intent up front:

```java
public class FileReader implements Reader {
    public int read(byte[] p) { /* ... */ }
}
```

Go does not. The compiler walks the methods and checks structurally:

```go
type FileReader struct { /* ... */ }
func (f *FileReader) Read(p []byte) (int, error) { /* ... */ }
// No 'implements' anywhere. *FileReader satisfies io.Reader.
```

The same `*FileReader` may satisfy `io.Reader`, `io.ReaderAt`, `io.Closer`, and a domain-specific `LineSource` interface, all without naming any of them.

This decouples the **definition** of an interface from the **definition** of its implementations. In Go, the canonical pattern is: the **consumer** of a behavior defines the interface it needs, and any pre-existing type with the right methods satisfies it. The implementing type does not need to know the consumer exists.

## 3. Interface values: the (type, value) pair

An interface value at runtime is a two-word structure:

```text
struct {
    type *typeDescriptor   // which concrete type is in here?
    value unsafe.Pointer   // pointer to the actual data
}
```

When you assign a concrete value to an interface variable, both halves are filled in:

```go
var r Reader = &FileReader{path: "/tmp/x"}
// r.type  → *FileReader
// r.value → pointer to the FileReader struct
```

This two-word layout is what makes type assertions cheap and what produces the typed-nil trap (§7).

## 4. Type assertions and type switches

To recover the concrete type behind an interface value:

```go
var r Reader = openSomething()

// Single-value form: panics if r doesn't hold a *FileReader
fr := r.(*FileReader)

// Two-value (comma-ok) form: ok is false instead of panicking
fr, ok := r.(*FileReader)
if !ok {
    // r holds something else
}
```

For multi-way dispatch, a **type switch**:

```go
switch v := r.(type) {
case *FileReader:
    fmt.Println("file:", v.path)
case *NetworkReader:
    fmt.Println("network:", v.addr)
case nil:
    fmt.Println("nothing")
default:
    fmt.Println("unknown:", v)
}
```

Type switches and assertions are how you escape an interface back to a specific type. Idiomatic Go uses them sparingly — most code at any given level deals in one consistent abstraction (an `io.Reader`, not "either a `*FileReader` or a `*NetworkReader`"). When you find yourself writing many type switches, the abstraction is probably too leaky.

## 5. `any`, the empty interface, and when to use it

The empty interface, `interface{}`, is satisfied by every type — it has no methods, so any method set satisfies it. Since Go 1.18, the predeclared identifier `any` is an alias for `interface{}`. They are identical; `any` is preferred in new code.

```go
var x any = 42         // ok
var y any = "hello"    // ok
var z any = struct{}{} // ok
```

`any` is Go's last resort. Use it only when you genuinely don't know the type until runtime:

- `encoding/json.Unmarshal` decoding into a `map[string]any` for ad-hoc JSON.
- `fmt.Printf` and friends — variadic `...any` for arbitrary formatting.
- Generic-pre-1.18 collection libraries (no longer the right answer; see Tier 3 generics).

The cost: any time you pull a value out of an `any`, you need a type assertion or type switch. You've moved type checking from compile time to runtime. With generics in Go 1.18+, most of the historical uses of `any` are now better expressed with type parameters.

## 6. Accept interfaces, return structs

A widely-cited Go convention: **functions should accept the smallest interface they need, and return concrete types**.

```go
// GOOD
func Copy(dst io.Writer, src io.Reader) (int64, error) { /* ... */ }

func NewServer(addr string) *Server { /* ... */ }
```

Rationale:
- **Accept interfaces** — callers can pass anything that satisfies the interface, including types they define themselves. The function under-specifies what it needs, which is more flexible.
- **Return structs** — callers get the full method set, can read fields, and the return type is self-documenting. If the function returned an interface, the caller would have to know what concrete type was actually returned to do anything beyond the interface's methods.

The exception to "return structs" is when the return type really is polymorphic — e.g., `error` (an interface), or `io.ReadCloser` from `http.Response.Body`. But these are exceptions; the default is concrete return types.

This pattern also shapes where interfaces are **defined**. In Java, you write the interface in the library and the implementation imports it. In Go, you typically write the interface in the **consumer** package — the place that calls `Read`, not the place that defines `*FileReader`. The implementing type just needs the right methods; it doesn't need to know about your interface.

```go
// In package "report" (the consumer)
package report

type sourceReader interface {
    Read(p []byte) (int, error)
}

func Generate(r sourceReader) (*Report, error) { /* ... */ }
```

Now `report.Generate` can take an `*os.File`, an `*http.Response.Body`, a `*bytes.Buffer`, or a `*FileReader` from `package storage` — none of which know about `sourceReader`.

## 7. The typed-nil-interface pitfall

This is the most-discussed subtle bug in Go.

```go
type MyError struct { /* ... */ }
func (e *MyError) Error() string { return "my error" }

func doWork() error {
    var err *MyError  // nil pointer
    return err        // returns... what?
}

func main() {
    if err := doWork(); err != nil {
        fmt.Println("got error:", err)  // THIS PRINTS
    }
}
```

The output: `got error: <nil>`. The check `err != nil` was true, even though the pointer inside was nil.

The reason is the (type, value) pair from §3. `doWork` returned an interface value with:
- type = `*MyError` (non-nil — there's a real type tag)
- value = nil pointer

An interface value is `nil` **only if both halves are nil**. The interface here has a non-nil type and a nil value, so `err != nil` is true. The downstream `fmt.Println(err)` calls `err.Error()` on the underlying `*MyError`, which crashes if `Error()` dereferences the receiver.

The fix: **return `nil` directly when there is no error**, not a typed-nil pointer.

```go
func doWork() error {
    var err *MyError
    if somethingFailed() {
        err = &MyError{ /* ... */ }
    }
    if err != nil {
        return err
    }
    return nil  // explicit nil return
}
```

The longer fix is to never declare `err *MyError` as the return-bearing variable in the first place — work with the interface type from the top:

```go
func doWork() error {
    if somethingFailed() {
        return &MyError{ /* ... */ }
    }
    return nil
}
```

This bug shows up most often when:
- Using `errors.New` or `fmt.Errorf` for the success path. (No one does — that's the whole point.)
- Wrapping a typed error and forgetting to check for the typed-nil case.
- `interface{}` passed through a generic / dynamic dispatch that forgets the asymmetry.

The Go FAQ and Go vet's `nilness` analyzer both call this out, and `go vet` will warn on the obvious pattern.

## 8. Small interfaces: the standard library lesson

The Go standard library's most-used interfaces are tiny:

| Interface | Methods | Where it's defined |
|---|---|---|
| `error` | `Error() string` | builtin |
| `io.Reader` | `Read(p []byte) (n int, err error)` | `io` |
| `io.Writer` | `Write(p []byte) (n int, err error)` | `io` |
| `io.Closer` | `Close() error` | `io` |
| `fmt.Stringer` | `String() string` | `fmt` |
| `sort.Interface` | `Len() int`, `Less(i, j int) bool`, `Swap(i, j int)` | `sort` |

The largest commonly-used interface in the standard library has three methods. The Go community has internalized this:

> The bigger the interface, the weaker the abstraction.
>
> — Rob Pike, _Go Proverbs_, 2015

A one-method interface composes through interface embedding:

```go
type ReadWriter interface {
    Reader
    Writer
}
```

…and any function that needs only the read half can take a `Reader` and get a `*ReadWriter` for free, because the structural satisfaction relation handles the subset trivially.

The pattern to internalize: define the smallest interface you need at the call site. If you find yourself defining an interface with 8 methods, you're probably re-creating an OOP-style "service interface" — and you'll get more flexibility by splitting it into smaller pieces and embedding them when needed.

## Related

- [Go Tour for TS + Java Developers](01-go-tour-for-ts-java-devs.md) — visibility, package model
- [Types, Zero Values & Composite Literals](03-types-zero-values-composites.md) — struct embedding (the other half of "composition over inheritance")
- [Errors as Values](06-errors-as-values.md) — `error` is just an interface
- [Pointers, Methods & Receivers](07-pointers-methods-receivers.md) — method sets and which types satisfy which interfaces
- [TypeScript Structural Typing & Type Compatibility](../../typescript/type-system/structural-typing.md) — the closest analogue in TS
- [Java Patterns](../../java/) — `implements` and the nominal model

## References

- The Go Programming Language Specification, "Interface types" — https://go.dev/ref/spec#Interface_types
- The Go Programming Language Specification, "Type assertions" — https://go.dev/ref/spec#Type_assertions
- Effective Go, "Interfaces and other types" — https://go.dev/doc/effective_go#interfaces_and_types
- Go FAQ, "Why is my nil error value not equal to nil?" — https://go.dev/doc/faq#nil_error
- Rob Pike, _Go Proverbs_, 2015 — https://go-proverbs.github.io/
- "Codebase refactoring (with help from Go)" — https://go.dev/blog/codelab-share (the small-interfaces lesson, applied)
- Go 1.18 release notes — `any` alias — https://go.dev/doc/go1.18#predeclared
- `go vet` `nilness` analyzer — https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/nilness
