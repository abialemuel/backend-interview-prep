# Go — Backend Interview Prep

This section covers the Go programming language for backend engineering interviews. It targets a backend engineer who already programs in another language (e.g., Python, Java, TS) and wants to become fluent enough in Go to pass a senior-level interview.

The Go version covered is **Go 1.26** (released February 2026). Where a feature was introduced later than Go 1.0, the introducing version is noted inline. Features added in Go 1.24 (February 2025), Go 1.25 (August 2025), and Go 1.26 are explicitly called out where relevant (e.g., `range over int`, `range over func` previews, generic type aliases, SWAR string search, improved escape analysis, the new `testing/synctest` package, and the expanded `crypto/*` and `log/slog` APIs).

## Files in this section

| File | Description |
|------|-------------|
| `01-core-concepts.md` | Language fundamentals: programs, types, structs, pointers, functions, control flow, defer, slices, maps, strings, modules, tooling. |
| `02-concurrency.md` | Goroutines, GMP scheduler, channels, `select`, `sync`, `context`, pitfalls, patterns, memory model, atomics vs mutexes. |
| `03-interfaces-and-errors.md` | Interfaces, generics, nil-interface gotcha, error-as-value, `errors.Is/As/Join`, wrapping, panic/recover. |
| `04-testing-and-performance.md` | `testing`, table-driven tests, benchmarks, fuzzing, coverage, `pprof`, trace, race detector, escape analysis, allocation tips. |
| `05-interview-questions.md` | 28+ questions grouped by difficulty, with model answers. |

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
| 1.18 | Mar 2022 | Generics, fuzzing |
| 1.21 | Aug 2023 | `slices`, `maps`, `cmp` packages; `log/slog` |
| 1.22 | Feb 2024 | Per-iteration loop variables; `range over int` |
| 1.23 | Aug 2024 | `range over func` (experimental previews) |
| 1.24 | Feb 2025 | Generic type aliases; `crypto/*` enhancements; SWAR search |
| 1.25 | Aug 2025 | `crypto/hkdf`, `crypto/sha3` finalised; toolchain directive in `go.mod` |
| 1.26 | Feb 2026 | Mature `range over func`; improved escape analysis; `testing/synctest` promoted |

The exact feature set of Go 1.26 is reflected throughout; anything described as "current" implies the behavior of Go 1.26.