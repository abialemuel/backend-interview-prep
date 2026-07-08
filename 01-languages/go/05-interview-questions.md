# Interview Questions

## Easy

### Q1: Why don't slices in Go have their own backing array?
**Answer:** A slice is a 3-word header — a pointer to a backing array, plus `len` and `cap`. The backing array is shared: slicing (`s[a:b]`) creates a new header that points into the same memory. This is what makes slices cheap to pass around, but it also means two slices can refer to overlapping memory. The runtime grows the array as needed by allocating a new one and copying the contents. Knowing this explains most slice behavior: shared mutation, growth cost, and the "huge array kept alive by a small slice" pitfall.

### Q2: What's the difference between an array and a slice in Go?
**Answer:** An array (`[N]T`) has a fixed size that is part of its type — `[3]int` and `[4]int` are distinct types and cannot be assigned to one another. A slice (`[]T`) is a dynamically-sized view over an array, with run-time `len` and `cap`. Passing an array passes a fresh copy of its entire contents (potentially large); passing a slice passes only the 3-word header. Arrays are rarely used directly in Go; slices are the default workhorse.

### Q3: What are the zero values of the basic types in Go?
**Answer:** Numeric types default to `0`, `bool` to `false`, `string` to `""`, and all reference types (pointer, slice, map, channel, function, interface) to `nil`. Arrays and structs get each element/field's zero value recursively. There are no uninitialized variables, so reading a local that you forgot to assign yields a defined value rather than UB. This is why `var x int` is safe and `m[k]` on a missing key returns the zero value.

### Q4: Why does Go have `iota` and how does it work?
**Answer:** `iota` is a per-`const` block counter that resets to 0 for each `const (...)` group and increments by 1 for each subsequent line. It's the idiomatic way to model enums, bit flags, and unit-conversion constants:

```go
const (
    _  = iota
    KB = 1 << (10 * iota)
    MB
    GB
)
```

Untyped constants like `Pi` also have arbitrary precision until assigned to a typed variable, which prevents accidental overflow during constant arithmetic.

### Q5: Outline how Go module versioning works.
**Answer:** A module is versioned with semantic versioning and **semantic import versioning**: v0 and v1 share the bare path (`github.com/you/lib`), but v2+ includes the major version in the path (`github.com/you/lib/v2`). This means v2 is a distinct import from v1 — a program can depend on both simultaneously. Version selection uses **minimum version selection (MVS)**: the build selects the highest version any module in the graph requires, yielding reproducible builds without a lockfile.

### Q6: What are Go workspaces and when are they used?
**Answer:** A workspace (Go 1.18+, `go.work`) is a set of modules developed together: when building any one of them, the others are resolved from local directories instead of fetched from the proxy. It replaces the older `replace`-line workflow where you'd add `replace github.com/foo => ../foo` in each `go.mod`. Workspaces are local-developer tooling — they may be ignored in CI by setting `GOWORK=off`, and many teams don't commit `go.work`.

### Q7: Why does Go compile so fast compared to C++ or Rust?
**Answer:** Three deliberate design choices: (1) imports form a DAG with no cycles, so packages can be compiled independently and in parallel; (2) package exports are computed once and the resulting object file carries the export data, so importers don't re-parse dependencies; (3) the language has no template metaprogramming and a small surface, so the compiler doesn't perform heavy per-instantiation work. Generics added in 1.18 are implemented with shape-based GC stenciling rather than full monomorphization, keeping compile times bounded.

### Q8: What's the difference between `string`, `[]byte`, and `[]rune`?
**Answer:** `string` is an immutable UTF-8 byte slice; `len(s)` is the byte count, not character count. `[]byte` is the mutable counterpart and `string ↔ []byte` conversion copies bytes (O(n)). `[]rune` is `[]int32` — one element per Unicode code point, useful when you need to iterate over characters or count them. To avoid quadratic string building use `strings.Builder`; to count "characters" use `utf8.RuneCountInString(s)`. Indexing `s[i]` yields a `byte` (uint8), not a rune.

