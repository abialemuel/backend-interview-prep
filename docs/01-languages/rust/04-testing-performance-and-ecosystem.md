# Testing, Performance, and Ecosystem

## Unit tests

Unit tests conventionally live in the same file as the code they test, inside a `#[cfg(test)]` module — that attribute means the module is compiled only when running `cargo test`, so test code never ships in the release binary.

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("divide by zero");
    }
    a / b
}

#[cfg(test)]
mod tests {
    use super::*; // pulls in everything from the enclosing module, including private items

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 2), 4);
    }

    #[test]
    #[should_panic(expected = "divide by zero")]
    fn panics_on_bad_input() {
        divide(1, 0);
    }

    #[test]
    fn returns_result() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("math is broken"))
        }
    }
}
```

A few things worth knowing cold: `use super::*` in the test module is why unit tests can reach private functions — they're compiled as part of the same module tree. A `#[test]` function can return `Result<(), E>` instead of panicking on failure, so you can use `?` inside a test the same way you would in application code. `cargo test` runs tests in parallel by default (one thread per test unless the test mutates shared global state, in which case `--test-threads=1` or a `Mutex`-guarded fixture is the usual fix).

## Integration tests

Files under a top-level `tests/` directory are compiled as **separate crates**, each linked only against your library's *public* API:

```
myservice/
├── src/
│   └── lib.rs
└── tests/
    └── api_test.rs
```

```rust
// tests/api_test.rs
use myservice::add;

#[test]
fn integration_add() {
    assert_eq!(add(2, 3), 5);
}
```

This is a deliberate forcing function: an integration test can only exercise what a real downstream consumer of your crate could call. If you find yourself wanting to reach a private function from `tests/`, that's a signal the function should either be public or the behavior should be tested through the public surface that actually uses it.

## Mocking — there is no built-in mock framework

Rust has no reflection-based mocking the way Mockito (Java) or `unittest.mock` (Python) work, and no interface-satisfaction-by-accident the way Go lets you swap in a test double without either side declaring intent. The mechanism is the same trait-based dependency injection you'd use in idiomatic Go: define a trait for the dependency, write a real implementation and a hand-rolled test double.

```rust
use std::cell::RefCell;

trait EmailSender {
    fn send(&self, to: &str, body: &str) -> Result<(), String>;
}

struct RealEmailSender;

impl EmailSender for RealEmailSender {
    fn send(&self, _to: &str, _body: &str) -> Result<(), String> {
        // would call out to a real SMTP/API provider here
        Ok(())
    }
}

struct FakeEmailSender {
    sent: RefCell<Vec<(String, String)>>,
}

impl EmailSender for FakeEmailSender {
    fn send(&self, to: &str, body: &str) -> Result<(), String> {
        self.sent.borrow_mut().push((to.to_string(), body.to_string()));
        Ok(())
    }
}

fn notify_user(sender: &impl EmailSender, user_email: &str) -> Result<(), String> {
    sender.send(user_email, "Welcome!")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sends_welcome_email() {
        let fake = FakeEmailSender { sent: RefCell::new(Vec::new()) };
        notify_user(&fake, "a@example.com").unwrap();
        assert_eq!(fake.sent.borrow().len(), 1);
    }
}
```

