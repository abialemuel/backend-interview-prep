# Interfaces and Errors

## Interfaces

### Implicit satisfaction (structural / "duck typing")

Go interfaces are **implicit**: a type satisfies an interface by implementing its methods; no `implements` keyword, no inheritance declaration. This is sometimes called structural typing or compile-time duck typing.

```go
type Reader interface{ Read(p []byte) (int, error) }

type FileReader struct{}
func (FileReader) Read(p []byte) (int, error) { return 0, nil }

var r Reader = FileReader{} // compiles implicitly
```

The compiler checks the assignment. You can assert it statically at compile time with the canonical:

```go
var _ Reader = (*FileReader)(nil)  // verify implementation at compile time
```

Why implicit? It lets you define interfaces in the package that **consumes** them, not in the package that produces concrete types. This is the foundation of `io.Reader`/`Writer` — defined once in `io`, satisfied by dozens of unrelated packages.

### Interface composition

Compose interfaces from other interfaces by embedding:

```go
type ReadWriter interface {
    Reader
    Writer
}
```

This is purely method-set composition; you can also mix concrete methods:

```go
type ReadWriteCloser interface {
    Read([]byte) (int, error)
    Writer
    Closer
}
```

### The empty interface `any` and why prefer typed interfaces

`any` is an alias for `interface{}` introduced in Go 1.18:

```go
func Print(v any) { fmt.Println(v) }
```

`any` carries **no compile-time type information you can use** — you must type-assert or reflect to use the value. Prefer:

- Specific small interfaces (`Reader`, `Stringer`) over `any`.
- Generics (`func Print[T any](v T)`) when you don't need dynamic dispatch — keeps the concrete type available.
- `any` is fine for truly generic containers like `map[string]any` in JSON unmarshaling, but production code usually models with concrete structs.

### Small interfaces — `io.Reader` and friends

The Go proverb is **"The bigger the interface, the weaker the abstraction."** The standard library follows it fanatically:

| Interface | Methods | Purpose |
|-----------|---------|---------|
| `io.Reader` | `Read(p) (n int, err error)` | byte source |
| `io.Writer` | `Write(p) (n int, err error)` | byte sink |
| `io.Closer` | `Close() error` | resource release |
| `fmt.Stringer` | `String() string` | string form |
| `error` | `Error() string` | error value |
| `sort.Interface` | 3 methods | sortable collection |

Small interfaces encourage composition. Most user-defined interfaces should have 1–3 methods; if you find yourself with 10, split it or use a struct.

### Type assertions

```go
var i any = "hi"
s, ok := i.(string) // safe form; ok=false if i is nil or not a string
s2 := i.(string)     // panics if assertion fails
```

A type assertion can also assert against an interface type:

```go
if r, ok := v.(io.Reader); ok { r.Read(buf) }
```

### Type switches

```go
func describe(i any) {
    switch v := i.(type) {
    case int:    fmt.Println("int", v)
    case string: fmt.Println("string", v)
    case nil:    fmt.Println("nil")
    default:     fmt.Printf("other %T\n", v)
    }
}
```

Inside a `case`, `v` has the static type of that case (no `.(type)` casts needed).

### The nil-interface vs interface-with-nil-value gotcha

This is **the** classic interface trap and a common interview question.

```go
type MyError struct{}
func (*MyError) Error() string { return "boom" }

func doWork() error {
    var e *MyError = nil
    return e      // returns a non-nil error!
}

err := doWork()
fmt.Println(err == nil) // false
```

Why? An interface value is internally a (type, value) pair. It is `nil` only when **both** the type and the value are nil. Here the type is `*MyError` and the value pointer is nil — so the interface is non-nil but its underlying value is nil. The fix is to return an explicit `nil`:

```go
func doWorkGood() error {
    if cond {
        return &MyError{}
    }
    return nil
}
```

Rule of thumb: **never store a typed-nil pointer in an interface unless you understand the consequences.** When forwarding error-like values across boundaries, return concrete `nil` rather than a typed nil.

## Generics

### Type parameters

From Go 1.18 on, you can write array/slice/map/function/type declarations with type parameters:

```go
func Map[K, V, U any](m map[K]V, f func(V) U) map[K]U {
    out := make(map[K]U, len(m))
    for k, v := range m { out[k] = f(v) }
    return out
}

type Number interface{ int | int64 | float64 }
func Sum[T Number](xs []T) T {
    var s T
    for _, x := range xs { s += x }
    return s
}
```

Methods cannot introduce their own type parameters — only the receiver's are available. Under the hood, generics compile via **GC-shape stenciling**: one code instantiation per "shape" (e.g., all pointer types share one) plus a runtime dictionary of type metadata. That's a deliberate middle ground between C++-style full monomorphization (fastest code, slow compiles, binary bloat) and Java-style erasure (boxing everywhere) — a good trade-off to cite when asked "are Go generics zero-cost?" (answer: mostly, but dictionary indirection can inhibit inlining in hot paths; measure). Since Go 1.26, a generic type may refer to itself in its own type-parameter list (`type Adder[A Adder[A]] interface { Add(A) A }`), enabling recursive constraints that were previously rejected.

