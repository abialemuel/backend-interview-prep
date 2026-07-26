# Core Concepts

## Design philosophy

Go was designed at Google by Robert Griesemer, Rob Pike, and Ken Thompson to solve three pain points in large-scale server software: slow builds, unwieldy dependency graphs, and languages that were either too dynamic (Python) or too heavyweight (C++/Java). The pillars are:

- **Simplicity.** The spec is small (~50 pages). There is one way to format code (`gofmt`), one testing tool (`go test`), one way to do modules. New features are added reluctantly and only when the trade-off is clearly positive.
- **Fast compilation.** A Go program compiles to a single static binary; package imports form a DAG with no cycles, allowing parallel compilation of dependencies. A large Go service typically builds in seconds, not minutes.
- **Strong static typing** without ceremony. Types are checked at compile time; there is no implicit widening; union types are expressed via interfaces, not class hierarchies.
- **Composition over inheritance.** No classes, no inheritance. Structs embed other structs or interfaces to reuse behavior.
- **Batteries included.** The standard library ships with `net/http`, `crypto/*`, `database/sql`, `encoding/json`, `log/slog`, and more; for many services no third-party dependencies are needed.
- **Concurrency as a first-class citizen.** Goroutines and channels are baked into the language and runtime (covered in `02-concurrency.md`).

Trade-offs worth knowing: Go deliberately omits exceptions, festivals of syntactic sugar, and pattern matching; enum types are emulated with `iota` constants; generics arrived late (1.18) and are intentionally restricted compared to C++ templates or Rust traits.

## Program structure

Every Go program is made of packages. An executable begins in `package main`, with a `func main()` entry point. A package name usually matches the last path segment of its import path (`import "net/http"` => `package http`).

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    name := os.Args[0]
    fmt.Printf("running %s\n", name)
}
```

Unused imports and unused locals are **compile errors** (not warnings), which is one of the strongest signals of Go's bias toward minimal, correct code.

## Variables and short declarations

```go
var i int           // 0, zero value
var j = 7           // inferred int
k := 3              // short declaration, function-scope only
name, age := "ada", 36  // multiple assignment incl. new vars
_, ok := m["nope"]  // blank identifier discards a value
```

`:=` requires at least one new variable on the left side. It is only legal inside a function body, never at package level.

## Basic types

| Category | Types |
|----------|-------|
| Boolean | `bool` |
| Numeric | `int`, `int8/16/32/64`, `uint` variants, `uintptr`, `byte` (=`uint8`), `rune` (=`int32`), `float32`, `float64`, `complex64`, `complex128` |
| String | `string` |
| Composite | `array`, `struct`, `slice`, `map`, `function`, `channel`, `interface` |

`int` and `uint` are platform-dependent (32 or 64 bits) but at least 32. `byte` and `rune` are aliases, not distinct types — they convert without a cast.

## Zero values

Every type has a zero value, so uninitialized memory is still safe:

| Type | Zero |
|------|------|
| numeric | `0` |
| bool | `false` |
| string | `""` |
| pointer, slice, map, chan, func, interface | `nil` |
| array | each element's zero value |
| struct | each field's zero value |

There are no uninitialized variables. This is intentional: it removes a large class of UB that haunts C/C++.

## Constants

```go
const Pi = 3.14159
const (
    StatusOK = iota  // 0
    StatusCreated    // 1
    StatusNoContent  // 2
)
```

- Constants are **untyped** by default; they acquire a type only when the context requires it (`var f float64 = Pi`). Untyped constants have higher precision than any typed value: an untyped `float` constant or `int` constant can hold arbitrarily large numbers; arithmetic never overflows until the constant is used in a typed context.
- `iota` is a per-`const` block counter reset to 0; it increments per line, including across empty lines. It is the idiomatic way to write enums in Go.
- Go has no native `enum` type. The idiom is a named type + `iota` constants + an exhaustive `switch` (linters such as `exhaustive` enforce coverage). Interviewers may ask how to make invalid values unrepresentable — the honest answer is you can't fully; you validate at boundaries and keep the constant type unexported where possible.

## Structs

```go
type Point struct{ X, Y float64 }

p := Point{1, 2}
p.X = 10
pp := &Point{X: 3, Y: 4} // pointer literal
```

**Embedding** is Go's replacement for inheritance:

```go
type Logger struct{}
func (Logger) Log(s string) {}

type Server struct {
    Logger          // promoted field/method — "embedding"
    Addr string
}

