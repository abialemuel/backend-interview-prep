# Rust — Backend Interview Prep

This section covers Rust for backend engineering interviews. It targets a backend engineer who already ships services in Go, Java, or a similar language and needs to reason clearly about ownership, borrowing, and Rust's concurrency model well enough to survive a senior-level interview — not to become a Rust expert overnight.

The language is covered at **Rust 1.97** (the current stable release, July 2026), on the **2024 edition** (stable since 1.85, February 2025; the edition itself is a compiler-facing opt-in for syntax changes, not a language fork — old and new editions interoperate freely in the same dependency graph). Where a detail is edition-specific, it's called out inline.

## Why Rust shows up in backend interviews now

Rust's pitch to a backend team is narrow and specific: **memory safety without a garbage collector**, enforced entirely at compile time via ownership and borrowing. That buys predictable, GC-pause-free latency and C/C++-level performance, at the cost of a steeper learning curve and slower iteration speed. A few reasons this now comes up in real interview loops rather than only academic ones:

- **Performance-critical infrastructure.** Cloudflare rewrote its NGINX-based proxy layer as Pingora (Rust), cutting CPU usage roughly 70% and memory usage roughly 67% at over a trillion requests/day. Discord moved specific high-throughput read-path services from Go to Rust and reported a 5x throughput improvement, driven largely by eliminating GC pauses and getting tighter control over memory layout. AWS uses Rust for parts of its virtualization and storage stack (Firecracker, parts of S3) where both safety and raw performance matter.
- **The pattern isn't "replace Go/Java everywhere."** It's Go or another GC'd language for the API layer and business logic, Rust for the narrow hot path or safety-critical subsystem underneath. Interviewers increasingly want to know you can reason about *when* that trade-off is worth it, not that you'd rewrite a CRUD service in Rust for fun.
- **WASM.** Rust compiles to WebAssembly cleanly and is the dominant language for WASM plugin systems and edge compute runtimes (Cloudflare Workers, Fastly Compute, Envoy/Proxy-Wasm filters). Backend roles touching edge compute or plugin sandboxes increasingly expect familiarity with Rust-to-WASM, even if the core service is written in something else.
- **Compile-time-enforced correctness is a genuinely different way of thinking.** Even engineers who will never ship Rust professionally are asked to reason about ownership and the borrow checker in interviews, because it tests whether a candidate understands *why* data races and use-after-free bugs happen at all — the mental model transfers back to reasoning about aliasing and lifetimes in any language.

None of this means Rust is "better than Go." It means Rust is the right tool for a specific, narrowing set of backend problems, and knowing where that boundary sits is itself the interview signal.

## Files in this section

| File | Description |
|------|--------------|
| `01-core-concepts.md` | Ownership, borrowing, lifetimes, the type system, structs/enums/pattern matching, `Option`/`Result`, traits and generics, modules, smart pointers (`Box`, `Rc`, `Arc`, `RefCell`). |
| `02-concurrency.md` | Fearless concurrency, `Send`/`Sync`, threads and `mpsc` channels, async/await, the Tokio runtime, `Arc<Mutex<T>>` patterns, deadlocks and blocking-in-async pitfalls. |
| `03-traits-error-handling-and-patterns.md` | Static vs dynamic dispatch (`dyn Trait`, monomorphization), error handling (`thiserror`, `anyhow`, `?`), iterators and zero-cost abstractions, builder pattern, `unsafe` Rust. |
| `04-testing-performance-and-ecosystem.md` | Testing and mocking without a built-in mock framework, `criterion` benchmarking, Cargo/crates.io, performance vs Go/C++, and honest guidance on when Rust is the wrong choice. |
| `05-interview-questions.md` | 20 questions grouped by difficulty (junior/senior/staff), with model answers, including "Rust vs Go" and a system design question. |

## Recommended reading order

1. `01-core-concepts.md` — ownership and the borrow checker are the whole game; nothing else in Rust makes sense until this clicks.
2. `02-concurrency.md` — how ownership rules extend to threads and `async`, and why the compiler catches data races other languages catch (if you're lucky) at 3am in production.
3. `03-traits-error-handling-and-patterns.md` — the idioms you're expected to produce in a live-coding round: trait design, error types, iterators.
4. `04-testing-performance-and-ecosystem.md` — where the "is this actually worth it" conversation happens; read this before an interview that asks you to justify Rust for a use case.
5. `05-interview-questions.md` — self-test; answer aloud, then compare with the model answer.

## Conventions used

- Code blocks are runnable Rust, edition 2024, and are written to be simple and obviously correct rather than clever — every non-trivial snippet should compile as written with `cargo build` on stable 1.97.
- "Idiomatic" means the way experienced Rust programmers actually write it — favoring `?` over manual `match` on `Result`, iterators over index loops, and `impl Trait` over generic type-parameter soup where either would work.
- Trade-offs are called out explicitly. This section is not a Rust advocacy piece; the goal is to help you decide *when* Rust is the right call for a backend service and defend that decision under questioning.

## Versions cheat sheet

| Version | Release | Notable |
|---------|---------|---------|
| 1.0 | May 2015 | Stability guarantee begins |
| 1.31 | Dec 2018 | 2018 edition; NLL (non-lexical lifetimes) borrow checker |
| 1.39 | Nov 2019 | `async`/`await` stabilized |
| 1.56 | Oct 2021 | 2021 edition; disjoint closure captures |
| 1.75 | Dec 2023 | `async fn` in traits (basic form) stabilized |
| 1.85 | Feb 2025 | 2024 edition stable; RPIT lifetime capture changes; unsafe `extern` blocks |
| 1.97 | Jul 2026 | Current stable at time of writing |

Anything described as "current" in this section implies Rust 1.97 on the 2024 edition.

Sources for the adoption figures above: [Cloudflare — replacing NGINX with Pingora](https://blog.cloudflare.com/how-we-scaled-pingora/), industry reporting on Discord's Go-to-Rust read-path migration, and AWS's public engineering blog on Firecracker and Rust usage.
