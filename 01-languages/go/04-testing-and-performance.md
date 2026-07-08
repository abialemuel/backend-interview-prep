# Testing and Performance

The `go test` tool, the `testing` package, and the bundler of `pprof`/`trace` tooling are part of what makes backend Go pleasurable. Expect interview questions on every section below.

## The `testing` package

A test file ends in `_test.go`. Test functions:

```go
func TestFoo(t *testing.T) {
    if got := foo(1); got != 2 {
        t.Errorf("foo(1) = %d, want 2", got)
        // or t.Fatalf to stop
    }
}
```

- Functions named `TestXxx` take `*testing.T`. Subtests with `t.Run`.
- `t.Fail()` marks failed but continues; `t.Fatal/Fatalf` fails and stops; `t.SkipNow/Skip` skips.
- `t.Parallel()` signals the test (or subtest) can run in parallel with other parallel tests. Since 1.22 subtests of the same test can interleave.
- `t.Cleanup(f)` registers a function to run when the test (and all its subtests) completes — preferred over `defer` for parallel subtests.

Run:

```sh
go test ./...                  # run all
go test -run TestFoo/pkg        # filter by regex
go test -v ./...
go test -count=1 ./...          # disable caching
go test -failfast
go test -short                 # skip long tests via testing.Short()
```

The tool caches test **results** (not runs) keyed by input digest; `-count=1` disables the cache.

## t.Helper and test helpers

A helper is a function that calls `t.Helper()` so that failure line reports point to the caller:

```go
func assertEqual[T comparable](t *testing.T, want, got T) {
    t.Helper()
    if want != got {
        t.Errorf("want %v, got %v", want, got)
    }
}
```

Use it for every assertion-style helper so the file:line reported is the test, not the helper.

## Table-driven tests

Idiomatic. One test function iterates a slice of cases and uses `t.Run` to give each its own name (and tracking in output):

```go
func TestParseAtoi(t *testing.T) {
    cases := []struct {
        name string
        in   string
        want int
        err  bool
    }{
        {"decimal", "42", 42, false},
        {"negative", "-7", -7, false},
        {"empty", "", 0, true},
        {"bad", "xyz", 0, true},
    }
    for _, c := range cases {
        t.Run(c.name, func(t *testing.T) {
            n, err := strconv.Atoi(c.in)
            if c.err {
                if err == nil { t.Fatal("want error, got nil") }
                return
            }
            if err != nil { t.Fatalf("unexpected: %v", err) }
            if n != c.want { t.Fatalf("got %d, want %d", n, c.want) }
        })
    }
}
```

Reasons to always use `t.Run`:

- Each case runs as a subtest; `go test -v` numbers them.
- A failing case doesn't stop others (unless `t.Fatal` inside it).
- You can run a single case by name: `go test -run TestParseAtoi/bad`.

## Example tests

`ExampleXxx` functions double as documentation and tests:

```go
func ExampleAdd() {
    fmt.Println(add(1, 2))
    // Output: 3
}
```

If the `// Output:` comment is present, `go test` compares stdout against it.