### Q9: What is `defer` and what is its execution order?
**Answer:** `defer` schedules a function call to run when the surrounding function returns, in **LIFO order** — the last deferred runs first. Arguments to the deferred call are evaluated at the `defer` statement, not at execution. Deferred calls run even on panic, so `defer mu.Unlock()` or `defer f.Close()` is the canonical way to ensure cleanup. Since Go 1.14 most `defer`s in functions with ≤8 defers are "open-coded" and cost effectively nothing, so performance is no longer a reason to avoid them.

### Q10: What is the cleaned-up loop variable behavior since Go 1.22?
**Answer:** Before Go 1.22, the `for i := range xs` loop variable was a single shared variable across all iterations — closures captured by reference would see the final value. Since 1.22, each iteration gets a fresh variable; closures capture per-iteration state and the historical bug disappears. This made many `i := i` lines obsolete. The same per-iteration semantics now apply to `range over int` (1.22), `range over func` (1.23/1.26), and to nested `for` loops.

## Medium

### Q11: Why did Go until 1.17 not have generics, and what does it have now?
**Answer:** The Go designers deliberately avoided generics for a decade, prioritizing simplicity and fast compilation over expressiveness — they wanted the language to remain easy to read and tool. They also weren't satisfied with earlier proposals; the 2018 draft (Contracts) was abandoned. In **Go 1.18 (2022)** they shipped type parameters with constraints, giving you `func Map[K comparable, V any]`. The constraint-system uses type sets instead of just method lists, and generics are designed to not slow the compiler (implemented with shape stenciling, not full monomorphization). Combined with the new `slices`/`maps` packages (1.21+), the standard library now lets you write reusable, type-safe data utilities.

### Q12: Explain `append()` semantics and slice growth.
**Answer:** `append(s, x)` returns a new slice. If `s` has spare capacity (cap > len), the new element is written in the existing backing array, `len` is incremented, and the same header is returned. If `s` is at capacity, the runtime allocates a new larger array, copies the existing elements plus `x`, and returns a header with new `cap`. The growth policy roughly doubles up to ~1024 elements and grows ~1.25× thereafter. Because the underlying array may be shared, the receiver of `append`'s return must always store it back: `s = append(s, x)`. Preallocation (`make([]T, 0, n)`) avoids the growth overhead in known-size code.

### Q13: Why can the result of `append` mutate an unrelated slice?
**Answer:** Because slicing shares the backing array. If `sub := big[5:8]` is created with cap > len, a subsequent `sub = append(sub, x)` that fits in the cap will write directly into `big`'s array — overwriting `big[8]`. To get an independent copy, do `sub := append([]T(nil), big[5:8]...)` or use `slices.Clone(big[5:8])` (since Go 1.21). This aliasing is the classic interview gotcha and the reason "always copy before returning a sub-slice" is a discipline.

### Q14: Why are maps unsafe for concurrent use, and what do you do about it?
**Answer:** A `map[K]V` is not internally synchronized; concurrent writes (or a write concurrent with a read) produce undefined behavior — silent corrupted values, in some runtime versions even hard crashes ("fatal error: concurrent map writes"). There's no `sync.Map` "upgrade" semantics for plain maps either. Mitigations: (1) wrap the map in `sync.RWMutex` yourself; (2) use `sync.Map` if your workload is "disjoint keys written by independent goroutines, otherwise read-mostly"; (3) channel-produce all writes from a single owner goroutine; (4) use `sync/atomic` per-entry if you need lock-free counters. Running `go test -race ./...` will catch map misuse in tests.

### Q15: Differentiate a nil interface from an interface holding a nil concrete value.
**Answer:** An interface value is internally a `(type, value)` pair. The interface is `nil` only when **both** are unset. If a function having return type `error` does `var p *MyError = nil; return p`, callers see a **non-nil interface** whose dynamic value is `nil`. Specifically:

```go
func f() error { var e *MyError = nil; return e }
err := f()
fmt.Println(err == nil) // false
```

The fix: only return a literal `nil` from the function. Equivalently, declare the result as a concrete `nil` only when you intend a real `nil` interface. This is a top-three Go interview gotcha.

### Q16: How does interface satisfaction work? How do you ensure it at compile time?
**Answer:** Interfaces are satisfied implicitly (structural typing): a type implements an interface if it has all of the interface's methods — no `implements` keyword. To prove a type satisfies an interface at compile time, you can declare:

```go
var _ io.Reader = (*MyReader)(nil)
```

If the methods are missing for `*MyReader`, the build fails at the line. This idiom is preferred in library packages to catch refactor breakage immediately. The compiler never auto-derives interfaces; the programmer declares them wherever they're consumed.

### Q17: What's the difference between a goroutine and a thread?
**Answer:** An OS thread is heavyweight: ~1–8 MB stack, scheduled by the kernel, context switching is expensive, and you typically have hundreds at most. A goroutine is runtime-managed: ~2 KB initial stack that grows, scheduled cooperatively-preemptively by the Go runtime in user space, and you can run hundreds of thousands. Goroutines are multiplexed over a smaller number of OS threads via the **GMP** model — the P (`GOMAXPROCS`) limit determines how many run simultaneously, but blocking goes onto the OS thread's M while the P hops to another M.

### Q18: Explain the GMP scheduler.
**Answer:** **G** = goroutine, **M** = machine (OS thread), **P** = processor (logical scheduling resource, count = `GOMAXPROCS`). A G runs only on an M that holds a P. Each P has a local run queue (and a global queue for sharing); idle Ps **steal** work from other Ps to balance load. When an M blocks on a syscall, the runtime **detaches the P** and lets it (or a freshly woken M) run other Gs, while the blocked M waits. Since 1.14 the runtime **asynchronously preempts** long-running goroutines via a signal-based mechanism, so a tight `for {}` with no calls no longer starves the scheduler.

### Q19: What's the difference between a buffered and an unbuffered channel?
**Answer:** An unbuffered channel `make(chan T)` synchronizes sender and receiver: the send blocks until a receiver is ready (a rendezvous). A buffered channel `make(chan T, n)` lets up to `n` sends complete without a receiver. Both still block when full (send) or empty (receive) after the buffer/concurrency is exhausted. A buffered channel is good for batching, decoupling producer rate from consumer rate, or as a semaphore (`make(chan struct{}, n)`). An unbuffered channel is good for explicit handshakes and synchronization.

### Q20: What happens on sending to, receiving from, and closing a channel?
**Answer:** Receive from a closed channel returns the remaining buffered values; once drained, subsequent receives return the zero value with `ok=false` and **never block**. Send on a closed channel **panics**. Closing a closed channel **panics**. Sending on a nil channel blocks forever; receiving on a nil channel blocks forever (useful to disable a `select` case dynamically). The cliché is "only the sender closes the channel"; many receivers close to signal "I'm done" via `defer close` of a downstream channel they own — but cross-goroutine close coordination requires care.

### Q21: How does the `select` statement behave when multiple cases are ready?
**Answer:** Pseudo-random uniformly among the ready cases. There's no priority order; the runtime deliberately picks one randomly so code doesn't accidentally depend on case order. To enforce priority, do a non-blocking `select { case x := <-c: use(x); default: }` first, then a blocking select with the rest. Adding `default` makes the whole `select` non-blocking; without one, all-ready-blocks. `time.After` and `ctx.Done()` are common cases that produce timeouts and cancellation.

### Q22: What are best practices for `context.Context`?
**Answer:** Accept `context.Context` as the first parameter of every function that does I/O or starts goroutines. Pass the parent down; cancel derives new contexts. **Always `defer cancel()`** any context you create with `WithCancel`/`WithTimeout`/`WithDeadline` to release resources promptly, even on early returns or panics. Use `WithValue` only for cross-cutting concerns (trace ID, request ID, authenticated principal) and never for business parameters. Use unexported key types to avoid collisions in the value bag. Don't store contexts in struct fields or globals.

### Q23: How do goroutine leaks happen and how do you prevent them?
**Answer:** A goroutine leaks when it blocks indefinitely — sending on a channel nobody reads, waiting on a context that's never cancelled, or in a `select` with no actionable case. Since they're cheap you won't notice one; you'll notice 10,000. Prevention: every blocking call must have a cancellation path — accept a `context.Context`, and `select` it alongside the blocking op:

```go
select {
case v := <-in: handle(v)
case <-ctx.Done(): return
}
```

Race detector running in tests doesn't catch leaks directly, but `go.uber.org/goleak` and `go.uber.org/gomemleak` detect them; production servers should expose `/debug/pprof/goroutine`.

### Q24: How do you detect data races?
**Answer:** Build/test with `-race`: the Thread Sanitizer instrumented build catches races actually executed during the run and reports both goroutines' stacks. Run it in CI on every PR. It only reports races **observed** in the run, so high-coverage tests help; it's not exhaustive. Beyond `-race`, the race detector understands common patterns — but you must structure concurrent accesses with channels, mutexes, or atomics. For server production, also enable `-race` in a small fraction of requests ("production race detector") to catch rare races.

### Q25: When would you use mutexes vs channels for concurrency?
**Answer:** There's no right answer but a Go-flavored one: communicate by sharing memory only when state has natural ownership transfer between goroutines; otherwise share memory behind a mutex. Mutex is good for shared cache and counters; channels are good for passing ownership and signaling. Don't try to force every problem into one model. The famous Go proverb "Don't communicate by sharing memory; share memory by communicating" is a bias, not an injunction — `sync.Mutex` exists for a reason.

### Q26: How do `sync.WaitGroup.Add`, `Done`, and `Wait` orderings work?
**Answer:** `Add(n)` increments the counter; `Done()` is `Add(-1)`; `Wait()` blocks until the counter hits zero. Critically, `Add` must happen **before** the spawned goroutine starts if you're inside `Wait` immediately otherwise race-insensitive — and definitely before the matching `Done`. A common bug is calling `Add(1)` inside the goroutine body — the parent may call `Wait` before the child runs `Add`, see zero, and return too early. Add then `go func(){ defer wg.Done() }()`.

### Q27: When do you choose `sync/atomic` over `sync.Mutex`?
**Answer:** Atomic shines for a single integer or pointer state — counters, flags, single-slot CAS — where there's no compound operation. It's faster than a mutex (no lock acquisition or wakeup) and doesn't park the goroutine. Mutex is good for multi-statement critical sections, compound operations on multiple fields, or anything that benefits from reader/writer phase distinction (`RWMutex`). Atomics also establish happens-before edges for ordering non-atomic loads on the same goroutine, but not for unrelated fields — those still need mutex or channels. For 90% of cases: reach for mutex first; switch to `atomic.Int64` only if profiling points to a hot counter.

### Q28: What's the difference between `errors.Is` and `errors.As`?
**Answer:** `errors.Is(err, sentinel)` walks the unwrap chain comparing against a sentinel value with optionally-customized `Is(target) bool`. `errors.As(err, &target)` walks the chain and **assigns** the first error of a compatible concrete type into `target` — useful when the inner error carries structured data (a `*fs.PathError`'s `Path`, `Op`, `Err`). Both honor the `%w` verb in `fmt.Errorf`. Compare with `err == sentinel` which only checks identity at the outermost layer (won't see wrapped errors) — always prefer `errors.Is`.

## Hard

### Q29: Explain Go generics type constraints and the `comparable` shorthand.
**Answer:** A constraint is an interface; since 1.18 it may contain a `Union` of types via `~T | U` (the `~` means "and any named type whose underlying type is T"). `any` is alias `interface{}` (no constraints). `comparable` is a builtin "supports `==`" interface; it covers primitives, pointers, channels, arrays/structs of comparable fields, and interfaces themselves (with care — comparing interfaces with non-comparable dynamic types panics at runtime). A common pattern: `func Contains[T comparable](s []T, x T)` requires only equality; you can't use `comparable` when you want ordering, so reach for `cmp.Ordered` from the `cmp` package (Go 1.21+).

### Q30: How does error wrapping work and when do you use `%w`?
**Answer:** `fmt.Errorf("ctx: %w", err)` returns an error whose `Unwrap()` returns `err`, which lets `errors.Is`/`errors.As` traverse the chain. Use `%w` whenever you want callers down the stack to classify your error: `Is(err, fs.ErrExist)` works through wrappers. Use `%v` only for purely textual context where classification isn't needed. Always wrap at the call boundary to add who/where. Don't double-wrap the same error twice at consecutive layers — it makes `errors.Unwrap` chains deep without adding value. With multiple errors, `errors.Join(errs...)` (Go 1.20) builds a single error that walks across all children.