### Constraints

A constraint is just an interface. From 1.18:

- `any` (= `interface{}`) — no constraints.
- `comparable` — supports `==` and `!=` (allows interfaces, but be careful, see below).

Your own constraints may list **type sets** (since 1.18) using `|`:

```go
type Ordered interface {
    Integer | Float | ~string
}
```

The `~` prefix means "and any type whose underlying type is `string`" — i.e., named string types too.

The standard constraints live in `golang.org/x/exp/constraints` and (more permanently) in the `cmp` package (since Go 1.21): `cmp.Ordered`.

### Comparable

```go
func Contains[T comparable](xs []T, target T) bool {
    for _, x := range xs { if x == target { return true } }
    return false
}
```

`comparable` covers primitives, pointers, channels, arrays of comparable, and structs containing only comparable fields. It **does not** cover slices, maps, or functions. Since Go 1.20, ordinary interface types *do* satisfy `comparable` as a constraint — but comparing two interface values whose dynamic types aren't comparable still panics at runtime, so the compile-time guarantee is weaker for interfaces. Mention this nuance and you separate yourself from most candidates.

### Custom constraints and type sets

```go
type Addable interface { ~int | ~int64 | ~float64 }

// embed constraints in a type set
type Number interface {
    Addable
    ~int | ~float32
}
```

You can put any interface method list **or** a union of types in an interface; methods and union types can't mix (Go intentionally disallows it). To pass values of type-parameter types around, you can keep methods in `any`-like interfaces and union types separately.

### Generic type aliases (Go 1.24)

```go
type Set[T comparable] = map[T]struct{}
```

A generic *type alias* (`=`), allowed since Go 1.24, lets you reuse generic types across packages without re-declaring them.

### When to use generics vs interfaces

| Need | Use |
|------|-----|
| Same algorithm over multiple concrete numeric types | Generics |
| Container with element type stub (`Set[T comparable]`) | Generics |
| Polymorphic dispatch on behavior (e.g. `io.Reader`) | Interfaces |
| Code that must allow users to plug new types via methods | Interfaces |
| Performance-critical loops with type-specific ops | Generics (avoids boxing) |

Don't reach for generics reflexively; small generic helpers are common, but huge generic hierarchies are unidiomatic. The Go team's advice: start concrete, generalize when you see the third duplication.

## Error handling

### Errors as values (no exceptions)

Go represents failure as a returned `error` value. Code checks it:

```go
if err := doSomething(); err != nil {
    return fmt.Errorf("doSomething: %w", err)
}
```

There are no try/catch, no checked exceptions, no signal-handler style control flow. Functions that may fail return `(T, error)`, and `error` is itself an interface — so you can wrap, classify, and inspect errors at runtime.

Why not exceptions? Because exceptions split control flow into two channels, harder to reason about in concurrent code and harder to see when something can fail — explicit values are visible in the signature.

### `errors.New` and `fmt.Errorf`

```go
import "errors"

var ErrNotFound = errors.New("not found")

err := fmt.Errorf("user %q: %w", name, ErrNotFound)
```

`%w` (since Go 1.13) is the **wrap verb**: it embeds `err` so that `errors.Is` and `errors.As` can traverse the chain. `%v` formats an error textually without preserving tree structure.

### `errors.Is` and `errors.As`

```go
if errors.Is(err, ErrNotFound) { /* match sentinel */ }

var pathErr *fs.PathError
if errors.As(err, &pathErr) { /* extract concrete error type */ }
```

- `Is` walks the wrap chain comparing identity via `==`, calling each error's optional `Is(target error) bool` if defined (custom equality).
- `As` walks the chain finding the **first** error assignable to a target pointer; useful for typed error information (`*fs.PathError`, `*url.Error`, custom error types).

### `errors.AsType` (Go 1.26)

Go 1.26 added a generic, type-safe form of `As` — no pointer-to-pointer dance, no reflection cost:

```go
if pathErr, ok := errors.AsType[*fs.PathError](err); ok {
    log.Printf("failed on %s", pathErr.Path)
}
```

Prefer it in new code; know `errors.As` for the existing codebase and for interviews with older-version constraints.

### Sentinel errors

A sentinel is a package-level exported error value used as a sentinel; users compare with `errors.Is`:

```go
var ErrInvalid = errors.New("invalid input")
err := someFunc()
if errors.Is(err, ErrInvalid) { ... }
```

Pros: simple, decoupled. Cons: leaky error identity — many packages overuse sentinels; consider wrapped custom types so you can extend semantics without churning callers.

### Custom error types

