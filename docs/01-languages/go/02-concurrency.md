# Concurrency

Concurrency is Go's defining feature and the most common senior-interview topic. This file covers the model deeply.

## Goroutines

A goroutine is a function executing concurrently with other goroutines. `go f(args)` starts it.

```go
go handleReq(req)
go func(id int) { process(id) }(42)
```

- Goroutines are managed by the Go runtime, not the OS. They start with **small stacks (2 KB)** that grow/shrink dynamically, so you can have hundreds of thousands on a single machine.
- They are **M:N scheduled** onto OS threads by the runtime.
- An anonymous goroutine can't return a value directly; use a channel, `sync.WaitGroup`, or a `context` to coordinate.

## The GMP scheduler model

The runtime scheduler has three core abstractions (the letters are Go tradition):

| Symbol | Meaning |
|--------|---------|
| **G** | Goroutine — a user-level coroutine with its own stack, instruction pointer, scheduling metadata. |
| **M** | Machine — an OS thread; the actual executor. M must have a P to run user Go code. |
| **P** | Processor — a logical resource holding the local run queue of runnable Gs and the mcache; the count is `GOMAXPROCS`. |

Rules:

- Number of P is `GOMAXPROCS`, default = number of CPU cores. Since Go 1.25 the default is **container-aware** on Linux: it respects cgroup CPU quotas (a Kubernetes CPU limit of 2 gives `GOMAXPROCS=2`, not the node's core count) and the runtime re-checks the limit periodically. Before 1.25 you needed `uber-go/automaxprocs` — a classic "Go in Kubernetes" interview point.
- A G runs only when an M holds a P.
- Each P has a **local run queue** (256 slots, FIFO-ish) plus a global run queue shared by all Ps. Work-stealing: an idle P steals half of another P's queue when its own is empty.
- When a G blocks on a syscall, the runtime **hands off** the P to another M (or wakes one) so other Gs keep running. The blocked M/G wait on the syscall. With `network` poller (async I/O) the blocking G is parked and the same M/P keeps working.
- **Preemption** (since Go 1.14): the runtime sends an async `SIGURG`-based preemption point to long-running goroutines (no function calls). Previously a tight `for {}` loop could starve the scheduler across `GOMAXPROCS`; now the runtime forcefully preempts it after ~10ms. This is essential for fairness in long-running CPU-bound goroutines.

Memory: a goroutine stack starts small and grows by copying. The runtime is conservative about when it can shrink stacks (Go 1.16+ can shrink it).

## Channels

A channel is a typed, thread-safe FIFO pipe for goroutines to communicate values. The Go proverb: **"Don't communicate by sharing memory; share memory by communicating."**

```go
ch := make(chan int)    // unbuffered
ch := make(chan int, 5) // buffered capacity 5
ch <- 42                // send
v := <-ch               // receive
v, ok := <-ch           // ok=false on a closed, drained channel
close(ch)               // close: no more sends permitted; remaining values still receivable
for v := range ch { }   // range over channel: stop when closed
```

### Unbuffered vs buffered

- **Unbuffered channel send blocks until some goroutine receives.** Receive blocks until some send is in flight. It is a **synchronous rendezvous**.
- **Buffered send blocks only when the buffer is full**; receive blocks only when empty. A buffered channel is asynchronous up to its capacity.

```go
ch := make(chan int) // unbuffered
go func() { ch <- 1 }()      // blocks until someone receives
<-ch                          // unblocks the sender

buf := make(chan int, 1)
buf <- 1                      // does not block; one slot available
buf <- 2                      // blocks until previous is consumed
```

### Closing channels

- Only the **sender** should close. Closing signals "no more values." Closing twice **panics**.
- Receiving from a closed channel returns the zero value with `ok=false`; it never blocks again.
- Sending on a closed channel **panics**.
- Range over a channel exits when it becomes closed (and drained).
- Closing an already-closed or a nil channel blocks/deadlocks/panics as appropriate (closed twice panics; send to closed panics).

### Nil channel behavior

A nil channel send or receive **blocks forever**. This is sometimes useful in `select` to disable a case dynamically:

```go
var in chan int // nil until initialized in some condition
select {
case v := <-in: handle(v) // never runs while in == nil
case <-ctx.Done():
    return
}
```

## The `select` statement

`select` chooses one ready case (send or receive) pseudo-randomly; multiple ready cases compete uniformly.

```go
select {
case v := <-in: handle(v)
case out <- compute():        // send; blocks if out is full
case <-time.After(2*time.Second): fmt.Println("timeout")
case <-ctx.Done(): return
default: fmt.Println("nothing ready") // optional, makes select non-blocking
}
```

Patterns:

- **Timeout**: `case <-time.After(d)`. Before Go 1.23 each call allocated a timer that was not collected until it fired — a real leak in tight loops. Since 1.23 unstopped timers are GC-eligible immediately, so `time.After` is safe; reusing a `time.Timer` still avoids allocation churn in hot loops.
- **Non-blocking operation**: pair with `default`:
  ```go
  select { case v := <-ch: use(v); default: }
  ```
- **Random choice** among multiple ready cases (no priority). To enforce priority order, nest `select { case ...; default: }` inside.

## The `sync` package

### Mutex

```go
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
// critical section
```

- `sync.Mutex` is non-reentrant — locking twice from the same goroutine without Unlock self-deadlocks; the runtime only detects it if *every* goroutine ends up blocked.
- `sync.RWMutex` offers `RLock`/`RUnlock` for shared reads, `Lock`/`Unlock` for exclusive; useful when reads dominate. Beware: under contention, writers can starve in older Go; the writer-preference scheduling in Go 1.9+ mitigates this.
- **Never copy a Mutex**; pass structs holding one by pointer. `go vet` flags this.

### WaitGroup

```go
var wg sync.WaitGroup
for i := 0; i < n; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); work() }()
}
wg.Wait()
```

- `Add` must happen before the goroutine starts (positive delta) — otherwise a race between `Add` and `Wait`. `Done` decrements by 1.
- `wg.Add(-1)` is legal but error-prone; prefer `Done`.
- **Go 1.25 added `WaitGroup.Go`**, which wraps the Add/Done bookkeeping and is now the idiomatic form:

```go
var wg sync.WaitGroup
for _, t := range tasks {
    wg.Go(func() { work(t) })
}
wg.Wait()
```

`go vet` also gained a `waitgroup` analyzer (1.25) that flags misplaced `Add` calls.

### Once

```go
var initOnce sync.Once
initOnce.Do(func() { x = expensive() })
```

Guarantees exactly one execution of `Do(f)` across goroutines even on panic-recover sequences.

### Cond

```go
var (
    mu   sync.Mutex
    cond = sync.NewCond(&mu)
    ready bool
)

// waiter
mu.Lock()
for !ready { cond.Wait() }
mu.Unlock()

// signaler
mu.Lock()
ready = true
cond.Broadcast() // or Signal to wake one
mu.Unlock()
```

`Wait` atomically unlocks `mu` and suspends; wakes locked again on return. Always loop on the predicate (no "spurious wakeups" assumption—but defensive looping is idiomatic and helps when the predicate isn't globally true after wakeup).

### Map

`sync.Map` is for two specific cases:

1. Write-once, read-many caches (e.g. memoizing).
2. Multiple goroutines writing disjoint keys.

For most maps, a `map` + `sync.RWMutex` is faster (sync.Map optimizes for the disjoint/once cases, but it stores interface{} and has extra indirection).

APIs: `Store, Load, LoadOrStore, LoadAndDelete, Delete, Range`.

### Pool

`sync.Pool` is a per-P cache of reusable objects; clears on GC. Useful for short-lived buffers, e.g.:

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func process(s string) string {
    bp := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(bp)
    bp.Reset()
    bp.WriteString(s)
    // ...
    return bp.String()
}
```

**Caveat**: a pooled object may be reclaimed between Get/Put; never keep a reference after Put. Never Put structs that are managed elsewhere (`http.Request`, `bytes.Buffer` from `bufio.Scanner`).

### Atomic

```go
var c int64
atomic.AddInt64(&c, 1)
old := atomic.LoadInt64(&c)
atomic.StoreInt64(&c, 100)
ok := atomic.CompareAndSwapInt64(&c, expected, new)

var flag atomic.Bool
flag.Store(true)
flag.Load()
```

The `sync/atomic` package exposes functions per numeric width, plus typed `atomic.Bool`, `atomic.Pointer[T]` (since Go 1.19), `atomic.Int32/Int64/Uint32/Uint64/Uintptr`. Use these typed wrappers — clearer than `*int64` arithmetic.

## The `context` package

`context.Context` carries deadlines, cancellation, and request-scoped values across API and goroutine boundaries. It is mandatory for goroutine-aware code.

Construction:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
ctx, cancel := context.WithDeadline(parent, time.Now().Add(5*time.Second))
ctx = context.WithValue(parent, key, value)

// Go 1.20/1.21 additions worth knowing:
ctx, cancel := context.WithCancelCause(parent) // cancel(err); context.Cause(ctx) retrieves why
ctx = context.WithoutCancel(parent)            // detach: keeps values, drops cancellation
stop := context.AfterFunc(ctx, cleanup)        // run cleanup when ctx is done
```

- Cancel **propagates** to children derived from the cancelled context. That's how you stop a tree of goroutines on a single request abort.
- Always `defer cancel()` to release resources even on early returns.
- **`context.Background()`** is the root for top-level operations; `TODO()` is the temporary placeholder when you haven't decided.

### Value guidelines

`WithValue` is overused. Rules to follow:

- Use **Value only for cross-cutting concerns** the API can't take as parameters: trace IDs, request IDs, tenant scopes. Not for routing data or business parameters.
- Use an **unexported key type** of an arbitrary type:
  ```go
  type ctxKey int
  const keyRequestID ctxKey = 0
  ```
  Avoid string keys — collisions are real.
- Never pass a `Context` as a struct field; it's a method/function parameter, conventionally the first.

### Context misuse patterns (code-review checklist)

Interviewers increasingly ask "what would you flag in review?" — these are the classics:

- **Storing a context in a struct** — ties an object to one request's lifetime; pass per-call instead (the rare exception, e.g. `http.Request`, documents itself).
- **`context.WithValue` as a dependency injector** — loggers, DB handles, and config belong in constructors, not the value bag.
- **Ignoring `ctx` in blocking calls** — a `select` without a `<-ctx.Done()` case, or I/O helpers that never accept a context, defeat cancellation end-to-end.
- **Background work inheriting request cancellation** — audit logging or cache-fill that must outlive the request should use `context.WithoutCancel(ctx)` (1.21), not `context.Background()` (which drops trace values too).
- **Not checking `ctx.Err()`/`context.Cause(ctx)`** after Done fires — callers can't distinguish timeout from explicit cancel.
- **`context.TODO()` left in production code** — it's a marker, not a destination.

## Common pitfalls

### Data races

A race is when two goroutines access the same memory, at least one is a write, and they're unsynchronised. Symptoms range from "works fine" to corrupted bytes. The `-race` detector uses Thread Sanitizer to find them at runtime; it costs 5-10x slowdown so run it on tests.

```sh
go test -race ./...
```

Always: protect shared state with channels, `sync.Mutex`, `sync/atomic`, or restrict it to a single goroutine that owns it.

### Goroutine leaks

A goroutine that blocks forever (waiting on a channel nobody will send to, or a context never cancelled) leaks. Since goroutines are cheap you may not notice until thousands stack up, leaking memory and file descriptors.

Mitigation: always accept a `context.Context` and `select { case <-ctx.Done(): return }` in any waiting statement.

Detection: `go.uber.org/goleak` in tests; `/debug/pprof/goroutine` in production. Go 1.26 adds an experimental **goroutine-leak profile** (`GOEXPERIMENT=goroutineleakprofile`, exposed at `/debug/pprof/goroutineleak`) that reports goroutines blocked on concurrency primitives no longer reachable by any running goroutine — i.e., provably leaked.

### Deadlocks

The runtime **detects when all goroutines are blocked** and panics with `fatal error: all goroutines are asleep - deadlock!`. Common causes: sending on a channel no one will receive from, two goroutines each waiting on the other.

## Concurrency patterns

### Worker pool

Fixed goroutine count pulling jobs:

```go
func workerPool[T any](ctx context.Context, in <-chan T, n int, work func(T)) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done(): return
                case v, ok := <-in:
                    if !ok { return }
                    work(v)
                }
            }
        }()
    }
    wg.Wait()
}
```

### Fan-in / fan-out

Fan-out: one source feeds many workers (each consumes part of the work). Fan-in: many senders merge into one channel:

```go
func fanIn[T any](chans ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    for _, c := range chans {
        wg.Add(1)
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c { out <- v }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

Generics make this reusable from Go 1.18+.

### Pipeline

Each stage reads from input, writes to output:

```go
stage1() -> chan1 -> stage2(chan1) -> chan2 -> stage3(chan2)
```

Compose small functions that take / return channels.

### Done channel

Pre-`context` tradition: pass a `done <-chan struct{}`; close it to cancel.

```go
func gen(done <-chan struct{}) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; ; i++ {
            select {
            case out <- i:
            case <-done: return
            }
        }
    }()
    return out
}
```

Modern code uses `context.Context` instead.

### Semaphore with buffered channel

Limit concurrency cheaply:

```go
sem := make(chan struct{}, 10) // 10 concurrent
for _, w := range work {
    sem <- struct{}{}
    go func(w Work) {
        defer func() { <-sem }()
        process(w)
    }(w)
}
```

`golang.org/x/sync/semaphore` offers a weighted semaphore with a nicer API; the buffered-channel trick remains idiomatic and dependency-free.

### errgroup (structured concurrency in practice)

`golang.org/x/sync/errgroup` is the de-facto tool for "run N tasks, fail together, wait for all" — expect to be asked for it by name:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8)                       // bounded concurrency
for _, u := range urls {
    g.Go(func() error {
        return fetch(ctx, u)        // first error cancels ctx for the rest
    })
}
if err := g.Wait(); err != nil { return err }
```

