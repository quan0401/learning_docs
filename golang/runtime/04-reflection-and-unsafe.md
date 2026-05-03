---
title: "Reflection & `unsafe`"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, runtime, reflection, unsafe, encoding]
---

# Reflection & `unsafe`

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `runtime` `reflection` `unsafe` `encoding`

---

## Table of Contents

1. [What `reflect` is, and when it is the right tool](#1-what-reflect-is-and-when-it-is-the-right-tool)
2. [The three laws of reflection](#2-the-three-laws-of-reflection)
3. [Real-world reflection users, and why generics didn't kill them](#3-real-world-reflection-users-and-why-generics-didnt-kill-them)
4. [Reflection performance and how to mitigate it](#4-reflection-performance-and-how-to-mitigate-it)
5. [`unsafe.Pointer` and the four conversion rules](#5-unsafepointer-and-the-four-conversion-rules)
6. [`unsafe.Slice`, `unsafe.String`, `unsafe.SliceData`, `unsafe.StringData`](#6-unsafeslice-unsafestring-unsafeslicedata-unsafestringdata)
7. [When `unsafe` is justified](#7-when-unsafe-is-justified)
8. [Why most code should never reach for `unsafe`](#8-why-most-code-should-never-reach-for-unsafe)
9. [Comparison with Java reflection](#9-comparison-with-java-reflection)

## Summary

Go has two escape hatches from its static type system: **`reflect`** for dynamic type inspection, and **`unsafe`** for bypassing the type system entirely. They serve very different audiences. `reflect` is a public API used by the most popular libraries in the ecosystem — `encoding/json`, `database/sql`, validators, ORMs. `unsafe` is a deliberately uncomfortable package whose contents are explicitly **outside the Go 1 compatibility promise**.

The mental model: `reflect` lets a library do at runtime what a generic function would do at compile time, at a cost (allocations, no inlining, ~10–100× slower than the equivalent direct code). `unsafe` lets you reinterpret memory and skip allocations, at a cost (the runtime can break your program between point releases without warning, and the GC may corrupt or miss your data if you violate the conversion rules).

The pragmatic rules of the road: prefer generics over reflection where the API permits, cache `reflect.Type` lookups, and avoid `unsafe` until you can name exactly which Go release's behavior you depend on.

## 1. What `reflect` is, and when it is the right tool

The `reflect` package exposes Go's runtime type information as ordinary Go values. The two core types:

```go
package reflect

type Type interface {
    // Methods describing the type: Kind, Name, NumField, Field(i), Method(i), …
}

type Value struct {
    // An (immutable) handle to a value of arbitrary type.
    // Methods: Kind, Interface, Field(i), Call(args), SetInt(n), …
}
```

You enter the reflection world from an `interface{}` value:

```go
import "reflect"

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

u := User{ID: 7, Name: "Ada"}
t := reflect.TypeOf(u)   // reflect.Type
v := reflect.ValueOf(u)  // reflect.Value

for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)
    fmt.Printf("%s = %v (json tag: %q)\n",
        field.Name, v.Field(i).Interface(), field.Tag.Get("json"))
}
```

When is reflection the right tool?

- **The library doesn't know the user's types in advance.** Anything that walks an arbitrary struct — JSON encoder, database row scanner, validator, equality checker — fits.
- **Struct tags drive behavior.** `` `json:"foo,omitempty"` ``, `` `db:"user_id"` ``, `` `validate:"required,email"` ``. Tags are accessible only through `reflect.StructField.Tag`.
- **Type-driven dispatch where the registry is dynamic.** A protocol decoder that picks a handler based on the message struct's type.

When is reflection the wrong tool?

- You know the type at compile time. Use the type directly.
- You almost know the type at compile time and just want to skip a few keystrokes. Use generics.
- The cost of slowness in your hot path matters. Reflection is on the order of 10–100× slower than direct field access; benchmarking is mandatory.

## 2. The three laws of reflection

Rob Pike's "Laws of Reflection" (Go blog, 2011) names them. The laws map directly onto the API:

> 1. Reflection goes from interface value to reflection object.
> 2. Reflection goes from reflection object to interface value.
> 3. To modify a reflection object, the value must be settable.

**Law 1 — interface → reflection.** `reflect.TypeOf(x)` and `reflect.ValueOf(x)` take an `interface{}` (formally `any`) argument and return the runtime type and value descriptors. There is no way to reflect over a value that is not first wrapped in an interface, because `reflect.TypeOf` literally takes `interface{}`.

```go
var f float64 = 3.4
t := reflect.TypeOf(f)        // float64
v := reflect.ValueOf(f)       // 3.4 of kind Float64
fmt.Println(t.Kind() == reflect.Float64) // true
```

**Law 2 — reflection → interface.** `Value.Interface()` returns the value back as an `interface{}`, which you then type-assert to recover the concrete type:

```go
y := v.Interface().(float64)
```

This round-trip is how reflection-based decoders hand the result back to the caller's code.

**Law 3 — settability.** A `reflect.Value` is **settable** only if it refers to addressable storage that the caller has explicitly given access to. The pattern:

```go
var x float64 = 3.4
p := reflect.ValueOf(&x).Elem()  // pass &x, then .Elem() to dereference
p.SetFloat(7.1)                  // mutates x
fmt.Println(x)                   // 7.1
```

`reflect.ValueOf(x).SetFloat(7.1)` panics — you've passed a copy, and the reflection machinery cannot mutate it.

Settability is what makes JSON-into-struct decoding work: `json.Unmarshal` requires `*T`, calls `reflect.ValueOf(target).Elem()` to get a settable handle, then walks the struct fields and calls `SetString`, `SetInt`, `Set`, etc.

## 3. Real-world reflection users, and why generics didn't kill them

Generics (Go 1.18) eliminated *some* uses of reflection. The famously gnarly ones survived. The reason: reflection is needed when **the consumer's types are unknown to the library author**, and parametric polymorphism does not help with that — the library doesn't know what `T` to instantiate over.

| Library | Why reflection | Could generics replace it? |
|---|---|---|
| `encoding/json`, `encoding/xml`, `encoding/gob` | Walks arbitrary structs to read/write fields driven by tags | **No** — the library has no compile-time view of the user's types |
| `fmt.Printf` and the verbs `%v`, `%+v`, `%T` | Needs to format an arbitrary value's type and fields | **No** — same reason |
| `database/sql` `Rows.Scan(&dst)` | Maps SQL columns to arbitrary destination pointers | Partial — generic helpers (e.g., `sqlc`, `sqlx`) exist but the standard library still uses reflection |
| Validators (`go-playground/validator`) | Walks structs and reads `validate:"…"` tags | **No** — tag-driven |
| ORMs (`gorm`, `ent`) | Maps struct types to tables, reads `gorm:"…"` tags | **No** — same |
| `reflect.DeepEqual` | Compares values of any type | **Mostly no** — `slices.Equal` and `maps.Equal` cover slices/maps but not arbitrary structs |
| Old-style `Min`, `Max`, `Sort`, `Contains` | Needed reflection or `interface{}` to be polymorphic | **Yes** — `slices.Min`, `slices.Sort`, `slices.Contains` since Go 1.21 |

The pragmatic split: **structural code about user-defined types** stays reflection-based; **algorithmic code parameterized by element type** is generic now.

There is a third middle ground emerging in modern libraries: **code generation from struct definitions** (e.g., `easyjson`, `ffjson`, `goverter`). These read your struct types at build time and emit type-specific code, avoiding both reflection and `interface{}`. They are a maintenance choice, not a language feature.

## 4. Reflection performance and how to mitigate it

Reflection is slow for three concrete reasons:

1. **`reflect.TypeOf(x)` and `reflect.ValueOf(x)` allocate** — the argument must be wrapped in an `interface{}`, which boxes value-types onto the heap. See [`02-escape-analysis-and-allocation.md`](02-escape-analysis-and-allocation.md) for why.
2. **Field access goes through a hash lookup** — `Type.FieldByName("ID")` walks a name → index map. Even `Type.Field(i)` involves bounds checks and copies a `StructField` struct.
3. **Method dispatch happens through an interface** — there is no way for the compiler to inline through a `reflect.Value.Call`.

Mitigations, in rough order of impact:

**Cache `reflect.Type`.** A `reflect.Type` is comparable and uniquely identifies a type. Most reflection-heavy libraries build a cache keyed by `reflect.Type`:

```go
type encoder struct {
    typeCache sync.Map // map[reflect.Type]*typeInfo
}

func (e *encoder) infoFor(t reflect.Type) *typeInfo {
    if cached, ok := e.typeCache.Load(t); ok {
        return cached.(*typeInfo)
    }
    info := buildTypeInfo(t)  // expensive: walks all fields, parses tags
    actual, _ := e.typeCache.LoadOrStore(t, info)
    return actual.(*typeInfo)
}
```

`encoding/json` does exactly this internally. The first encode of a new struct type pays full reflection cost; every subsequent encode hits the cache.

**Avoid `Interface()` on the hot path.** Each call boxes the value. If you are summing integers, `Value.Int()` returns `int64` directly without allocation; only fall back to `Interface()` when you genuinely need an `any`.

**Generate code instead.** For the highest-throughput JSON paths, code generators (`easyjson`, `sonic`) produce type-specific encoders that match hand-written code's performance. The cost is a build-time step.

**Use `unsafe` to skip the conversion entirely** (only in carefully audited libraries; see §7). High-performance JSON libraries reach for this in the inner loop.

## 5. `unsafe.Pointer` and the four conversion rules

`unsafe.Pointer` is Go's untyped pointer. It can be converted to and from any concrete pointer type and to/from `uintptr`. The package documentation enumerates the **only four valid conversion patterns** — anything else is undefined behavior, and the runtime garbage collector may corrupt or move data behind your back.

**Rule 1 — Reinterpret one pointer type as another, when sizes and layouts are compatible.**

```go
import "unsafe"

func Float64bits(f float64) uint64 {
    return *(*uint64)(unsafe.Pointer(&f))
}
```

This is how `math.Float64bits` is actually implemented. The IEEE 754 representation of a `float64` is exactly the same memory as a `uint64`, so reinterpreting is sound.

**Rule 2 — Convert `Pointer` to `uintptr` (one-way) for printing or hashing.** The reverse direction loses GC tracking; the GC sees `uintptr` as plain integers.

```go
fmt.Printf("%p stored as 0x%x\n", &x, uintptr(unsafe.Pointer(&x)))
```

You may **not** save the `uintptr` to a variable, do work, and convert back. The original object may have been moved or freed.

**Rule 3 — Pointer arithmetic via `Pointer → uintptr → arithmetic → Pointer` in a single expression.**

```go
// Get a pointer to the third element of an array
p := unsafe.Pointer(uintptr(unsafe.Pointer(&arr[0])) + 2*unsafe.Sizeof(arr[0]))
```

The conversion *must* be a single expression. The Go runtime treats the ephemeral `uintptr` value specially for the duration of that one expression and updates it if the GC moves the object. Storing the intermediate `uintptr` in a variable breaks that guarantee.

`unsafe.Add(ptr, n)` (Go 1.17) is the safe shorthand for this pattern:

```go
p := unsafe.Add(unsafe.Pointer(&arr[0]), 2*unsafe.Sizeof(arr[0]))
```

**Rule 4 — Convert to `uintptr` directly in a syscall argument list.** Required for `syscall.Syscall` and friends, which take `uintptr` arguments by convention.

```go
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(&buf[0])), uintptr(len(buf)))
```

Same rule as #3: the conversion must appear in the call expression itself, not in a temporary.

These four patterns are exhaustive. The package documentation is explicit:

> The following patterns involving Pointer are valid. Code not using these patterns is likely to be invalid today or to become invalid in the future.

## 6. `unsafe.Slice`, `unsafe.String`, `unsafe.SliceData`, `unsafe.StringData`

Before Go 1.17, the standard pattern for zero-copy `string ↔ []byte` conversion was to construct a `reflect.SliceHeader` or `reflect.StringHeader` by hand:

```go
// PRE-1.17: fragile, undefined behavior in some configurations
sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
b := *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
    Data: sh.Data,
    Len:  sh.Len,
    Cap:  sh.Len,
}))
```

This relied on the layout of the runtime's slice/string headers, which the Go team explicitly does not guarantee. Programs using this pattern broke on the migration to register-based calling conventions in Go 1.17.

The `unsafe` package now has explicit, supported functions:

- **`unsafe.Slice(ptr *T, len IntegerType) []T`** (Go 1.17) — construct a slice from a pointer and length.
- **`unsafe.SliceData(s []T) *T`** (Go 1.20) — recover the pointer to the underlying array.
- **`unsafe.String(ptr *byte, len IntegerType) string`** (Go 1.20) — construct a string from a byte pointer and length.
- **`unsafe.StringData(s string) *byte`** (Go 1.20) — recover the pointer to the underlying bytes.

The Go 1.20 release notes summarize the design intent:

> Along with Go 1.17's Slice, these functions now provide the complete ability to construct and deconstruct slice and string values, without depending on their exact representation.

Modern zero-copy conversion looks like this:

```go
// []byte to string without copying (caller must guarantee the bytes are not mutated)
func bytesToString(b []byte) string {
    if len(b) == 0 {
        return ""
    }
    return unsafe.String(unsafe.SliceData(b), len(b))
}

// string to []byte without copying (caller MUST NOT mutate the result)
func stringToBytes(s string) []byte {
    if s == "" {
        return nil
    }
    return unsafe.Slice(unsafe.StringData(s), len(s))
}
```

The mutability constraint is real and not optional: Go strings are immutable, and the runtime relies on that for things like interning of constants. Mutating the bytes of `stringToBytes("hello")` is undefined behavior, and the example pattern is only safe when the caller controls the lifetime and treats the result as read-only.

`reflect.SliceHeader` and `reflect.StringHeader` are now formally deprecated. Use `unsafe.Slice` / `unsafe.String` and friends in new code.

## 7. When `unsafe` is justified

The legitimate cases are narrow:

1. **Zero-copy `string ↔ []byte` in performance-critical decoders.** JSON, protobuf, and HTTP header parsers reach for this in their hottest loops. The standard library itself does so internally — see the `runtime` package's use of `unsafe.String` for string interning.
2. **Atomic operations on arbitrary types.** `sync/atomic` exposes typed `*Int64`, `*Pointer[T]` etc. as of Go 1.19, but if you need a CAS on a custom type, `unsafe.Pointer` is the bridge.
3. **Low-level system interop.** `syscall`, `os`, `cgo` boundaries; Linux-specific `io_uring` bindings; the `mmap` packages; database drivers that wrap C libraries.
4. **Cache-line padding for false-sharing avoidance** in lock-free data structures. Niche but real in trading systems and runtime libraries.
5. **Hot-path field access in reflection-heavy frameworks** that have already cached the offset. Examples: high-performance JSON encoders compute the offset of each struct field once and use `unsafe.Add` to read it on every encode.

In all of these the `unsafe` use is **localized** — a handful of functions in one package, audited carefully, accompanied by tests against multiple Go versions.

## 8. Why most code should never reach for `unsafe`

The Go 1 compatibility promise (https://go.dev/doc/go1compat) is explicit:

> Use of package unsafe. Packages that import unsafe may depend on internal properties of the Go implementation. We reserve the right to make changes to the implementation that may break such programs.

Concretely, this has happened. Examples:

- **Go 1.17's register-based calling convention** broke programs that depended on stack frame layout assumed by `unsafe`-using libraries.
- **The garbage collector's handling of `uintptr`** has changed multiple times; programs that violated rule #3 from §5 silently got memory corruption on later versions.
- **`reflect.SliceHeader` and `reflect.StringHeader`** were re-defined to not match the runtime's actual layout in Go 1.20+.

Every `unsafe` usage is a fresh maintenance liability. Before reaching for it:

- Have you benchmarked the safe version? "Reflection feels slow" is not a benchmark.
- Have you tried generics? They are zero-cost at the call site for most cases that previously needed `unsafe`-based reflection tricks.
- Have you tried a code generator (e.g., `go generate` plus a tool that emits hand-rolled marshaling)?
- Are you writing a library or an application? Library `unsafe` is a tax paid by every consumer.

The default answer: do not import `unsafe`. The exceptions are narrow and listed in §7.

## 9. Comparison with Java reflection

| | Go `reflect` | Java `java.lang.reflect` |
|---|---|---|
| Capability | Read types and tags, read/write fields, call methods | Read types and annotations, read/write fields (incl. private), call methods, modify access |
| Compile-time check | Almost none (reflective calls fail at runtime) | None — same |
| Cost | Allocations (boxing into `interface{}`), no inlining | Method handles + inline caches help on hot paths; still significantly slower than direct |
| Annotations / metadata | Struct tags (`` `json:"x"` ``) — string-keyed | Annotations — typed, with their own runtime classes |
| Mutating final / unexported fields | Cannot set unexported fields without `unsafe` | `field.setAccessible(true)` — controversial but available |
| Equivalent of `unsafe` | The `unsafe` package; `reflect.Value` with `unsafe.Pointer` | `sun.misc.Unsafe` (legacy); `VarHandle`, `MethodHandles.Lookup` (modern) |
| Cross-version stability | `unsafe` explicitly excluded from Go 1 compat | `sun.misc.Unsafe` similarly unstable; `Unsafe` deprecation underway as of JDK 23 |

The biggest practical difference: **Java reflection is a deeply integrated runtime feature** (frameworks like Spring, Hibernate, and Jackson are essentially built on it) and the JVM's JIT can specialize through reflective call sites with adaptive inlining. **Go reflection is a library on top of the static type system**, with a tighter relationship to memory layout and no JIT to recover the lost speed. The result: idiomatic Go avoids reflection on the request path; idiomatic Java does not.

For the Spring developer learning Go, the closest analogue to "annotation-driven framework magic" is struct tags + a code generator, not runtime reflection. Sticking with reflection at request rate is what makes Go services slow.

## Related

- [Types, Zero Values & Composite Literals](../fundamentals/03-types-zero-values-composites.md) — what struct tags actually are
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — the (type, value) pair `reflect` exposes
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — addressable vs non-addressable values, the basis of settability
- [Generics in Go](03-generics.md) — when to use type parameters instead of reflection
- [The Garbage Collector](01-garbage-collector.md) — why `uintptr` violates GC tracking
- [Escape Analysis & Allocation](02-escape-analysis-and-allocation.md) — why `reflect.ValueOf(x)` allocates
- [Java Patterns](../../java/) — annotation-driven frameworks and what their Go equivalents look like

## References

- "The Laws of Reflection" — Rob Pike, Go blog, 2011-09-06 — https://go.dev/blog/laws-of-reflection
- `reflect` package — https://pkg.go.dev/reflect
- `unsafe` package — https://pkg.go.dev/unsafe
- Go 1.17 release notes (`unsafe.Add`, `unsafe.Slice`) — https://go.dev/doc/go1.17
- Go 1.20 release notes (`unsafe.SliceData`, `unsafe.String`, `unsafe.StringData`) — https://go.dev/doc/go1.20
- Go 1 compatibility promise (excludes `unsafe`) — https://go.dev/doc/go1compat
- `encoding/json` — https://pkg.go.dev/encoding/json
- `database/sql` `Rows.Scan` — https://pkg.go.dev/database/sql#Rows.Scan