## Benchmarks

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(1, 2)
    }
}
```

`b.N` is grown by the framework until a stable duration is measured (~1s). The body must process the entire `b.N`-sized workload so timings are valid.

Run with:

```sh
go test -bench=. -benchmem -benchtime=2s -count=5
go test -bench=BenchmarkX -run=^$  # no tests alongside
```

Use `b.ReportAllocs()` to print allocs/op alongside ns/op; `b.ReportMetric` for custom metrics.

### b.RunParallel

For benchmarks of CPU-bound parallel workloads:

```go
func BenchmarkWorkers(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            work()
        }
    })
}
```

The framework scales goroutines with `GOMAXPROCS`; each `pb.Next()` consumes a chunk of `b.N`.

### Common benchmark pitfalls

- **Compiler optimization elision**: if the result of your inner work is unused, the optimizer may eliminate it. Assign to a package-level sink:
  ```go
  var Sink int
  func BenchmarkX(b *testing.B) {
      var x int
      for i := 0; i < b.N; i++ { x = work() }
      Sink = x
  }
  ```
- **Setup not excluded**:
  ```go
  func BenchmarkX(b *testing.B) {
      setup()
      b.ResetTimer()
      for i := 0; i < b.N; i++ { work() }
  }
  ```
- **Allocations inside timer**: prepare inputs outside the loop or call `b.StopTimer()`/`b.StartTimer()`.
- **Naïvely trusting ns/op once**: use `-count=5` and the `benchstat` tool to compare; timings vary.

## Fuzzing (since Go 1.18)

A fuzz target lives in a `_test.go` file and looks like:

```go
func FuzzParseURL(f *testing.F) {
    f.Add("http://example.com/foo") // seed corpus
    f.Add("http://localhost:8080")
    f.Fuzz(func(t *testing.T, in string) {
        u, err := url.Parse(in)
        if err != nil { return }
        out := u.String()
        if _, err := url.Parse(out); err != nil {
            t.Fatalf("non-idempotent: %q -> %q", in, out)
        }
    })
}
```

- `f.Add(...)` registers seed corpora; types must match the `f.Fuzz(func(*T, A, B, ...))` signature.
- `go test -fuzz=FuzzParseURL` runs the fuzzing engine; finding inputs that violate invariants or panic.
- `go test -fuzztime=10m` controls duration; `go test -fuzzminimizetime` time spent minimising a found corpus entry.
- Fuzz corpus is stored in `testdata/fuzz/<FuncName>/` and is automatically re-used in normal `go test` runs — fixes get regression coverage for free.

## Mocks/stubs

Three patterns:

1. **Function variables.** Replace package-level functions for tests:

   ```go
   var currentTime = time.Now
   func TestX(t *testing.T) {
       old := currentTime
       defer func(){ currentTime = old }()
       currentTime = func() time.Time { return fixed }
   }
   ```

2. **Function-typed struct fields** for dependency injection:

   ```go
   type Clock struct{ Now func() time.Time }
   svc := Service{Clock: Clock{Now: func() time.Time { return fixed }}}
   ```

3. **Generated mocks** for interfaces using `mockery` (` mockery --name=Reader`) or `mockgen` (`go.uber.org/mock`):

   ```go
   //go:generate mockgen -source=store.go -destination=mock_store.go -package=store
   ```

   Generated mocks assert call expectations, return configured values, and verify ordered calls. They over-specify when interfaces are big — keep interfaces small and you engineer better tests for the same effort.

`testing/fakeserver` (community) and `httptest.NewServer` are the standard ways to fake outbound HTTP.

## Coverage

```sh
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out          # browser
go tool cover -func=coverage.out          # per-function table
```

Coverage is counted at the **statement** granularity. 70–90% is typical; aim for high coverage on critical business logic, not right at boilerplate.

## Profiling: pprof

`runtime/pprof` and `net/http/pprof` expose sampling profiles. Start a CPU profile programmatically:

```go
import "runtime/pprof"