Compared to a raw `WaitGroup`: error propagation, cancellation of siblings on first failure, and a built-in concurrency limit.

## Memory model basics (happens-before)

Go's memory model defines when one goroutine is guaranteed to observe a write by another. It is **not** a sequentially consistent model; you rely on synchronisation to establish happens-before relations.

- A `go f()` statement happens-before the new goroutine's `f` starts.
- A send on a channel happens-before the corresponding receive completes.
- A receive on a closed channel happens-after the close.
- An `Unlock` happens-before the next `Lock` on the same mutex.
- A `sync.Once.Do(f)` from any goroutine happens-after the single `f` returns.

In short: when there is no synchronisation edge between a write W and a read R, the memory model places **no constraint** on what R observes. Race-free programs achieve correctness by establishing edges via channel ops, mutex operations, `Once`, `WaitGroup.Done`→`Wait`, atomic ops.

- **Race-free** means every memory access between two goroutines is ordered by an explicit synchronisation edge.
- **Atomic operations** are linearisable and establish happens-before with other atomics on the same address but not, in general, with non-atomic accesses on different addresses (your program must still order non-atomic accesses with mutexes/channels).

## Generics in concurrent code

From Go 1.18 you can write generic goroutine-safe utilities:

```go
func Future[T any](ctx context.Context, f func() T) func() (T, error) {
    var (
        v   T
        err error
        m   sync.Mutex
        c   = make(chan struct{})
    )
    go func() {
        defer close(c)
        fv, fe := safe(f)
        m.Lock()
        v, err = fv, fe
        m.Unlock()
    }()
    return func() (T, error) {
        select {
        case <-c:
            m.Lock()
            defer m.Unlock()
            return v, err
        case <-ctx.Done():
            var z T
            return z, ctx.Err()
        }
    }
}
```

