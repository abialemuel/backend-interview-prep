# Go — Backend Interview Prep

This section covers the Go programming language for backend engineering interviews. It targets a backend engineer who already programs in another language (e.g., Python, Java, TS) and wants to become fluent enough in Go to pass a senior-level interview.

The Go version covered is **Go 1.26** (released February 2026). Where a feature was introduced later than Go 1.0, the introducing version is noted inline. Features added in Go 1.24 (February 2025), Go 1.25 (August 2025), and Go 1.26 are explicitly called out where relevant (e.g., Swiss-table maps, generic type aliases, weak pointers, `testing.B.Loop`, container-aware `GOMAXPROCS`, `sync.WaitGroup.Go`, the stable `testing/synctest` package, the Green Tea garbage collector, `errors.AsType`, and the goroutine-leak profile).

## Files in this section

| File | Description |
|------|-------------|
| `01-core-concepts.md` | Language fundamentals: programs, types, structs, pointers, functions, control flow, defer, slices, maps, strings, modules, tooling. |
| `02-concurrency.md` | Goroutines, GMP scheduler, channels, `select`, `sync`, `context`, pitfalls, patterns, memory model, atomics vs mutexes. |
| `03-interfaces-and-errors.md` | Interfaces, generics, nil-interface gotcha, error-as-value, `errors.Is/As/Join`, wrapping, panic/recover. |
| `04-testing-and-performance.md` | `testing`, table-driven tests, benchmarks, fuzzing, coverage, `pprof`, trace, race detector, escape analysis, allocation tips. |
| `05-interview-questions.md` | 46 questions grouped by difficulty (junior/senior/staff), with model answers, including a 2025–2026 section. |

## Recommended reading order

1. `01-core-concepts.md` — establish the mental model of the language and its tooling.
2. `02-concurrency.md` — Go's defining feature; understand the scheduler and how channels and `sync` relate.
3. `03-interfaces-and-errors.md` — the other big idiomatic area; ties into generics added in Go 1.18.
4. `04-testing-and-performance.md` — Go's toolchain shines here; expect questions about benchmarks and `pprof`.
5. `05-interview-questions.md` — self-test; answer aloud then compare with the model answer.

Re-read each file's pitfalls section before the interview; those are the most common failure points candidates are quizzed on.

## Conventions used

- Code blocks are runnable Go (tested mentally against Go 1.26 unless noted).
- Where a behavior changed in a specific version, the version is inline, e.g. "since Go 1.22, `for` loop variables are per-iteration."
- "Idiomatic" is used deliberately — it means the way experienced Go programmers would write it, not just any working code.

## Versions cheat sheet

| Version | Release | Notable for this section |
|---------|---------|--------------------------|
| 1.18 | Mar 2022 | Generics, fuzzing, workspaces |
| 1.21 | Aug 2023 | `slices`, `maps`, `cmp` packages; `log/slog`; `min`/`max`/`clear` builtins; `toolchain` directive |
| 1.22 | Feb 2024 | Per-iteration loop variables; `range` over int; `math/rand/v2` |
| 1.23 | Aug 2024 | `range` over func — iterators, **stable**; `iter` and `unique` packages; GC-friendly timers |
| 1.24 | Feb 2025 | Swiss-table maps; generic type aliases; `weak` pointers; `os.Root`; `testing.B.Loop`; `tool` directive in `go.mod` |
| 1.25 | Aug 2025 | `testing/synctest` stable; container-aware `GOMAXPROCS`; `sync.WaitGroup.Go`; trace flight recorder; experimental `encoding/json/v2` and Green Tea GC |
| 1.26 | Feb 2026 | Green Tea GC on by default; `errors.AsType`; `new(expr)`; self-referential generic constraints; goroutine-leak profile (experimental); revamped `go fix` |

The exact feature set of Go 1.26 is reflected throughout; anything described as "current" implies the behavior of Go 1.26.