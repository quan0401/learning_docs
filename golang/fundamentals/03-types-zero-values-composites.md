---
title: "Types, Zero Values & Composite Literals"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, types, zero-values, structs, embedding, iota]
---

# Types, Zero Values & Composite Literals

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `types` `zero-values` `structs` `embedding` `iota`

---

## Table of Contents

1. [Value types vs reference types](#1-value-types-vs-reference-types)
2. [The zero-value rule and why Go has no constructors](#2-the-zero-value-rule-and-why-go-has-no-constructors)
3. [Structs and composite literals](#3-structs-and-composite-literals)
4. [Struct embedding: composition without inheritance](#4-struct-embedding-composition-without-inheritance)
5. [Type definitions vs type aliases](#5-type-definitions-vs-type-aliases)
6. [`iota` and the enum idiom](#6-iota-and-the-enum-idiom)
7. [Struct tags](#7-struct-tags)

## Summary

Go's type system is small and copy-by-default. Every assignment is a copy unless the type is a built-in reference type (slices, maps, channels, function values, interface values, pointers). There is no constructor — types are required to be **useful at their zero value**, and factory functions named `New<Type>` are used only when zero is genuinely insufficient. Composition replaces inheritance through **struct embedding**: a struct can embed another struct (or interface) and the outer type promotes the inner type's fields and methods.

The model is alien on day one (especially the absence of constructors) and natural by week two. The payoff is that Go programs do not have the "is this object initialized?" or "did the constructor throw?" questions that haunt Java and TypeScript.

## 1. Value types vs reference types

Every type in Go is one of:

| Category | Types | Assignment behaviour |
|---|---|---|
| Built-in scalars | `bool`, all numeric types, `string` | copy |
| Composite values | `array` (fixed length), `struct` | copy (deep, recursively) |
| Reference (descriptor) | `slice`, `map`, `channel`, function value, interface value | copy of the descriptor; underlying data is shared |
| Pointer | `*T` | copy of the address |

```go
// Value: full copy
type Point struct { X, Y int }
p1 := Point{1, 2}
p2 := p1
p2.X = 99
fmt.Println(p1.X)  // 1 — p1 unchanged

// Slice: descriptor copied, backing array shared
s1 := []int{1, 2, 3}
s2 := s1
s2[0] = 99
fmt.Println(s1[0]) // 99 — shared backing array
```

The slice / map / channel rule trips up everyone arriving from Java (where every non-primitive is a reference) or TS (where every object is a reference). In Go, **struct values are copied**, but **slices, maps, and channels are descriptors that share their underlying storage**. This is covered in depth in [Slices, Arrays & Maps](04-slices-arrays-maps.md).

There is **no boxing**. An `int` is always a 64-bit (or 32-bit) word, never a heap-allocated wrapper. There is no `Integer` vs `int` distinction the way Java has — Go has just `int`.

## 2. The zero-value rule and why Go has no constructors

Every type has a **zero value**, the value its memory holds when freshly allocated and zeroed:

| Type | Zero value |
|---|---|
| Numeric | `0` |
| `bool` | `false` |
| `string` | `""` (empty, not nil — strings are not nullable) |
| Pointer, slice, map, channel, function, interface | `nil` |
| Struct | each field at its own zero value, recursively |
| Array | each element at its own zero value |

The **convention** that follows is: design types so that the zero value is immediately useful. The standard library is full of examples:

```go
var b bytes.Buffer  // zero value: empty buffer, ready to Write to
b.WriteString("hello")

var mu sync.Mutex   // zero value: unlocked mutex, ready to Lock
mu.Lock()
defer mu.Unlock()

var wg sync.WaitGroup  // zero value: zero counter, ready to Add
```

Compare with TypeScript / Java, where you typically need:

```ts
const buf = new Buffer();      // explicit construction
const mu = new Mutex();        // (hypothetical) — must initialize
```

```java
StringBuilder b = new StringBuilder();  // null until assigned
ReentrantLock mu = new ReentrantLock();
```

When zero is genuinely insufficient — because a field needs a non-zero default, or because construction has side effects (opening a file, dialling a connection) — the convention is a **factory function**:

```go
// Standard library: package "net/http"
func NewRequest(method, url string, body io.Reader) (*Request, error) { /* ... */ }

// Your code
func NewServer(addr string, opts ...Option) *Server { /* ... */ }
```

The naming convention is `New` for the primary constructor of a package's main type (e.g., `bytes.NewBuffer`, `errors.New`). This is convention only — the compiler does not enforce it.

Constructors that can fail return `(*T, error)`. The caller decides whether to handle the error or panic on it.

## 3. Structs and composite literals

A struct is a fixed sequence of named fields:

```go
type User struct {
    ID        int64
    Email     string
    CreatedAt time.Time
    isAdmin   bool       // unexported — visible only to this package
}
```

Construction via a **composite literal**:

```go
// Field-name form (preferred — robust to field reordering)
u := User{
    ID:        42,
    Email:     "alice@example.com",
    CreatedAt: time.Now(),
}

// Positional form (must list every field, in order — fragile, avoid in real code)
u := User{42, "alice@example.com", time.Now(), false}

// Zero composite literal
u := User{}
```

Notable rules:
- Trailing comma after the last field is **required** when the closing brace is on its own line. This is a `gofmt` convention; the parser accepts both.
- Omitted fields take their zero value.
- Field-name form is robust to adding new fields to the struct without breaking callers.

To get a pointer to a struct, take its address with `&`:

```go
u := &User{ID: 42, Email: "alice@example.com"}
// u has type *User
```

Use a pointer when you need to mutate the struct, when the struct is large enough that copying it is wasteful, or when the methods require pointer receivers (see [Pointers, Methods & Receivers](07-pointers-methods-receivers.md)).

## 4. Struct embedding: composition without inheritance

A struct can include a field with no name — only a type. The type is **embedded**, and the outer struct **promotes** the inner type's fields and methods so they appear as if they were on the outer type directly.

```go
type Logger struct {
    prefix string
}
func (l Logger) Log(msg string) {
    fmt.Println(l.prefix, msg)
}

type Server struct {
    Logger        // embedded, anonymous field
    addr  string
}

s := Server{Logger: Logger{prefix: "[server]"}, addr: ":8080"}
s.Log("started")           // promoted: equivalent to s.Logger.Log("started")
fmt.Println(s.prefix)      // promoted field access
```

This looks superficially like inheritance. It is not. Embedding is **composition with sugar**:

| Concern | Java inheritance | Go embedding |
|---|---|---|
| Override a method | `@Override` on subclass | redefine the method on the outer type — it shadows the inner |
| Call the "super" version | `super.method()` | `s.Logger.Log(...)` (explicit field reference) |
| `is-a` polymorphism | yes (`Server` is-a `Logger`) | no — the outer type is not assignable to the inner type |
| Multiple inheritance | not allowed | embed multiple types; ambiguity is a compile error you must resolve |

The key consequence: a `*Server` is not a `*Logger`. If you have a function that takes `*Logger`, you cannot pass `*Server` to it. Polymorphism in Go runs through **interfaces** (covered in [Interfaces & Structural Typing](05-interfaces-and-structural-typing.md)), not through inheritance.

Embedding interfaces is also valid and useful — especially in interface composition:

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Closer interface { Close() error }

// io.ReadCloser combines them
type ReadCloser interface {
    Reader
    Closer
}
```

## 5. Type definitions vs type aliases

These two declarations look similar but mean different things:

```go
type UserID int64       // type definition: UserID is a NEW type
type Bytes = []byte     // type alias: Bytes IS []byte
```

| | Type definition (`type T U`) | Type alias (`type T = U`) |
|---|---|---|
| Identity | `T` and `U` are distinct types | `T` and `U` are the same type |
| Implicit conversion | not allowed | not needed — same type |
| Method set | `T` has its own method set, separate from `U` | `T` shares `U`'s method set |
| Use case | domain types (`UserID`, `OrderID`), preventing accidental mixing | refactoring (renaming a type without breaking callers) |

Type definitions give you a primitive form of nominal typing on top of Go's structural type system:

```go
type UserID int64
type ProductID int64

func ban(u UserID) { /* ... */ }

var p ProductID = 7
ban(p)            // compile error: cannot use p (type ProductID) as type UserID
ban(UserID(p))    // explicit conversion required
```

This is Go's answer to the "branded types" pattern in TypeScript (see [TS Structural Typing & Type Compatibility](../../typescript/type-system/structural-typing.md) for the TS version).

Type aliases are rare in application code — their main use is graceful refactoring of standard library types (`os.PathError = fs.PathError` is the canonical example, added in Go 1.16 when `io/fs` was introduced).

## 6. `iota` and the enum idiom

Go has no `enum` keyword. The idiom uses `const` blocks plus the `iota` identifier, which is a per-block counter that starts at 0 and increments by 1 on each `const` line.

```go
type Status int

const (
    StatusPending Status = iota   // 0
    StatusActive                  // 1
    StatusSuspended               // 2
    StatusClosed                  // 3
)
```

`iota` resets to 0 at the start of each `const` block. Inside the block it advances by 1 on every line, regardless of whether `iota` is used on that line.

Common patterns:

```go
// Skip a value
const (
    _ = iota  // discard 0
    KB = 1 << (10 * iota)  // 1 << 10 = 1024
    MB                     // 1 << 20
    GB                     // 1 << 30
    TB                     // 1 << 40
)

// Bit flags
type Permission uint8
const (
    PermRead    Permission = 1 << iota  // 1
    PermWrite                            // 2
    PermExecute                          // 4
)
// PermRead | PermWrite == 3
```

For string output, the convention is to implement a `String()` method (so the type satisfies `fmt.Stringer`). The `stringer` tool (`go install golang.org/x/tools/cmd/stringer`) generates this for you:

```go
//go:generate stringer -type=Status
type Status int

const (
    StatusPending Status = iota
    StatusActive
    StatusSuspended
    StatusClosed
)
```

Running `go generate ./...` produces a `status_string.go` file with `func (s Status) String() string`.

A real downside: enums in Go are not closed. Any `int` literal can be assigned to `Status`, including values that don't correspond to declared constants. Validate at boundaries (input parsing, JSON unmarshalling) if you need exhaustiveness.

## 7. Struct tags

Struct fields can carry **tags** — a string literal after the type, used for metadata:

```go
type User struct {
    ID       int64     `json:"id"`
    Email    string    `json:"email" validate:"required,email"`
    Password string    `json:"-"`                          // omit from JSON
    Name     string    `json:"name,omitempty" db:"name"`
}
```

The format is convention: backtick string, space-separated `key:"value"` pairs. The `reflect` package exposes them via `field.Tag.Get("json")`.

Tags are how the standard library's `encoding/json`, `encoding/xml`, and most third-party validators / ORMs (`go-playground/validator`, `gorm`, `sqlx`) avoid annotation systems. They are inspected at runtime by reflection.

Common tag keys:

| Key | Used by | Example |
|---|---|---|
| `json` | `encoding/json` | `json:"email,omitempty"` |
| `xml` | `encoding/xml` | `xml:"email,attr"` |
| `db`, `column` | `sqlx`, `pgx`, `gorm` | `db:"email"` |
| `validate` | `go-playground/validator` | `validate:"required,email,max=255"` |
| `binding` | `gin` | `binding:"required"` |
| `form` | `gin`, `gorilla/schema` | `form:"email"` |

## Related

- [Go Tour for TS + Java Developers](01-go-tour-for-ts-java-devs.md) — basic types and syntax
- [Slices, Arrays & Maps](04-slices-arrays-maps.md) — the reference-type built-ins in detail
- [Interfaces & Structural Typing](05-interfaces-and-structural-typing.md) — polymorphism, contrasted with inheritance
- [Pointers, Methods & Receivers](07-pointers-methods-receivers.md) — when to take addresses, value vs pointer receivers
- [TypeScript Structural Typing & Type Compatibility](../../typescript/type-system/structural-typing.md) — branded types, structural compatibility (compare with Go's nominal type definitions)

## References

- The Go Programming Language Specification, "Types" — https://go.dev/ref/spec#Types
- The Go Programming Language Specification, "The zero value" — https://go.dev/ref/spec#The_zero_value
- The Go Programming Language Specification, "Composite literals" — https://go.dev/ref/spec#Composite_literals
- The Go Programming Language Specification, "Struct types" — https://go.dev/ref/spec#Struct_types (covers embedding and field promotion)
- The Go Programming Language Specification, "Iota" — https://go.dev/ref/spec#Iota
- Go FAQ, "Why is there no type inheritance?" — https://go.dev/doc/faq#inheritance
- Effective Go, "Embedding" — https://go.dev/doc/effective_go#embedding
- `golang.org/x/tools/cmd/stringer` — https://pkg.go.dev/golang.org/x/tools/cmd/stringer