One gotcha: a generic type parameter `T any` cannot be sent on a channel unless you instantiate it; channels of generic types are fine (`chan T`).

## sync/atomic vs mutex trade-offs

- **atomic** is for single variable/state machines — counters, flags, single-slot CAS. Faster, no lock contention. Lock-free but linearised.
- **mutex** is for multi-statement critical sections or coordinated state across fields. Easier to reason about, more flexible.
- For high-throughput counters (`ops/sec`, inflight gauge), `atomic.Int64` is the right choice.
- For anything where you need to update two fields together under a snapshot of state, mutex.

Rule of thumb: prefer mutex for clarity; reach for atomic when profiling shows mutex cost on a single counter.

## Pitfalls to remember

- Forgetting `defer cancel()` keeps a `context.WithTimeout` child registered with its parent (and its timer alive) until the deadline fires; `go vet`'s `lostcancel` check flags this.
- Calling `cancel()` is fine if the parent is also cancelled; calling cancel twice is OK (idempotent).
- A buffered channel with capacity N is not a synchronization for N+1 sends.
- `WaitGroup.Add` outside the spawning range: `wg.Add(1)` must occur before the matched `go func()`. Add inside the goroutine races with `Wait`.
- Iteration variable capture: from Go 1.22 per-iteration, but pre-1.22 closures captured the same `i` (loops captured by reference). Read each file's `for` perf-foot-gun rule.
- `close` from the receiver side is a frequent bug; closing a channel from a goroutine that isn't the sender can panic another goroutine's later send.
- Testing concurrent, time-dependent code with real sleeps is flaky by construction — use `testing/synctest` (stable since Go 1.25; see `04-testing-and-performance.md`).