```go
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Msg)
}

func (e *ValidationError) Is(target error) bool { /* optional equality */ }
func (e *ValidationError) Unwrap() error { /* optional parent */ }
```

Code that matches types uses `errors.As`. Make `Error()` informative; include the wrapped error in the output (or rely on `errors.Is/As` for traversal, depending on what you want printed).

### `errors.Join`

From Go 1.20:

```go
err := errors.Join(err1, err2)
if err != nil { /* handle composite */ }
```

Joins multiple errors into one whose `errors.Is` walks each, `errors.As` searches across all. Useful when collecting validation errors.

### Panics and `recover`

`panic` aborts the current goroutine with a runtime error; it unwinds defers. Recover only catches panics **in the deferred function of the same goroutine**.

```go
func safe(f func()) (err any) {
    defer func() {
        if r := recover(); r != nil {
            err = r
        }
    }()
    f()
    return nil
}
```

Use `recover` for **library boundaries** (an HTTP handler turning a panic into a 500) or in a long-running goroutine that should never bring down the service. **Don't** use panics as a normal control-flow feature — the standard idiom is errors-as-values; panics are reserved for genuinely unrecoverable situations: programmer bugs, invariants violated, "impossible" states.

### Error wrapping best practices

- Wrap at the call site to add context: `fmt.Errorf("foo %d: %w", id, err)`.
- Always include who/where: function context or "foo.bar: %w".
- Wrap **once per layer**; re-wrapping the same error repeatedly at one layer adds chain depth without information.
- Wrapping with `%w` makes the wrapped error part of your **API contract** — callers can now depend on `errors.Is(err, sql.ErrNoRows)` through your layer. If you don't want to expose the underlying error, wrap with `%v` (or translate to your own sentinel) deliberately.
- For library code, prefer **opaque** `errors.New("...")` instead of exposing internal sentinels; document only what's part of your API contract.
- Use `errors.Is`/`errors.As` to classify; don't switch on `err.Error()` strings.
- Type assertions like `err.(*MyError)` are **rarely needed** and bypass unwrapping semantics; prefer `errors.As`.
- Document with `package errors` documentation; if you wrap when you forward, callers will see line-of-context unrolling.

### Iterating the chain

```go
for err != nil {
    log.Printf("cause: %v", err)
    err = errors.Unwrap(err)
}
```

`Unwrap` returns the wrapped error or nil. `errors.Is`/`As` use it internally.

### `httptest` and testing for errors

Use `httptest.NewRecorder` to assert HTTP handlers' error responses:

```go
func TestHandler_BadRequest(t *testing.T) {
    rec := httptest.NewRecorder()
    req := httptest.NewRequest("GET", "/?id=abc", nil)
    handler(rec, req)
    if rec.Code != http.StatusBadRequest {
        t.Fatalf("got %d, want %d", rec.Code, http.StatusBadRequest)
    }
}
```

If your handler logs errors, you can assert via `errors.Is` by intercepting the error type with `HandlerError{}` provided you structure handlers to return errors:

```go
type HandlerError struct{ Code int; Err error }
func (h HandlerError) Error() string { return h.Err.Error() }
func (h HandlerError) Unwrap() error { return h.Err }

// in tests
var he HandlerError
if errors.As(err, &he) && he.Code == http.StatusBadRequest { /* match */ }
```

## Structured logging with `log/slog`

`log/slog` (Go 1.21) is the standard structured logger; interviewers in 2026 expect it rather than `log.Printf` or a third-party logger by default.

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

slog.InfoContext(ctx, "payment processed", "order_id", id, "amount_cents", n)
logger.LogAttrs(ctx, slog.LevelInfo, "payment processed",
    slog.String("order_id", id), slog.Int("amount_cents", n)) // zero-alloc form
reqLog := logger.With("request_id", reqID)                    // pre-bound attributes
```

Points that score in an interview:

- The front-end API (`Logger`) is decoupled from **handlers** (`TextHandler`, `JSONHandler`, or custom) — you can route the same records anywhere; Go 1.26 added `slog.NewMultiHandler` to fan one record out to several handlers.
- `LogAttrs` avoids the `any`-boxing of the loose key-value form; use it in hot paths.
- Pass `ctx` (`InfoContext`, etc.) so handlers can attach trace/span IDs.
- Log an error **once**, at the layer that handles it; wrapping and returning *and* logging at every layer double-reports. Wrap for classification, log at the top.

### Pitfalls

- Comparing errors with `==`: works for sentinels but doesn't traverse wrapping; always use `errors.Is`.
- Returning `nil` from a `func() error` returns a non-nil interface if the underlying type is concrete-nil — see gotcha above.
- `fmt.Errorf("%v")` drops wrapping; the chain is gone; `errors.Is` won't match. Always use `%w` when you want to keep the chain.
- Wrapping inside hot loops can create many error allocations; for inner errors prefer pre-allocated sentinels.