`RefCell` shows up here for exactly the reason covered in `01-core-concepts.md`: `send(&self, ...)` takes `&self` (the trait requires it, since callers shouldn't need `mut` access just to send an email), but the fake needs interior mutability to record calls it received. For anything beyond this level of hand-rolling, `mockall` generates mock structs from a trait definition via a derive-like macro (closest experience to gomock or Mockito), and `wiremock`/`httpmock` stand up an actual local HTTP server for testing code that calls out over HTTP.

## Benchmarking with `criterion`

`std` has an unstable `#[bench]` attribute gated behind the nightly compiler; on stable Rust, `criterion` is the standard tool, and it produces meaningfully more useful output than a bare timer — statistical analysis across many iterations, outlier detection, and automatic regression detection against the previous run.

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

This lives in a `benches/` directory, is declared in `Cargo.toml` with `[[bench]] name = "..." harness = false`, and pulls in `criterion` as a `dev-dependency`. `black_box` stops the compiler from constant-folding or eliminating the computation entirely, since an optimizer that can prove the result is unused (or always the same) would otherwise delete it. `cargo bench` runs it and prints a confidence interval, not a single number — treat a change smaller than the reported noise band as not proven.

## Doc tests

Code examples inside `///` doc comments are compiled and run as part of `cargo test`, each as its own tiny program linked against the crate's public API:

````rust
/// Adds two numbers together.
///
/// ```
/// assert_eq!(myservice::add(2, 2), 4);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
````

This means documentation examples cannot silently rot the way they can in most languages — if the API changes and the example no longer compiles or no longer holds, `cargo test` fails. It's a strong forcing function for keeping docs honest, and it's part of why crates.io documentation tends to be more trustworthy than average.

## Property-based testing and fuzzing

Example-based tests (`assert_eq!(add(2, 2), 4)`) only check the cases you thought to write. Two complementary techniques push further:

**Property-based testing** (`proptest`, or `quickcheck`) generates hundreds of randomized inputs and checks that an invariant holds for all of them, shrinking any failing case down to a minimal reproduction automatically:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn addition_is_commutative(a: i32, b: i32) {
        prop_assert_eq!(a.wrapping_add(b), b.wrapping_add(a));
    }
}
```

(`wrapping_add` is used here deliberately — the property under test is commutativity, not overflow behavior, and a random `i32` pair could otherwise trigger a debug-mode overflow panic unrelated to what's being tested.)

**Fuzzing** (`cargo-fuzz`, backed by libFuzzer) throws unstructured, mutated byte sequences at a target function and watches for panics, crashes, or sanitizer violations — the standard tool for hardening anything that parses untrusted input (request bodies, file formats, wire protocols):

```sh
cargo install cargo-fuzz
cargo fuzz init
cargo fuzz run fuzz_target_1
```

Both are worth naming in a senior interview specifically for parser or deserialization code — "how would you gain confidence this input-handling code has no edge-case panics" is a real question, and "property tests for the invariants, a fuzz target for the untrusted-input surface" is the expected shape of the answer.

## Profiling — no built-in equivalent to `go tool pprof`

This is a genuine ecosystem gap worth naming honestly: Go ships `net/http/pprof` and `go tool pprof` in the standard toolchain, giving CPU, heap, goroutine, and contention profiles with zero added dependencies. Rust has no equivalent shipped with `cargo` or `rustc` — you reach for external tools:

| Need | Tool |
|------|------|
| CPU flamegraph | `cargo flamegraph` (wraps Linux `perf` or `dtrace`) |
| Heap allocation profiling | `dhat-rs` (Valgrind DHAT-compatible, works without Valgrind installed) or `heaptrack` |
| General memory/leak checking | `valgrind --tool=massif` / `valgrind --tool=memcheck` |
| Undefined behavior detection (in `unsafe` code) | `cargo miri test` |
| Async task/runtime introspection | `tokio-console` (live view of task states, poll times, contention — the closest thing to `go tool trace` for async Rust) |

The upside once you've wired one of these in: because there's no GC and no interpreter overhead, a CPU flamegraph in Rust points almost entirely at *your* code and the libraries you chose — there's no "40% of samples are in the garbage collector" category to reason around, which is a common early confusion for engineers profiling Rust for the first time coming from a GC'd language.

## Cargo and the crates.io ecosystem

`Cargo.toml` sections you'll see in any real service: `[package]`, `[dependencies]`, `[dev-dependencies]` (test/bench-only), `[build-dependencies]`, `[profile.release]` (optimization level, LTO, panic strategy), and `[workspace]` for multi-crate repos. `Cargo.lock` pins exact resolved versions; it's committed for binaries (reproducible builds) and typically *not* committed for libraries (so downstream consumers resolve their own compatible version set).

| Command | Purpose |
|---------|---------|
| `cargo build` / `cargo build --release` | Compile (debug / optimized) |
| `cargo run` | Build and run the binary |
| `cargo test` | Run unit tests, integration tests, and doc tests |
| `cargo bench` | Run benchmarks |
| `cargo add <crate>` | Add a dependency to `Cargo.toml` |
| `cargo clippy` | Lint beyond the compiler's own warnings |
| `cargo fmt` | Canonical formatting |
| `cargo doc --open` | Build and open API documentation |
| `cargo tree` | Print the resolved dependency graph |
| `cargo audit` | Check dependencies against the RustSec vulnerability database |

Crates.io uses semantic versioning, and the default `^1.2.3` requirement means "any version compatible with 1.2.3 under semver" (`>=1.2.3, <2.0.0`) — the same MVS-adjacent trust-the-ecosystem model as Go modules, but with a caret-range default rather than Go's minimum-version selection.

Ecosystem crates a backend engineer should recognize on sight: **tokio** (async runtime), **axum** or **actix-web** (HTTP frameworks), **sqlx** (async, compile-time-checked SQL queries) or **diesel** (compile-time-checked ORM-style query builder), **serde** (the near-universal serialization framework — almost every crate that has a wire format uses it), **tonic** (gRPC), **tracing** (structured logging and distributed tracing), and **thiserror**/**anyhow** (error handling, covered in `03-traits-error-handling-and-patterns.md`).

## Performance characteristics vs Go and C++

| Aspect | Go | Rust | C++ |
|--------|----|------|-----|
| Memory management | Tracing GC | Ownership, compile-time freed (RAII via `Drop`) | Manual, or RAII by convention (not compiler-enforced) |
| Data-race safety | Runtime detector only, opt-in (`-race`) | Compile-time guarantee in safe code | Not enforced |
| Compile speed | Fast | Slow, improving release over release | Slow to very slow |
| Tail latency | Good, with rare GC-pause risk under memory pressure | Excellent — no GC pause to worry about at all | Excellent |
| Binary/memory overhead per value | Some (interface boxing, GC bookkeeping) | Minimal — comparable to a C struct | Minimal |
| Cold start | Fast | Fast — no VM or JIT warmup | Fast |
| Concurrency model | Built into the runtime (goroutines, scheduler) | Opt-in via the ecosystem (`tokio`, etc.) | Opt-in via libraries |
| Learning curve | Low | High | High |

The headline trade-off: Rust removes garbage collection entirely, so there is no stop-the-world pause, ever — this is the concrete reason Discord and Cloudflare cite for moving specific services to Rust, since a GC pause of even a couple of milliseconds is a real problem at their p99/p999 latency targets. Go's modern GC is genuinely good and sub-millisecond in most cases, but "good and rare" isn't the same guarantee as "structurally cannot happen." Against C++, Rust's raw performance is comparable (both compile through LLVM to native code) — the difference is that Rust gets there with memory safety guaranteed by the compiler instead of by programmer discipline, at the cost of the borrow checker's upfront design tax and, occasionally, small explicit costs (bounds checks on slice indexing, refcounting on `Rc`/`Arc`) that you can strip out with `unsafe` once profiling proves they matter.

## When Rust is the wrong tool

This section deserves the same honesty the rest of this repository applies to every other trade-off — Rust is not a strictly-better Go, it's a different point on the safety/performance/velocity curve, and picking it has real costs:

- **Compile times slow the iteration loop.** A large Rust codebase's clean build (and often incremental build) is measurably slower than the equivalent Go build; this compounds in CI, in local `cargo check` loops, and in onboarding — waiting on the compiler is a daily tax that most Go and even many Java/Python shops don't pay at the same scale. Incremental compilation, `sccache`, and splitting a monorepo into smaller crates all help, but none make it disappear.
- **The hiring pool is smaller and more expensive.** Go, Java, Python, and TypeScript backend engineers are abundant; experienced Rust engineers are not, and the ramp-up time for a competent backend engineer to become *productive* in Rust (not just able to make it compile) is genuinely longer than the equivalent ramp in Go, because ownership and the borrow checker are new mental models, not new syntax over familiar concepts.
- **For typical CRUD backend work, Go usually wins.** A service that receives JSON, does a database round trip measured in milliseconds, and returns JSON is bottlenecked on the database and the network, not on your language's memory management. Rust's safety guarantees buy you very little there, while its compile times and learning curve cost you real velocity. This is the honest, unglamorous majority case for backend services.
- **The library ecosystem, while much stronger than a few years ago, still has gaps** in some niches compared to decades-deep Java/Python ecosystems — vendor SDKs, legacy protocol support, and some enterprise integrations are more likely to have a first-class Go or Java client than a first-class Rust one.
- **Design work shifts earlier.** Satisfying the borrow checker for a genuinely shared, mutable data structure sometimes requires committing to a data-ownership design up front that a GC'd language would let you defer or refactor cheaply later. That's often a *good* forcing function for correctness, but it is a real cost in a fast-moving early-stage codebase where the shape of the data model is still changing weekly.

**Where Rust earns its cost**: a narrow, well-understood hot path where GC pauses or memory overhead are the actual measured bottleneck (a proxy, a serialization layer, a matching engine, an ingestion pipeline); a service where predictable p99/p999 latency is a hard requirement, not a nice-to-have; CPU- or memory-bound workloads where profiling — not intuition — shows the language runtime itself is the constraint; anything targeting WASM or an embedded/constrained environment where a GC runtime isn't an option at all. The Discord and Cloudflare pattern — Go (or another GC'd language) for the API layer and business logic, Rust for the specific subsystem underneath that needed it — is the realistic adoption model for most companies, not "rewrite everything."

**The interview-ready framing**: ask whether the service exists to move business logic quickly and safely, or to save CPU/memory cost and guarantee tail latency at scale. The former points at Go (or whatever your team already runs); the latter is where Rust's cost is worth paying.
