---
title: "Slices, Arrays & Maps"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, slices, arrays, maps, gotchas]
---

# Slices, Arrays & Maps

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `slices` `arrays` `maps` `gotchas`

---

## Table of Contents

1. [Arrays: fixed-size, value-typed, rarely used directly](#1-arrays-fixed-size-value-typed-rarely-used-directly)
2. [Slices: descriptor + backing array](#2-slices-descriptor--backing-array)
3. [`len`, `cap`, and how `append` reallocates](#3-len-cap-and-how-append-reallocates)
4. [The slice aliasing bug](#4-the-slice-aliasing-bug)
5. [Slice tricks and gotchas](#5-slice-tricks-and-gotchas)
6. [Maps: hash tables, unordered, nil-write panics](#6-maps-hash-tables-unordered-nil-write-panics)
7. [Preallocation: when it matters](#7-preallocation-when-it-matters)

## Summary

Slices and maps are where Go programmers most often get bitten — not because they're hard, but because they look like Java's `ArrayList` and `HashMap` and behave subtly differently. A slice is a three-word **descriptor** (pointer, length, capacity) over a backing array; copying a slice copies the descriptor but shares the array. `append` may or may not reallocate, and whether it does depends on capacity in a way that produces non-deterministic-looking aliasing bugs. Maps are hash tables with random iteration order, and writing to a `nil` map panics — a constraint that forces explicit initialization with `make`.

This is the single most bite-prone area for new Go programmers. Most senior Go bugs in production are slice aliasing, slice-of-slices retaining a giant backing array, or map writes during concurrent iteration.

## 1. Arrays: fixed-size, value-typed, rarely used directly

An array's size is part of its type:

```go
var a [3]int          // [3]int — distinct from [4]int
b := [3]int{1, 2, 3}
c := [...]int{1, 2, 3}  // size inferred from literal: still [3]int
```

Arrays are **value types**. Assigning or passing an array copies it:

```go
a := [3]int{1, 2, 3}
b := a
b[0] = 99
fmt.Println(a)  // [1 2 3] — a unchanged
```

In practice, you almost never use arrays directly. They appear as:
- The **backing storage** for slices (which is what you actually work with).
- Fixed-size buffers in performance-sensitive code (e.g., `[16]byte` for a UUID, `[32]byte` for a SHA-256 digest).
- Map keys when the key needs to be a struct of fixed-size values.

If you find yourself reaching for an array in application code, you probably want a slice instead.

## 2. Slices: descriptor + backing array

A slice is a three-word value:

```text
struct {
    ptr *T   // pointer to the start of the visible window
    len int  // number of elements visible
    cap int  // number of elements available before reallocation
}
```

Visualize it:

```text
backing array:   [10][20][30][40][50][60][70][80]
                   ^                  ^
                   ptr                ptr+cap
                   |←——— len = 4 ———→|
                   |←——————— cap = 6 ——————→|
```

Three ways to create a slice:

```go
// 1. Literal
s := []int{1, 2, 3}

// 2. make
s := make([]int, 5)        // len=5, cap=5, all zero
s := make([]int, 0, 100)   // len=0, cap=100 — preallocated

// 3. Slicing an existing array or slice
arr := [5]int{1, 2, 3, 4, 5}
s := arr[1:4]              // len=3, cap=4 (cap goes to end of backing array)
```

**The descriptor is copied; the backing array is shared.** This is the single most important sentence to internalize about slices.

```go
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:3]      // s2 = [2 3], shares s1's backing array
s2[0] = 99
fmt.Println(s1)    // [1 99 3 4 5] — s1 was mutated through s2
```

## 3. `len`, `cap`, and how `append` reallocates

`len(s)` is how many elements you can index. `cap(s)` is how many elements fit in the backing array starting from `s[0]`.

`append` grows a slice. The semantics:

1. If `len(s) < cap(s)`, `append` writes the new element into the existing backing array and returns a slice with `len+1`.
2. If `len(s) == cap(s)`, `append` allocates a **new, larger** backing array, copies the old elements, writes the new element, and returns a slice pointing to the new array.

The growth strategy (current Go runtime, subject to change without notice):
- Up to a threshold (~256 elements at the time of writing), capacity roughly **doubles**.
- Past the threshold, growth slows — the runtime aims for amortized O(1) append while avoiding huge over-allocation.

The exact formula is in `runtime/slice.go` in the Go source tree. **Do not depend on it** — it's a runtime detail and has changed across versions.

```go
s := make([]int, 0, 4)
fmt.Println(len(s), cap(s))  // 0 4

s = append(s, 1, 2, 3, 4)
fmt.Println(len(s), cap(s))  // 4 4 — full

s = append(s, 5)
fmt.Println(len(s), cap(s))  // 5 8 — reallocated, capacity doubled
```

## 4. The slice aliasing bug

This is the canonical Go bug. It comes from the rule "the descriptor is copied; the backing array is shared," combined with the fact that `append` may or may not reallocate.

```go
func process(items []int) {
    items = append(items, 999)  // does this affect the caller?
    items[0] = -1                // does this affect the caller?
}

func main() {
    s := make([]int, 3, 10)   // len=3, cap=10 — room to append in place
    s[0], s[1], s[2] = 1, 2, 3
    process(s)
    fmt.Println(s)  // [-1 2 3] — s[0] mutated, but the 999 is invisible
                    // (process's local `items` got len=4 and saw the 999,
                    //  but the caller's `s` still has len=3)
}
```

The mutation to `s[0]` is visible because both descriptors point to the same backing array. The `999` is invisible because `append` returned a new descriptor with `len=4`, but only inside `process`'s local variable.

If `s` had been `make([]int, 3, 3)` instead, `append` would have reallocated, and the `s[0] = -1` mutation would have been to the **new** backing array — also invisible to the caller. The behavior of an identical-looking call depends on capacity that the caller may not be tracking.

The fix is to always **return** the appended slice and have the caller reassign:

```go
func process(items []int) []int {
    items = append(items, 999)
    items[0] = -1
    return items
}

s = process(s)
```

This is exactly why every standard library function that grows a slice has the signature `func(s []T, ...) []T`. Read the documentation for `append`:

> The append built-in function appends elements to the end of a slice. **If it has sufficient capacity, the destination is resliced to accommodate the new elements. If it does not, a new underlying array will be allocated.** Append returns the updated slice. **It is therefore necessary to store the result of append**, often in the variable holding the slice itself.
>
> — https://pkg.go.dev/builtin#append (emphasis added)

## 5. Slice tricks and gotchas

A condensed reference of the common operations. The full canonical list is the [Slice Tricks wiki page](https://github.com/golang/go/wiki/SliceTricks).

```go
// Copy a slice (independent backing array)
dst := make([]int, len(src))
copy(dst, src)
// or, since Go 1.21:
dst := slices.Clone(src)

// Delete element at index i (no order preservation — fast)
s[i] = s[len(s)-1]
s = s[:len(s)-1]

// Delete element at index i (preserve order)
s = append(s[:i], s[i+1:]...)
// or, since Go 1.21:
s = slices.Delete(s, i, i+1)

// Insert at index i
s = append(s[:i], append([]int{x}, s[i:]...)...)
// or, since Go 1.21:
s = slices.Insert(s, i, x)

// Reverse
slices.Reverse(s)  // since Go 1.21

// Filter in place (preserves underlying array)
n := 0
for _, x := range s {
    if keep(x) {
        s[n] = x
        n++
    }
}
s = s[:n]
```

**The "subslice retains the giant array" gotcha.** If you take a small slice of a huge slice and store it long-term, the huge backing array stays alive in memory because the descriptor still points to it.

```go
func firstLine(buf []byte) []byte {
    i := bytes.IndexByte(buf, '\n')
    return buf[:i]   // returned slice keeps `buf` alive in GC
}
```

If `buf` is 100 MB and the caller stores `firstLine(buf)` for hours, the 100 MB stays in memory. The fix when retaining a small subslice from a large source is to copy:

```go
func firstLine(buf []byte) []byte {
    i := bytes.IndexByte(buf, '\n')
    out := make([]byte, i)
    copy(out, buf[:i])
    return out
}
```

This was the [bug behind a real CockroachDB memory leak](https://github.com/cockroachdb/cockroach/issues/55216) and similar issues in many large Go services.

**The `nil` slice vs empty slice distinction.** A nil slice has `len == 0`, `cap == 0`, and `ptr == nil`. An empty slice has the same lengths but a non-nil pointer. They are mostly interchangeable — `len(nil_slice)` is 0, `range nil_slice` does nothing, `append(nil_slice, x)` works — but `encoding/json` serializes `nil` as `null` and `[]int{}` as `[]`. Use this if it matters; otherwise, treat them as the same.

## 6. Maps: hash tables, unordered, nil-write panics

A map is a hash table:

```go
m := map[string]int{"a": 1, "b": 2}
m["c"] = 3

v, ok := m["a"]    // comma-ok idiom: ok is false if key not present
v := m["missing"]  // returns zero value (0 for int) — no error

delete(m, "a")

for k, v := range m { /* ... */ }  // iteration order is randomized
```

Three rules that bite newcomers:

**1. The zero value of a map is `nil`, and writing to a nil map panics.**

```go
var m map[string]int
m["a"] = 1  // panic: assignment to entry in nil map
```

`make` (or a literal) is required to initialize:

```go
m := make(map[string]int)
m := map[string]int{}
m := make(map[string]int, 100)  // hint: expecting ~100 entries
```

This is the inverse of slices, where `append(nil, x)` works fine. The reasoning is in the [Go FAQ](https://go.dev/doc/faq#nil_maps): a nil map is a useful distinct state for "no map at all," and silently allocating on first write would mask programmer errors.

**2. Iteration order is randomized, intentionally.**

The Go runtime randomizes the starting bucket on every iteration of `range m`. This was [deliberately added in Go 1.0](https://go.dev/blog/maps) to discourage code that depends on iteration order — order is not stable across runs, across Go versions, or even across iterations within the same program.

If you need deterministic iteration, sort the keys:

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    use(k, m[k])
}
```

**3. Maps are not safe for concurrent use.**

Concurrent reads are fine. **Any concurrent access where at least one goroutine writes** is a data race, and the runtime will detect it (a fatal error, not a panic — you cannot `recover` from it):

```text
fatal error: concurrent map writes
```

For concurrent maps, use `sync.Map` (good for "write-once, read-many" or "disjoint key sets per goroutine") or a regular map guarded by `sync.RWMutex` (good for everything else). This is covered in Tier 2.

## 7. Preallocation: when it matters

When you know the final size of a slice or map, hint it:

```go
// Bad: slice grows incrementally, multiple reallocs
out := []User{}
for _, row := range rows {
    out = append(out, parseUser(row))
}

// Good: one allocation
out := make([]User, 0, len(rows))
for _, row := range rows {
    out = append(out, parseUser(row))
}
```

For maps:

```go
m := make(map[string]User, expectedSize)
```

The hint is **just** a hint — the runtime is free to allocate more or less. But a good hint avoids the doubling-and-copying that drives append-heavy hot loops.

When **not** to preallocate:
- The size is small (under ~16 elements). The overhead of the calculation costs more than the saved allocation.
- The size is unknown and a wild guess would be worse than letting `append` grow.

`go tool pprof` heap profiles will show you when this matters. Don't optimize without measuring.

## Related

- [Go Tour for TS + Java Developers](01-go-tour-for-ts-java-devs.md) — basic types and syntax
- [Types, Zero Values & Composite Literals](03-types-zero-values-composites.md) — value vs reference types
- [Pointers, Methods & Receivers](07-pointers-methods-receivers.md) — escape analysis and heap allocation
- [Operating Systems — Virtual Memory & Paging](../../operating-systems/fundamentals/02-virtual-memory-and-paging.md) — what "the GC retains the giant backing array" actually costs

## References

- The Go Programming Language Specification, "Slice types" — https://go.dev/ref/spec#Slice_types
- The Go Programming Language Specification, "Map types" — https://go.dev/ref/spec#Map_types
- The Go Programming Language Specification, "Appending to and copying slices" — https://go.dev/ref/spec#Appending_and_copying_slices
- "Go slices: usage and internals" — https://go.dev/blog/slices-intro
- "Arrays, slices (and strings): The mechanics of 'append'" — https://go.dev/blog/slices
- "Go maps in action" — https://go.dev/blog/maps
- Slice Tricks (community wiki) — https://github.com/golang/go/wiki/SliceTricks
- `append` documentation — https://pkg.go.dev/builtin#append
- Go FAQ, "Why are map operations not defined to be atomic?" — https://go.dev/doc/faq#atomic_maps
- `slices` standard library package (Go 1.21+) — https://pkg.go.dev/slices