s := Server{}
s.Log("hi")         // promoted; equivalent to s.Logger.Log("hi")
```

- Embedded field name is the type name; methods and fields of the embedded type are *promoted* to the outer type.
- Embedding an interface lets a struct forward the interface methods to a field set later (useful for testing — inject a mock).

## Pointers

```go
x := 5
p := &x        // *int
*p = 10        // dereference assignment
n := *p        // read
```

Go has pointers but no pointer arithmetic (which eliminates a huge class of UB). The `&` operator yields an address, `*` dereferences. You cannot take the address of a map element or of a constant.

Adjacent modern topics worth one sentence each in an interview: Go 1.24 added the **`weak` package** (weak pointers that don't keep their referent alive — used for canonicalization and caches, e.g. by the `unique` package) and **`runtime.AddCleanup`**, the flexible successor to `runtime.SetFinalizer`.

### When to use pointers vs values

A common interview question.

**Use value receivers / value parameters when:**
- The type is small and cheap to copy (ints, small structs).
- The semantics are value-like: time.Time, time.Duration, geometry points.
- You want to avoid escape to the heap and keep allocation count zero.

**Use pointers when:**
- The receiver is large (copying it on every method call is wasteful).
- The method must mutate the receiver.
- The struct contains a `sync.Mutex` or similar — copying a mutex is a compile error in `go vet`; you must pass by pointer to share it.
- A function returns a new instance and the caller needs to share identity.
- You have a `nil`-able semantics where a `*T` can be `nil` but a `T` cannot (except for slice/map/chan/func/interface which are reference-ish).

**Rule of thumb for receivers**: be consistent. Don't mix value and pointer receivers on the same type; prefer pointer receivers when in doubt, because value receivers on a type with pointer-receiver methods make it impossible to satisfy an interface uniformly.

## Functions

```go
func add(a, b int) int        { return a + b }       // one return
func divmod(a, b int) (int, int) { return a/b, a%b } // multiple returns
func half(ok bool) (n int, err error) {               // named returns
    if !ok { return 0, fmt.Errorf("bad") }
    n = 10
    return // "naked" return uses named returns
}
func sum(nums ...int) int {                            // variadic
    total := 0
    for _, n := range nums { total += n }
    return total
}
sum(1, 2, 3)
nums := []int{1, 2, 3}; sum(nums...)  // spread a slice
```

Functions are first-class values:

```go
var f func(int) int = square
g := func(x int) int { return x * x }
```

Closures capture by reference, so be careful across goroutines and across iterations of pre-1.22 `for` loops (the loop-var bug).

## Control flow

### `if`

```go
if x := compute(); x > 0 {       // init; x scoped to if/else
    use(x)
} else if x == 0 {
    handleZero()
} else {
    handleNeg()
}
```

No parentheses around conditions; braces are mandatory.

### `for` (no `while`)

Go has one loop construct. Since Go 1.22, each iteration of the loop variable is a fresh variable, eliminating the historical closure-capture bug.

```go
for i := 0; i < 10; i++ { }   // C-style
for cond { }                  // while(cond)
for { break }                 // while(true)
for i, v := range s { }       // slice/array index+value
for k, v := range m { }       // map key+value
for i := range n { }          // range over int since 1.22 (0..n-1)
for v := range ch { }         // receive from channel
for v := range seq { }        // range over func — iterators, since 1.23
```

### Iterators (`range` over func, Go 1.23)

Since Go 1.23, `range` accepts functions matching `iter.Seq[V]` (`func(yield func(V) bool)`) and `iter.Seq2[K, V]`. The `iter` package defines the types; `slices` and `maps` grew iterator-returning helpers (`slices.Values`, `slices.All`, `maps.Keys`, `maps.Values`).

```go
for k := range maps.Keys(m) { }       // iter.Seq[K]
for i, v := range slices.All(s) { }   // iter.Seq2[int, V]

// Writing one: yield returns false when the consumer breaks out.
func Filter[V any](seq iter.Seq[V], keep func(V) bool) iter.Seq[V] {
    return func(yield func(V) bool) {
        for v := range seq {
            if keep(v) && !yield(v) {
                return
            }
        }
    }
}
```

Why interviewers ask: iterators enable lazy pipelines without allocating intermediate slices, and they standardize a single iteration shape across containers. Know that `break` in the consumer makes `yield` return `false`, and that `iter.Pull` converts a push iterator into a pull-style `next`/`stop` pair when you need to interleave two sequences.

### `switch`

```go
switch day {
case "Mon", "Tue": work()
case "Wed": meet()
default: relax()
}

switch {                       // tagless = if/else if
case x < 0: neg()
case x == 0: zero()
default: pos()
}