f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
work()
```

For a long-running server, expose debug endpoints:

```go
import _ "net/http/pprof"
http.ListenAndServe("localhost:6060", nil)
```

Then capture:

```sh
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30   # CPU
go tool pprof http://localhost:6060/debug/pprof/heap                 # allocations, live
go tool pprof http://localhost:6060/debug/pprof/allocs               # allocations total
go tool pprof http://localhost:6060/debug/pprof/goroutine             # current goroutines
go tool pprof http://localhost:6060/debug/pprof/block                # blocking samples
go tool pprof http://localhost:6060/debug/pprof/mutex                # contention
```

Inside `go tool pprof`:

- `top` — top frames by sample count.
- `list <regex>` — source-level annotation.
- `web` — graphviz SVG.
- `traces` — call stacks.
- `peek <regex>` — show what calls and what called a regex.

In Go 1.26 pprof adds more complete inline-frame attribution and improved symbolization, and experimental `view="lines"` for source-line summaries. The cumulative-mode (`-sample_index=inuse_space` vs `alloc_space`) is still important to set correctly.

## Trace

```sh
go test -trace=trace.out ./...
go tool trace trace.out
```

Browser UI shows goroutine scheduling, syscall blocks, GC events, network poller activity, per-P time slices. Indispensable for latency investigations.

You can also start a trace inside a running server:

```go
import "runtime/trace"
trace.Start(os.Stdout); defer trace.Stop()
```

## Race detector

Run any test or build with `-race`:

```sh
go test -race ./...
go build -race -o svc
go run -race main.go
```

The race detector instruments memory accesses and **only** reports races encountered during this particular run; it's a sampling tool, not exhaustive. Use it always in CI.

## Escape analysis and stack vs heap

Go has **no manual `malloc`**. The compiler decides whether each variable lives on the stack (cheap, freed automatically) or escapes to the heap (GC-managed). "Escape analysis" walks the program graph to determine whether a reference to a variable crosses a function boundary that would outlive its stack frame.

Common triggers for escape:

- Returning a pointer to a local.
- Storing a pointer in an interface (the value type isn't known at compile time, so it escapes).
- Sending a value on a channel that has capacity or survives the call.
- Storing into a slice that grows.
- Closure-captured variables whose lifetime exceeds the enclosing scope.

Inspect:

```sh
go build -gcflags="-m" ./...
go build -gcflags="-m -m" ./...    # more detail
```

Learn to read the output: `./x.go:10:7: x escapes to heap`. From Go 1.24 onward, the runtime improves escape analysis in several cases (slice-of-T->*[T] conversions via `unsafe.SliceData` patterns; stronger closures). In Go 1.26 escape analysis handles more closure patterns and grows-array scenarios more cleverly, helping common-place functions like `slices.Concat` avoid escapes.

## Performance tips

### Reduce allocations

Most server latency cost is allocation pressure → GC pauses → stop-the-world. Avoiding allocations:

- Preallocate slices/maps: `make([]T, 0, n)`, `make(map[K]V, n)`.
- Reuse buffers via `sync.Pool`.
- Use `bytes.Buffer` and `strings.Builder` instead of string concatenation.
- Return values (not pointers) for small types to prevent escape.
- Avoid `fmt.Sprintf`/`fmt.Errorf` in hot paths — they allocate.
- Prefer `strconv.AppendInt(buf, n, 10)` to `strconv.Itoa` when you have a buffer.

### strings.Builder and bytes

```go
var sb strings.Builder
sb.Grow(estimated)
for _, p := range parts { sb.WriteString(p) }
return sb.String()
```

`strings.Builder` avoids the quadratic cost of `+` and reuses a single underlying `[]byte`. Use `bytes.Buffer` when you need the buffer across many operations and don't want the final string-copy overhead.

### sync.Pool

For stateless, GC-friendly reuse (see `02-concurrency.md`). Be careful: `*bytes.Buffer` from `bufio.Scanner` is owned by the scanner; don't pool it.

### Preallocate slices/maps

```go
m := make(map[string]int, expectedSize)
s := make([]int, 0, expectedSize)
```

`make(map, hint)` adjusts initial bucket count; `make([]T, 0, cap)` avoids the early re-grows.

### Fast paths with `[]byte` and `string`

```go
const greeting = "hello"
buf := []byte(greeting) // copies once
```

The `string(byteslice)` ↔ `[]byte(string)` conversions copy; for indexing maps, use the `m[string(b)]` pattern — the compiler can elide the temp string (since Go 1.20 + tighter in 1.24 with SWAR search for short needles).

### Avoid reflection; use concrete types and generics

Reflection (`reflect`) is slow and ambiguous; when you'd otherwise use `any` with reflection, prefer generics to keep the types concrete.

### Inlining

Small functions may be inlined by the compiler; check with `-gcflags="-m"`. Trade-offs: too-big functions won't inline; sometimes a small hot helper that does the inner loop is good for performance.

### GC tuning

`GOGC` (default 100) sets the trigger ratio: GC triggers when heap doubles since the last GC. Lower for less latency, higher for throughput (more memory). `GOMEMLIMIT` (since 1.19) sets a soft memory ceiling: GC runs more aggressively to stay below it. In a container/Kubernetes setting, **set `GOMEMLIMIT`** to the cgroup limit (~85–90%) to avoid OOMKill while controlling GC cost (soft memory limit will trigger GC before hitting the hard limit).

### Benchmarking in CI

- Run on a quiet machine; no laptops throttling.
- `benchstat old.txt new.txt` to compare oranges to oranges.
- Run multiple iterations: `-count=10`.
- Beware CPU scaling/governors.

## testing/synctest (Go 1.26)

`testing/synctest` (formerly an experiment, now promoted in 1.26) provides a "fake clock" goroutine sandbox. Inside `synctest.Test(func(t *testing.T))`, the runtime sees time advance only when goroutines block. This tests time-dependent code without real sleeping:

```go
import "testing/synctest"

func TestBackoffRetries(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
        defer cancel()
        out := retryWithBackoff(ctx) // sleeps simulated instantly
        if out != expected { t.Fatalf("got %v", out) }
    })
}
```

This eliminates squishy "1ms tolerance" timing tests and makes `context.WithTimeout` tests deterministic.

## Pitfalls

- Running benchmarks on a default build — disable inlining/optimization carefully; or run a `cpu.prof` to know what you're actually measuring.
- Treating `b.N` as 1 — use a loop. The framework expects the body to scale with `b.N`.
- Comparing `ns/op` between different machines — use `benchstat` on the same hardware; consider `-cpu=1,4,8` plans.

## Summary checklist for an interview

1. Know that table-driven tests are the idiomatic form and `t.Run` for subtests.
2. Know that `b.N` is framework-controlled; you scale the loop.
3. Know how pprof samples CPU and heap, where to find in `pprof`, and the differences between `inuse_space` and `alloc_space`.
4. Know that `-race` is a CI must.
5. Know that escape analysis decides stack vs heap; debug with `-gcflags="-m"`.
6. Know preallocation, `strings.Builder`, `sync.Pool`, `GOGC`/`GOMEMLIMIT`.