### Q31: When is `panic` appropriate and how does `recover` work?
**Answer:** Panic is appropriate for genuinely unrecoverable conditions: programmer-bug invariants, "this should never happen",突发 impossible states. It's also appropriate for short-circuiting deeply nested logic where you immediately abort, but then `recover` should be at a top-level boundary (e.g., an HTTP handler) so the goroutine survives. `recover` must be called **in a deferred function of the same goroutine** — a deferred in another goroutine won't catch the panic of the first. Example:

```go
func recoverer(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic: %v", rec); w.WriteHeader(500)
            }
        }()
        h.ServeHTTP(w, r)
    })
}
```

Don't use panic to signal expected errors — return `error` values.

### Q32: Explain escape analysis and how to interpret `-gcflags="-m"`.
**Answer:** Escape analysis decides whether each variable lives on the stack (cheap) or escapes to the heap (GC-managed). The compiler walks the program graph; a value escapes if a pointer to it might outlive its stack frame — by being returned, stored in an interface, captured by a closure, sent on a channel, or stored into the heap. `go build -gcflags="-m"` prints decisions like `x.go:10: x escapes to heap`; `-m -m` adds rationale. Reading this output is essential for optimizing allocations: you'll spot functions where small structs accidentally escape, e.g., due to interface conversion. Go 1.24/1.26 have improved several common cases (closure capture, slice growth).

### Q33: How does `pprof` work and what do you look at?
**Answer:** `pprof` is a sampling profiler that records stack traces at a configurable rate (CPU default 100 Hz). You enable either `runtime/pprof` for batch programs or `net/http/pprof` for servers; the latter exposes `/debug/pprof/...` endpoints. You capture by `go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30` (CPU) or `heap`, `allocs`, `goroutine`, `block`, `mutex`. Inside the interactive shell, `top` ranks functions by sample count, `list <regex>` annotates the source, and `web` produces an SVG call-graph. For heap profiles, distinguish `inuse_space` (live) from `alloc_space` (cumulative) — they answer different questions ("what is heaviest now" vs "what churns most").

### Q34: Why are table-driven tests the idiomatic form in Go?
**Answer:** They keep test code DRY by factoring the assertion loop while still recording individual case results via `t.Run`. Each case is its own subtest, so `go test -v` prints name separators, `go test -run TestFoo/decimal` runs only that case, and failures in one case don't abort the others (unless you use `t.Fatal` inside it). They also encourage covering edge cases (empty input, negatives, off-by-one) — you enumerate the category, then fill rows. Compared to a series of separate test functions, the maintenance cost is much lower because you only edit a struct row to add a case.

### Q35: How is a Go map implemented internally, and why is iteration order random?
**Answer:** A Go map is a hash table with **buckets and overflow lists**, where buckets are arrays of 8 slots. The hashing is randomized by a per-map seed (the `hash0` field) initialized at `make` time; iteration starts at a **random bucket and random slot within that bucket**, which is why range order differs across runs. This randomization is intentional — it protects against hash collision DoS attacks and prevents code from depending on iteration order. Concurrent writes break the bucket growth invariants, hence the concurrent-map-writes runtime panic. Maps with comparable keys support `==`; using a non-comparable key panics.

### Q36: Discuss the nil-interface-vs-interface-with-nil-value gotcha in a real-world bug.
**Answer:** The classic scenario: a function returns an `error`, calls a helper that returns a typed error pointer, and stores the result:

```go
type RepositoryError struct{ Code int }
func (e *RepositoryError) Error() string { return "repo error" }

func findRepo(id int) error {
    var e *RepositoryError
    if id == 0 { return e }       // BUG: non-nil interface wrapping nil pointer
    e = &RepositoryError{Code: id}
    return e
}

if err := findRepo(0); err != nil {
    log.Fatal("got error: ", err) // prints a nil-looking error
}
```