switch v := i.(type) {         // type switch
case int: fmt.Println("int")
case string: fmt.Println("string")
default: fmt.Printf("unknown %T\n", v)
}
```

Switches **do not fall through** by default; use the explicit `fallthrough` keyword when needed (rarely needed). Unlike C, `case`s can list multiple values comma-separated.

## Defer

`defer` schedules a function call to run when the surrounding function returns, in **LIFO order**. Arguments are evaluated at the `defer` statement, not at execution time.

```go
func copyFile(dst, src string) (err error) {
    in, err := os.Open(src)
    if err != nil { return err }
    defer in.Close()    // runs even if we panic

    out, err := os.Create(dst)
    if err != nil { return err }
    defer out.Close()

    _, err = io.Copy(out, in)
    return
}
```

**LIFO ordering**: the last deferred call runs first.

**Closure pitfall**:

```go
for i := 0; i < 3; i++ {
    defer func() { fmt.Println(i) }() // pre-1.22 printed 3,3,3; 1.22+ prints 2,1,0
}
```

Wait — carefully: the captured `i`. Even from 1.22 (per-iteration variable), the deferred closure is evaluated at call time after the loop. Because `i` is per-iteration now, each closure captures its own copy: prints 2,1,0. Pre-1.22, with a shared `i`, it prints 3,3,3. **Passing the value as an argument avoids the ambiguity across all versions**:

```go
for i := 0; i < 3; i++ {
    defer func(i int) { fmt.Println(i) }(i) // always 2,1,0
}
```

**Open-coded defer** (Go 1.14+): the compiler may inline up to eight `defer` statements in a function for negligible cost (no deferrecord allocation). This makes `defer` cheap enough to use in hot paths like mutex unlock; defer is no longer a perf reason to avoid it in non-tiny functions.

## Slices internals

A slice is a 3-word header (struct):

```go
type slice struct {
    data *T   // pointer to backing array
    len  int
    cap  int
}
```

The **backing array** is the actual contiguous memory; the slice header is a view over it. Copying a slice header (assignment or pass-by-value) copies only those 3 words — they share the underlying array until a `make`/`append`/`copy` creates a new one.

```go
s := []int{1, 2, 3}     // len 3, cap 3
s = append(s, 4)        // cap may grow to 6 (roughly doubles while small, then ~1.25x)
sub := s[1:3]           // len 2, cap depends; shares the same backing array
sub = append(sub, 99)   // overwrites s[2] -> 99, since cap was available!
```

**Slicing pitfall**: a slice referencing a huge backing array keeps it alive. To return a small slice, copy it:

```go
func firstThree(s []byte) []byte {
    return append([]byte(nil), s[:3]...) // fresh backing array
}
```

**Growth policy**: append roughly doubles capacity for small slices (below a threshold of 256 elements in the current runtime), then transitions smoothly toward ~1.25x growth. The exact numbers aren't part of the spec; what matters is that there is an amortized-growth heuristic and preallocation is preferable in known-size code:

```go
s := make([]int, 0, n) // preallocate cap when size is known
```

### Maps

```go
m := map[string]int{"a": 1}
v := m["a"]
v, ok := m["a"] // comma-ok to detect missing
delete(m, "a")
m = make(map[string]int) // empty
var nilMap map[string]int
// nilMap["a"] = 1 // panics: writing to nil map
_ = nilMap["a"] // returns zero value; reads from nil map are safe
for k, v := range m { ... } // iteration order is randomized
m2 := map[[2]string]int{} // any comparable type may be a key
```

- Since Go 1.24 the built-in map is a **Swiss-table** design (open addressing with SIMD-friendly group probing), replacing the old bucket+overflow-list layout — faster lookups/inserts and lower memory. The API, semantics, and randomized iteration order are unchanged; only the internals moved.
- Iteration order is **randomized** intentionally (prevents reliance on order). If you need stable order, sort the keys with `slices.Sorted(maps.Keys(m))`.
- Maps are **not concurrency-safe**. Concurrent reads + writes, or concurrent writes, race; run with `-race` and you'll see. Wrap with `sync.RWMutex` or `sync.Map` (the latter is fine for caches with few writes, many reads of disjoint keys).
- `delete` during iteration is valid; adding keys during iteration is not specified — don't.

## Strings, runes, bytes

- `string` is an immutable byte sequence (UTF-8 by convention, not enforced).
- `[]byte` is the mutable equivalent; converting `string ↔ []byte` copies the bytes (cost O(n)). Starting from Go 1.20 the compiler can sometimes elide the copy when the conversion is in a `m[string(...)]` indexing or similar limited cases — but assume it copies.
- `rune` is an `int32` representing a Unicode code point. Iterating a string with `for i, r := range s` yields byte-offset + rune pairs; iterating a `[]byte` yields byte indexes + bytes (uint8).

```go
s := "Héllo"
b := []byte(s)        // copy
rs := []rune(s)       // one rune per Unicode code point
for i, r := range s { // i is byte offset, r is rune
    fmt.Printf("%d %U\n", i, r)
}
```

Counting "characters": `len([]rune(s))` (or `utf8.RuneCountInString(s)`) — `len(s)` is the byte length, not character count. For high-performance string building use `strings.Builder`.

## Type declarations

```go
type Celsius float64
type MySlice []int

