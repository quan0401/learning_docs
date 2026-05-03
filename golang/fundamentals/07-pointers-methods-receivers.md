---
title: "Pointers, Methods & Receivers"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, pointers, methods, receivers, escape-analysis]
---

# Pointers, Methods & Receivers

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `pointers` `methods` `receivers` `escape-analysis`

---

## Table of Contents

1. [Pointers, briefly: addresses, no arithmetic](#1-pointers-briefly-addresses-no-arithmetic)
2. [Methods on types](#2-methods-on-types)
3. [Value receivers vs pointer receivers](#3-value-receivers-vs-pointer-receivers)
4. [Method sets and which interfaces a type satisfies](#4-method-sets-and-which-interfaces-a-type-satisfies)
5. [The consistency rule](#5-the-consistency-rule)
6. [Escape analysis: stack vs heap](#6-escape-analysis-stack-vs-heap)
7. [Nil pointer dereferences](#7-nil-pointer-dereferences)

## Summary

Go has pointers but not pointer arithmetic. The address-of operator `&` and the dereference operator `*` are the two pieces of syntax you need; everything else about pointers in C is gone. Methods attach to types — either a concrete type or a type definition — and the receiver can be a value (`func (s S) M()`) or a pointer (`func (s *S) M()`). The choice between the two is a design decision with concrete consequences: pointer receivers can mutate, are cheaper to call for large structs, and change which interfaces the type satisfies. Once you pick a style for a type, **be consistent across all its methods**.

The Go compiler decides whether your local variables live on the stack or the heap through a static analysis called **escape analysis**. You don't manage memory directly, but you can read the analysis output to see why a pointer caused a heap allocation.

## 1. Pointers, briefly: addresses, no arithmetic

```go
x := 42
p := &x        // p is a *int
fmt.Println(*p) // 42 — dereference

*p = 99
fmt.Println(x)  // 99
```

Rules:
- `&x` takes the address of an addressable expression (local variables, struct fields, array/slice elements). You **cannot** take the address of a map element (`&m["key"]` is a compile error).
- `*p` dereferences a pointer. If `p` is nil, this panics.
- There is **no pointer arithmetic**. `p + 1` does not compile. (`unsafe.Pointer` lets you bypass this for low-level interop, covered in Tier 3.)
- The zero value of any pointer type is `nil`.

Pointers exist for two reasons: to share storage (so a callee can mutate caller state) and to avoid copying (so a large struct doesn't get duplicated on every call).

```go
// Without pointer: callee gets a copy, mutation invisible
func grow(s string) { s += "!" }
x := "hi"; grow(x); fmt.Println(x)  // "hi"

// With pointer: callee mutates caller's variable
func grow(s *string) { *s += "!" }
x := "hi"; grow(&x); fmt.Println(x)  // "hi!"
```

## 2. Methods on types

A method is a function with a **receiver**:

```go
type Counter struct {
    n int
}

func (c Counter) Value() int { return c.n }    // value receiver
func (c *Counter) Incr()     { c.n++ }         // pointer receiver
```

The receiver is named (`c` here, by convention a 1–2 letter name derived from the type — not `this` or `self`) and lives at the start of the parameter list. Inside the method body, the receiver is just a regular variable.

You can attach methods to any type defined in the **same package**. This includes type definitions over built-ins:

```go
type Celsius float64

func (c Celsius) Fahrenheit() Celsius {
    return c*9/5 + 32
}

t := Celsius(100)
fmt.Println(t.Fahrenheit())  // 212
```

You **cannot** attach methods to types defined in another package — including built-ins like `int` and `string`. If you want a method on `string`, define `type MyString string` in your package and add the method there.

## 3. Value receivers vs pointer receivers

The single most consequential daily choice:

```go
func (c Counter) Value() int { return c.n }    // value receiver — copies Counter
func (c *Counter) Incr()     { c.n++ }         // pointer receiver — operates on caller's Counter
```

When called on a value (`var c Counter; c.Incr()`), Go automatically takes the address: `(&c).Incr()`. When called on a pointer (`c := &Counter{}; c.Value()`), Go automatically dereferences: `(*c).Value()`. The syntax is symmetric; the semantics are not.

| Receiver type | Mutation visible to caller? | Receiver copied per call? | Method available on `T` value? | Method available on `*T`? |
|---|---|---|---|---|
| Value `T` | no | yes | yes | yes (auto-deref) |
| Pointer `*T` | yes | no (just the pointer) | yes (auto-address, only if addressable) | yes |

**Use a pointer receiver when:**
- The method needs to mutate the receiver. (Without a pointer, you mutate a copy that's discarded when the method returns.)
- The struct is large enough that copying it is wasteful. The conventional cutoff is "more than a couple of machine words," but profiling beats guessing.
- Any other method on the same type uses a pointer receiver. (See §5.)
- The type contains a `sync.Mutex` or similar. Copying a `sync.Mutex` is a bug — the lock state would not be shared, and `go vet` flags this.

**Use a value receiver when:**
- The receiver is small (`time.Time`, a `Point`, a small enum-like type) and immutable.
- You want callers to be able to use the type as a map key or compare it with `==`. Pointer-receiver-only methods don't prevent this, but mixing the two creates surprises.

## 4. Method sets and which interfaces a type satisfies

The **method set** of a type determines which interfaces it satisfies. The rule is precise:

| Type | Method set |
|---|---|
| `T` | All methods declared with receiver type `T` |
| `*T` | All methods declared with receiver type `T` **or** `*T` |

That is: a pointer's method set is a **superset** of its value's method set.

Concrete consequence:

```go
type Counter struct{ n int }
func (c *Counter) Incr() { c.n++ }   // pointer receiver

type Incrementer interface{ Incr() }

var c Counter
var i Incrementer = c    // COMPILE ERROR: Counter does not satisfy Incrementer
                         //   (Incr has pointer receiver)
var i Incrementer = &c   // OK
```

The asymmetry exists because not every value is **addressable**. Map elements, function return values, and interface values are not addressable, so the compiler cannot synthesize a pointer to call a pointer-receiver method. To keep the rules predictable, the language says: pointer-receiver methods are in `*T`'s method set only.

This trips people up most often when:
- Storing a value in a map and trying to call a pointer-receiver method on it (`m["key"].Incr()` is a compile error).
- Putting a value in an interface variable and finding the interface assignment fails because half the methods are pointer-receivers.

The fix is almost always to use `*T` everywhere instead of `T`. See §5.

## 5. The consistency rule

Within a given type, **all methods should use the same receiver style**. Mixing them is allowed but discouraged.

```go
// BAD — mixes receivers
type Buffer struct { /* ... */ }
func (b Buffer) Len() int    { /* ... */ }
func (b *Buffer) Write(p []byte) (int, error) { /* ... */ }

// GOOD — pointer receivers throughout
type Buffer struct { /* ... */ }
func (b *Buffer) Len() int    { /* ... */ }
func (b *Buffer) Write(p []byte) (int, error) { /* ... */ }
```

Why mixing is bad:
- The method set of `Buffer` (value) and `*Buffer` (pointer) differ. `Buffer` has only `Len`; `*Buffer` has both `Len` and `Write`. Code that holds a `Buffer` value and tries to use it as an `io.Writer` fails; code that holds `*Buffer` works. The asymmetry confuses readers and breaks interface assignments unpredictably.
- `go vet` and `gofmt` will not warn — this is a discipline issue.

The standard library follows this consistently. `*bytes.Buffer`, `*sync.WaitGroup`, `*http.Request` — all-pointer. `time.Time`, `time.Duration` — all-value. The decision is per-type, made when the type is designed.

A useful rule of thumb from the [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments#receiver-type):

> If in doubt, use a pointer receiver.

## 6. Escape analysis: stack vs heap

Go does not have a manual `new` / `delete` — every value is allocated automatically, and the garbage collector reclaims it. But the compiler still decides **where** to allocate: stack (cheap, freed when the function returns) or heap (managed by the GC).

The rules of thumb:
- If the compiler can prove a value does not outlive the function frame, it allocates on the stack. Stack allocation is essentially free.
- If the value's address is taken and the address can outlive the frame (returned, stored in a global, captured by a closure that escapes), the value **escapes** to the heap.

You can ask the compiler to print its decisions:

```bash
go build -gcflags='-m' ./...
```

Output looks like:

```text
./main.go:14:9: &x escapes to heap
./main.go:14:6: moved to heap: x
./main.go:20:13: ... argument does not escape
```

What causes a value to escape:

| Pattern | Escapes? |
|---|---|
| Take address, use only inside function | no |
| Take address, return the pointer | yes — must outlive the frame |
| Pass by value to another function | typically no |
| Pass by pointer to another function (depends on what callee does) | maybe — interprocedural analysis |
| Store into an interface variable | usually yes — interface holds a pointer |
| Captured by a closure that's returned or stored | yes |
| Stored in a slice (depends on context) | usually yes for the element data |

**You usually don't need to think about this.** The language is designed so escape analysis is correct without your input. But when profiling shows allocation pressure (high `gc` CPU time, large `alloc_space` in pprof), reading `-gcflags='-m'` output is how you find what's escaping and why.

The classic example: returning a pointer to a local variable.

```go
// Doesn't compile in C; does in Go — at the cost of a heap allocation.
func newCounter() *Counter {
    c := Counter{n: 0}
    return &c    // c escapes to heap
}
```

Whether this is "bad" depends entirely on context. For a constructor called once, the allocation is invisible. For a function called millions of times per second in a hot loop, it might be the bottleneck.

## 7. Nil pointer dereferences

Dereferencing a nil pointer panics:

```go
var p *Counter
p.Incr()   // panic: runtime error: invalid memory address or nil pointer dereference
```

This is the "NullPointerException" of Go. It is the runtime panic you will see most often.

Pointer receivers are still callable on a nil pointer **if the method does not dereference the receiver**:

```go
type List struct{ head *Node }

func (l *List) IsEmpty() bool {
    return l == nil || l.head == nil   // works even if l is nil
}

var l *List
l.IsEmpty()  // true — does not panic
```

This pattern is occasionally useful for nil-safe convenience methods, but be deliberate about it. The reader needs to know that a nil receiver is a supported case.

The flip side: typed-nil interface values (covered in [Interfaces & Structural Typing](05-interfaces-and-structural-typing.md), §7) can produce non-nil interface values that hold a nil pointer. Calling methods on those gets you back to dereference territory — and that's the bug pattern that's hardest to spot at the call site.

## Related

- [Types, Zero Values & Composite Literals](03-types-zero-values-composites.md) — when to take addresses, struct embedding
- [Interfaces & Structural Typing](05-interfaces-and-structural-typing.md) — how method sets determine interface satisfaction
- [Slices, Arrays & Maps](04-slices-arrays-maps.md) — addressability rules for indexed expressions
- [Errors as Values](06-errors-as-values.md) — pointer-receiver `Error()` methods and the typed-nil trap
- [Operating Systems — Virtual Memory & Paging](../../operating-systems/fundamentals/02-virtual-memory-and-paging.md) — what "the heap" actually is

## References

- The Go Programming Language Specification, "Method declarations" — https://go.dev/ref/spec#Method_declarations
- The Go Programming Language Specification, "Method sets" — https://go.dev/ref/spec#Method_sets
- The Go Programming Language Specification, "Address operators" — https://go.dev/ref/spec#Address_operators
- Go Code Review Comments, "Receiver Type" — https://go.dev/wiki/CodeReviewComments#receiver-type
- Effective Go, "Pointers vs. Values" — https://go.dev/doc/effective_go#pointers_vs_values
- Go FAQ, "Should I define methods on values or pointers?" — https://go.dev/doc/faq#methods_on_values_or_pointers
- "Allocation efficiency in high-performance Go services" (Segment) — https://segment.com/blog/allocation-efficiency-in-high-performance-go-services/ (a real-world tour through reading `-gcflags=-m` output)
- `go vet` `copylocks` analyzer — https://pkg.go.dev/cmd/vet — flags copying of `sync.Mutex` and similar lock types
