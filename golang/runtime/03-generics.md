---
title: "Generics in Go"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, runtime, generics, type-parameters, constraints]
---

# Generics in Go

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `runtime` `generics` `type-parameters` `constraints`

---

## Table of Contents

1. [Why Go added generics, and what they replaced](#1-why-go-added-generics-and-what-they-replaced)
2. [Type parameters: the `[T any]` syntax](#2-type-parameters-the-t-any-syntax)
3. [Constraints, type sets, and union elements](#3-constraints-type-sets-and-union-elements)
4. [`comparable`, `constraints.Ordered`, and `cmp.Ordered`](#4-comparable-constraintsordered-and-cmpordered)
5. [Worked example: the `slices` and `maps` packages](#5-worked-example-the-slices-and-maps-packages)
6. [Implementation: GC shape stenciling](#6-implementation-gc-shape-stenciling)
7. [What Go generics cannot do](#7-what-go-generics-cannot-do)
8. [When to prefer interfaces, when to prefer generics](#8-when-to-prefer-interfaces-when-to-prefer-generics)
9. [Comparison: Java type erasure, TypeScript generics](#9-comparison-java-type-erasure-typescript-generics)

## Summary

Generics shipped in **Go 1.18** on 22 March 2022, more than a decade after the first release. They add **type parameters** to functions and types — a parametric polymorphism mechanism that complements (rather than replaces) Go's interface-based polymorphism. The design lands in a deliberately restricted spot: there are no parameterized methods, no variance, and no method sets attached to a type parameter beyond what its constraint guarantees.

Constraints are interfaces, but with a wider grammar — they can describe **type sets** with union (`|`) and underlying-type (`~`) elements. The compiler implements generic functions through **GC shape stenciling**: one compiled body per memory layout (pointer-shape, int-shape, …), not per type. This keeps binary size sub-linear in instantiations and is closer to Java erasure than to C++ monomorphization, but with full compile-time type safety.

The right mental model: generics are for code that operates **uniformly across types** (containers, algorithms, slice/map utilities). For code that operates **polymorphically across behaviors** (anything you'd reach for an interface for), interfaces remain the right tool.

## 1. Why Go added generics, and what they replaced

For a decade, Go-without-generics had three workarounds, none good:

1. **`interface{}` plus type assertions.** A `Set` implementation stored `interface{}` values; callers paid a type assertion on every read. Loses type safety, costs an interface allocation per element, and the IDE cannot help with autocompletion.
2. **Code generation (`go generate`).** Tools like `genny` produced one source file per element type. Works, but adds a build step, makes diffs noisy, and trips up tooling that doesn't understand generated code.
3. **`reflect`.** Used for cross-cutting libraries (encoding/json, fmt). Slow and disables compile-time checking entirely; see [`04-reflection-and-unsafe.md`](04-reflection-and-unsafe.md).

The "An Introduction to Generics" Go blog post (2022-03-22) puts the move plainly: "Generics are the biggest change we've made to Go since the first open source release." The proposal that landed (Type Parameters, design doc 43651) had been debated since 2009; multiple alternative designs were prototyped and discarded before this one shipped.

The historical use-cases that drove adoption: typed `Min`/`Max`/`Sort`, `slices.Contains`, `maps.Keys`, generic channels of `T`, generic LRU caches, and database-row scanning helpers. Most of these had `interface{}` versions in the standard library before 1.18 (`sort.Slice`, for example) — generics let them shed the runtime cost.

## 2. Type parameters: the `[T any]` syntax

Type parameters appear in **square brackets** before the regular parameter list:

```go
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

x := Min[int](3, 5)   // explicit instantiation
y := Min(3.0, 5.0)    // type argument inferred from arguments
```

Generic types take the same shape:

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    n := len(s.items) - 1
    v := s.items[n]
    s.items = s.items[:n]
    return v, true
}

s := &Stack[int]{}
s.Push(1)
s.Push(2)
v, _ := s.Pop()
```

Two language details that bite TS/Java developers:

- **`any` is a predeclared alias for `interface{}`** (Go 1.18). Use `any` in new code; `interface{}` is the old spelling.
- **Type inference is purely from arguments**, not from return type context. `var x int = Identity(3)` works; `var x int = Identity[any](3)` does not type-check the way a TS user might expect — `int` is not assignable to `any` returned out of a generic. Go's inference is intentionally weaker than Hindley-Milner; if it can't figure it out, you instantiate explicitly.

## 3. Constraints, type sets, and union elements

A constraint is just an interface. The novel part: interfaces used as constraints can carry **type elements** that older Go interfaces could not.

```go
type Number interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}

func Sum[T Number](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}
```

The key pieces:

| Element | Meaning |
|---|---|
| `int \| int64` | The type set is exactly `int` or `int64` |
| `~int` | The type set is **all types whose underlying type is `int`** — including `type Celsius int` |
| `int \| string` | Union of disjoint sets |
| `int; String() string` | A type element AND a method element — the type must be in the union AND have the method |

The `~` (approximation) operator matters because Go programmers routinely define named types over primitives (`type UserID int64`). Without `~`, `Sum` would reject `[]UserID`.

A constraint cannot be used as an ordinary interface type. `var x Number` is a compile error if `Number` includes a union or `~` element — those interfaces can only appear in type-parameter lists.

## 4. `comparable`, `constraints.Ordered`, and `cmp.Ordered`

Two predeclared constraints ship with the language:

- **`any`** — equivalent to `interface{}`.
- **`comparable`** — the set of types that support `==` and `!=`. Required for `map[K]V` keys and for any generic algorithm that compares values.

```go
func Index[T comparable](haystack []T, needle T) int {
    for i, v := range haystack {
        if v == needle {
            return i
        }
    }
    return -1
}
```

`comparable` was widened in **Go 1.20**: type parameters declared as `comparable` can now be satisfied by interface types (which are comparable, even though `==` may panic at runtime if the dynamic types are not comparable). This was a deliberate trade-off — strictly safer Go 1.18 rules turned out to be too restrictive in practice.

For ordering, the story evolved across two releases:

- **Go 1.18**: `golang.org/x/exp/constraints.Ordered` — an experimental package, off the standard tree.
- **Go 1.21** (released August 2023): `cmp.Ordered` lands in the standard library as the canonical name. The `cmp` package also adds `cmp.Compare[T cmp.Ordered](a, b T) int` and `cmp.Less[T cmp.Ordered](a, b T) bool`.

```go
import "cmp"

func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

Use `cmp.Ordered` in new code on Go 1.21+. The `constraints` package in `golang.org/x/exp` is now a thin shim and may stay there indefinitely, but the standard-library name is the one to internalize.

## 5. Worked example: the `slices` and `maps` packages

Go 1.21 promoted two packages from `golang.org/x/exp` into the standard library:

- **`slices`** — `Contains`, `Index`, `Sort`, `SortFunc`, `BinarySearch`, `Equal`, `Compact`, `Reverse`, `Insert`, `Delete`, `Min`, `Max`, …
- **`maps`** — `Keys`, `Values`, `Equal`, `Clone`, `Copy`, `DeleteFunc`. (Note: in Go 1.23 these returned iterators, see the Go 1.23 release notes if you're on a recent toolchain.)

These are the canonical worked examples for "what Go generics actually look like in production."

```go
import (
    "cmp"
    "slices"
)

type Order struct {
    ID    string
    Total float64
}

orders := []Order{
    {"o-3", 49.00},
    {"o-1", 12.50},
    {"o-2", 99.99},
}

// Sort by ID using a custom comparison
slices.SortFunc(orders, func(a, b Order) int {
    return cmp.Compare(a.ID, b.ID)
})

// Type-safe contains check on a slice of any comparable element
ids := []string{"o-1", "o-2", "o-3"}
has := slices.Contains(ids, "o-2")
```

Before 1.18 this required either `sort.Slice` (closure-allocating, no compile-time check that the index is in range) or hand-rolled type-specific code. The generic `slices.SortFunc` is faster than `sort.Slice` on most workloads — partly because the compiler can specialize the comparison call site per shape.

## 6. Implementation: GC shape stenciling

The Go team chose a deliberately middle-of-the-road compilation strategy.

- **C++ templates** monomorphize: `std::vector<int>` and `std::vector<long>` become two distinct compiled bodies, each fully type-specialized. Excellent runtime, terrible compile time and binary size on large generic codebases.
- **Java generics** erase: the bytecode for `ArrayList<Integer>` and `ArrayList<String>` is identical; `T` is erased to `Object`. Tiny binaries, but you box every primitive (`Integer`, not `int`) and lose type information at runtime.
- **Go generics** stencil **per GC shape**, not per type. A "shape" is the memory layout the garbage collector needs to know — pointer-shape, int-shape, struct-shape with these field offsets, etc. All pointer-typed instantiations share one compiled body; the function takes a hidden "dictionary" parameter describing the actual type at runtime.

Concretely: if you write `func Map[T, U any](s []T, f func(T) U) []U` and instantiate it with `(string, int)`, `(*User, *Order)`, and `(*Cart, *Invoice)`, the compiler typically emits **two** compiled bodies — one for the pointer/pointer shape and one for the string/int shape. The pointer/pointer body is shared by all three pointer-shaped pairs, dispatching through the dictionary for type-specific operations.

Consequences:

- **Binary size scales with the number of distinct shapes**, not types. In practice that is bounded.
- **Inlining is harder than for C++ templates.** Hot generic code paths can be slower than hand-written equivalents because the compiler may not inline through the shape body. Benchmarks before reaching for generics in tight loops are still worth running.
- **Method dispatch through type parameters** uses the dictionary — a level of indirection comparable to interface dispatch, not as cheap as a direct call.

The original implementation strategy is described in PlanetScale-engineer-and-Go-team-member Keith Randall's GopherCon 2021 talk, "Generics implementation: GC Shape Stenciling," and in the implementation design docs in the `golang/proposal` repo. The 1.18 release notes themselves stay silent on the technique — it is an implementation detail and the Go team reserves the right to switch strategies.

## 7. What Go generics cannot do

Coming from TS or Java, several things you might expect simply don't exist:

| Want | Status |
|---|---|
| Methods with new type parameters on a generic type (`func (s *Stack[T]) Map[U any](f func(T) U) *Stack[U]`) | **Not allowed.** Methods can only use the type parameters declared on the receiver. |
| Variance (covariance, contravariance) | **None.** `[]Cat` is not assignable to `[]Animal` in any form, generic or not. |
| Specialization (a special-case body for `T == string`) | **None.** Can be simulated with `any` + type switch, but loses generic type safety. |
| Higher-kinded types (`Functor[F[_]]`) | **None.** Type parameters cannot themselves be parameterized. |
| Method-set constraints written as `T: Stringer` shorthand | **No.** Use `interface { String() string }` as the constraint. |
| Default type parameters (`Cache[K = string, V any]`) | **None.** All type parameters are required. |
| Type-parameter access to fields of a struct constraint | **Limited.** Field access on a type parameter requires every type in the constraint's type set to have that field at the same offset; in practice, write a method instead. |

The "no parameterized methods" rule is the one TS users hit hardest — `Array.prototype.map<U>` does not have a direct Go translation. Idiomatic Go pulls the operation out as a top-level function: `slices.Map[T, U any](s []T, f func(T) U) []U` (community implementation; not in standard library as of Go 1.21).

## 8. When to prefer interfaces, when to prefer generics

The decision rule the Go team has repeatedly given:

> If you find yourself writing the exact same code twice, with the only difference being that the two copies use different types, consider whether you can use a type parameter. Another way to put it: avoid type parameters until you notice that you are about to write the same code multiple times.
>
> — "When To Use Generics" Go blog, 2022

Concrete heuristic:

| Use generics | Use interfaces |
|---|---|
| Operating on **container shapes** (slice/map/channel of T) | Operating on **behavior** (anything that calls a method) |
| Functions where the *type* matters but the *operations* are fixed (`==`, `<`, `+`) | Functions where the operations are open and pluggable (`Read`, `Encode`) |
| Returning a value of the input's type (`Identity[T any](v T) T`) | Returning a polymorphic value (`io.Reader`, `error`) |
| Static dispatch is important for the hot path | Open extension by downstream packages is important |
| You'd otherwise write `interface{}` plus a type assertion | You're describing a contract a type can satisfy |

A classic mistake: using generics where an interface is a better fit. `func Process[T Stringer](v T)` is almost always worse than `func Process(v Stringer)`. The generic version produces a stencil per shape with no type-safety benefit over the interface version, since the only operation in scope is `String()` anyway.

The reverse mistake — using interfaces where generics fit — was the entire pre-1.18 status quo. `func Min(a, b interface{}) interface{}` is the bad version; `func Min[T cmp.Ordered](a, b T) T` is correct.

## 9. Comparison: Java type erasure, TypeScript generics

| | Go | Java | TypeScript |
|---|---|---|---|
| Compile-time check | Yes | Yes | Yes |
| Runtime type info | **Per-shape dictionary** carried into generic body | **Erased** — `List<String>.getClass()` returns `List`, the `String` is gone | **Erased** — types disappear at runtime entirely |
| Primitive support | Native; no boxing | **Boxed only** — `List<Integer>`, never `List<int>` | N/A (TS has no primitive/object split at the type level the way Java does) |
| Variance | None | Use-site (`List<? extends Animal>`) | Structural; covariant by default for read-only positions |
| Method-level type parameters on generic types | **Disallowed** | Allowed (`<U> U map(Function<T,U> f)`) | Allowed |
| Higher-kinded types | None | None | None (limited, via tricks) |
| Conditional types / `infer` | **None** | None | Yes (`T extends U ? X : Y`, `infer R`) |
| Implementation strategy | GC-shape stenciling | Erasure + bridge methods | Erasure |

For a TS developer, the single biggest adjustment is the absence of conditional types. Patterns like `ReturnType<T>`, `Parameters<T>`, `Awaited<T>` cannot be expressed in Go's type system; you write functions, not types, that operate at the type-parameter level. See [TypeScript Conditional Types and `infer`](../../typescript/type-system/conditional-types-and-infer.md) for the contrasting model.

For a Java developer, the biggest adjustment is that Go does not box primitives — `Sum[int64](xs)` operates on the actual `int64` storage, not on `Long` wrappers. The downside is that the compiler must specialize per shape, where the JVM gets one body for free.

For both: there are **no methods on a type parameter beyond what the constraint provides**. There is no `T : Comparable<T>` self-bound nor a `T extends keyof O` index-access pattern. Constraints are interfaces, full stop.

## Related

- [Types, Zero Values & Composite Literals](../fundamentals/03-types-zero-values-composites.md) — the `~` operator and named types
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — the polymorphism mechanism generics complement, not replace
- [Pointers, Methods & Receivers](../fundamentals/07-pointers-methods-receivers.md) — why methods on a generic receiver can't introduce new type parameters
- [The Garbage Collector](01-garbage-collector.md) — what "GC shape" means in shape stenciling
- [Escape Analysis & Allocation](02-escape-analysis-and-allocation.md) — why generic instantiation does not necessarily produce extra allocations
- [Reflection & `unsafe`](04-reflection-and-unsafe.md) — the reflect-based libraries generics did *not* eliminate
- [TypeScript Generics & Constraints](../../typescript/type-system/generics-and-constraints.md) — structural constraints, default parameters, variance
- [TypeScript Conditional Types & `infer`](../../typescript/type-system/conditional-types-and-infer.md) — the type-level computation Go lacks

## References

- "An Introduction to Generics" — Go blog, 2022-03-22 — https://go.dev/blog/intro-generics
- Go 1.18 release notes (generics, `any`, `comparable`, `~` token) — https://go.dev/doc/go1.18
- Go 1.20 release notes (relaxed `comparable`) — https://go.dev/doc/go1.20
- Go 1.21 release notes (`cmp`, `slices`, `maps` packages) — https://go.dev/doc/go1.21
- "When To Use Generics" — Go blog, 2022-04-12 — https://go.dev/blog/when-generics
- Type Parameters Proposal (design 43651) — https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md
- The Go Programming Language Specification, "Type parameter declarations" — https://go.dev/ref/spec#Type_parameter_declarations
- The Go Programming Language Specification, "Type constraints" — https://go.dev/ref/spec#Type_constraints
- `slices` package — https://pkg.go.dev/slices
- `maps` package — https://pkg.go.dev/maps
- `cmp` package — https://pkg.go.dev/cmp
- `golang.org/x/exp/constraints` — https://pkg.go.dev/golang.org/x/exp/constraints