The fix: declare local variable of **concrete pointer** only when you'd assign non-nil; otherwise return literal `nil` directly. Or — preferred — initialize returns as `error(nil)`. Linters like `nilerr` and `wrapcheck` catch the "return nil, non-nil interface" smells.

### Q37: How do you structure a worker pool with backpressure?
**Answer:** Use a buffered channel as a bounded job queue and a separate semaphore channel to limit concurrent workers:

```go
jobs := make(chan Job, 100)        // queue
sem := make(chan struct{}, 8)      // max workers

for _, j := range work {
    select {
    case jobs <- j:
    case <-ctx.Done(): return
    }
    sem <- struct{}{}
    go func(j Job) {
        defer func(){ <-sem; if r := recover(); r != nil { log.Print(r) } }()
        process(ctx, j)
    }(j)
}
```

If `jobs` (bounded queue) fills, the producer blocks — this is backpressure. The semaphore bounds concurrency so the runtime doesn't explode with goroutines. Add a `sync.WaitGroup` to wait for outstanding workers before exiting. Alternatively, use `golang.org/x/sync/semaphore.Weighted` for weighted limits across heterogeneous tasks.

### Q38: Describe the happens-before edges in Go's memory model.
**Answer:** Go's memory model is **not** sequentially consistent globally; programs are race-free only when ordering is established via synchronisation edges: (1) `go f()` of a goroutine happens-before `f` runs; (2) a channel **send** happens-before the corresponding **receive** completes (closed-channel receive happens-after `close`); (3) an `Unlock` of a mutex happens-before the next `Lock`; (4) `sync.Once.Do(f)` of all calls happens-after the single `f` returns; (5) `WaitGroup.Add(n)` from one goroutine happens-before a `Wait` returns; (6) atomic operations on the same address are linearised. Without one of these edges, the memory model doesn't constrain what one goroutine sees after another writes — which is what `-race` helps uncover.

### Q39: How would you profile memory in a long-running server and what do `inuse_space` and `alloc_space` tell you?
**Answer:** Expose `net/http/pprof` and hit `/debug/pprof/heap` for `inuse_space` (live objects), or `/debug/pprof/allocs` for `alloc_space` (cumulative allocations); the same endpoint takes `?gc=1` to force a GC first, or `?seconds=30` for a sampling profile. `inuse_space` is the right view for "where's my memory now?" (good for hunting leaks); `alloc_space` is right for "what allocates the most?" (good for hunting churn and GC pressure). `go tool pprof` lets you swap sample index: `(pprof) sample_index = alloc_space`. Always profile in production-like conditions: with production traffic, real payloads. Pair with `benchstat` over before/after to verify an optimization helped.

### Q40: Why choose one `sync` primitive over another in a high-throughput scenario?
**Answer:** For a counter that's updated millions of times per second: `atomic.Int64` wins over `sync.Mutex` because there's no lock acquisition or goroutine wakeup, and no contention beyond CMPXCHG spinning. For shared state that requires compound operations across fields, `sync.Mutex` is the right call — atomics don't compose. For a read-heavy cache, `RWMutex` is acceptable but beware writer starvation (mitigated since 1.9); if keys are mostly static after warmup, `sync.Map` may be faster. For one-shot initialization, `sync.Once`. For a fan-out-and-wait, `WaitGroup`. For a per-P reusable buffer cache, `sync.Pool`. The "thou shalt use channels" guideline applies to communication patterns, not numeric counters; using channels to count requests would be absurdly expensive.

### Q41: How do generics interoperate with interfaces, and what are the trade-offs?
**Answer:** Generic functions can take interface-typed parameters (`func Print[T fmt.Stringer](t []T)`); you can also constrain type parameters to interfaces. Differences: interfaces use dynamic dispatch (a vtable lookup on receiver), generics are static — values remain their concrete type at compile time. Generics avoid boxing values into interfaces (a big win for `[]int`-style processing); interfaces can let users define **new types after your library is compiled**, which generics cannot (a type must statically match the constraint). For library APIs, prefer small interfaces that consumers can implement, and use generics inside the library to factor internal helpers. Since 1.24 you can alias generic types: `type Set[T comparable] = map[T]struct{}` to expose a generic type alias cleanly.