// type alias — interchangeable with the original (Go 1.9+)
type Text = string
```

A named type (`type Celsius float64`) is distinct from `float64`; you must convert explicitly: `Celsius(f)`. Type aliases (`type T = X`) are just renaming, used for refactoring visibility across packages.

## Packages and modules

A **module** is a versioned collection of packages declared in `go.mod`:

```
module github.com/you/svc

go 1.26

require (
    github.com/spf13/cobra v1.8.0
    golang.org/x/sync v0.10.0
)
```

- `go.mod` declares the module path, Go version, requires, replaces, excludes, retract. Since Go 1.21 the `go` directive is a **minimum** and a `toolchain` directive (also 1.21) may pin the exact toolchain (`toolchain go1.26.0`). Since Go 1.24 a `tool` directive tracks executable tool dependencies (`go get -tool ...`, run via `go tool <name>`), replacing the old blank-import `tools.go` hack. Go 1.25 added an `ignore` directive to exclude directories from package patterns.
- `go.sum` is a cryptographic hash ledger of expected contents of every dependency ever downloaded; checked on build.
- **Versioning**: Go modules use semantic import versioning. For v2+ the major version is in the path: `github.com/you/lib/v2`. v0 and v1 omit a suffix.
- **Minimum version selection (MVS)**: Go selects the maximum of required versions across the module graph, not the lowest-common-denominator or "newest satisfying." Predictable, reproducible builds.

Commands:

```sh
go mod init github.com/you/svc
go mod tidy
go mod download
go mod why github.com/pkg/errors
go get github.com/spf13/cobra@v1.8.0
go get github.com/spf13/cobra@latest
go get -u       # update direct deps
go mod edit -go=1.26
```

### Workspaces (`go.work`)

Introduced in Go 1.18, workspaces let you edit multiple modules locally without editing `go.mod` to add `replace` lines:

```sh
go work init ./a ./b
go work use ./c
```

`go.work` records modules to be resolved from local directories instead of fetching them. The `GOWORK=off` env can disable it for a single build. Workspaces are intended for local development only and are not committed in many projects.

## Build tags (build constraints)

```go
//go:build linux && amd64
// +build linux amd64 // legacy form, kept for older toolchains
package mypkg
```

- Apply to whole files (file-level).
- The legacy `// +build` form must precede the package clause with a blank line; the new `//go:build` form uses boolean expressions and a single line.
- Also implicit constraints come from filename suffixes like `foo_linux.go`, `foo_amd64.go` — they're auto-applied.

## Tooling

| Command | Purpose |
|---------|---------|
| `go build` | Compile packages into a binary or check builds |
| `go run` | Compile + run a one-shot |
| `go test` | Run tests/benchmarks/fuzz targets in packages |
| `go vet` | Static checks for suspicious constructs |
| `go fmt` / `gofmt` | Canonical formatting |
| `go mod` | Module operations |
| `go generate` | Run `//go:generate` directives (e.g., generate mocks) |
| `go doc` | Read package docs |
| `go env` | Inspect environment variables (`GOOS`, `GOARCH`, `GOPATH`, `GOMODCACHE`, etc.) |
| `go work` | Workspace commands |
| `go tool` | Run bundled tools (`pprof`, `trace`, `cover`, `compile`, etc.) |
| `go fix` | Apply automated "modernizer" rewrites to newer idioms/APIs (fully revamped in 1.26 on the vet analysis framework) |

Standard linters beyond `go vet` (e.g. `staticcheck`, `golangci-lint`) are common in production but not shipped with the toolchain.

## Pitfalls to remember

- Modifying an element through a map value won't compile: `m["x"].Field = 1` is illegal because you can't take the address of a map element. Read-modify-write: copy out, edit, write back, or use `*Point` values.
- `len` of a nil slice is 0; iterating a nil slice or map is a no-op (safe).
- Comparing two slices or maps with `==` is a compile error (except `[]byte` vs `string` via special case). Use `slices.Equal` / `maps.Equal`.
- Pointer-to-large-array param: `func f([1<<20]byte)` copies the whole array; use `*[1<<20]byte` or `[]byte`.