# Python — Backend Interview Prep

This section covers Python for backend engineering interviews, written for a backend engineer whose primary language is something else (e.g., Go) and who needs to be fluent enough in Python to pass a senior-level interview — including the async/web stack (ASGI, Uvicorn, FastAPI, Pydantic) that modern Python services are built on.

The Python version covered is **3.14** (released October 2025), the current stable release. Features are marked with introducing versions inline (3.11, 3.12, 3.13, 3.14). Python 3.15 is due October 2026. Where a feature is new in 3.13/3.14 (free-threaded mode, the JIT, deferred annotations), it is explicitly called out — interviewers increasingly ask about the GIL's future, and "Python can't do parallelism" is now a partially wrong answer.

## Files in this section

| File | Description |
|------|-------------|
| `01-core-concepts.md` | Data model, mutability, everything-is-an-object, functions/decorators/closures, generators, context managers, OOP and dunder protocols, dataclasses, enums, exceptions, typing system, packaging and environments. |
| `02-asyncio-asgi-and-web.md` | Event loop, coroutines, tasks, `TaskGroup`, timeouts, async pitfalls (blocking calls), WSGI → ASGI, ASGI spec internals, Uvicorn, Starlette, FastAPI, Pydantic v2, dependency injection, middleware, streaming/SSE, background jobs, async DB access. |
| `03-testing-and-tooling.md` | `pytest` in depth (fixtures, parametrize, mocking, async tests), test strategy, profiling (`cProfile`, `py-spy`), memory, performance optimization, `uv`, `ruff`, `mypy`/`pyright`, CI setup. |
| `04-interview-questions.md` | 40+ questions grouped junior/senior/staff with model answers, including GIL/free-threading, ASGI, and 2025–2026 topics. |

## Recommended reading order

1. `01-core-concepts.md` — the mental model: object semantics, mutability, protocols, typing.
2. `02-asyncio-asgi-and-web.md` — the defining skill for Python backend roles; most senior Python interviews probe async understanding and the ASGI stack.
3. `03-testing-and-tooling.md` — pytest is the assumed default; tooling questions (uv, ruff, mypy) signal "works in a modern team."
4. `04-interview-questions.md` — self-test; answer aloud, then compare.

## Why this matters for backend interviews

- Python backend interviews assume **async literacy**: why the event loop stalls, what ASGI is, when async helps and when it doesn't.
- The **GIL story changed**: 3.13 shipped an experimental free-threaded build; 3.14 made it officially supported. Knowing this beats candidates who repeat 2015 folklore.
- **FastAPI + Pydantic** is the dominant service framework; knowing its dependency injection and validation model is table stakes for product-backend roles.
- **Tooling modernized**: `uv` replaced pip/poetry for many teams, `ruff` replaced flake8/isort/black in many repos. Mentioning current tooling reads as "practicing engineer," not "read a tutorial."

## Conventions used

- Code blocks are runnable Python 3.12+ unless a version marker says otherwise.
- Version notes are inline: "since 3.12", "3.13+ (free-threaded build)".
- "Idiomatic" means what experienced Python engineers write: snake_case, type hints, comprehensions over loops, context managers over manual cleanup.

## Versions cheat sheet

| Version | Release | Notable for this section |
|---------|---------|--------------------------|
| 3.11 | Oct 2022 | Exception groups + `except*`, `TaskGroup`, `asyncio.timeout`, self-type, 10–60% CPython speedup ("faster CPython") |
| 3.12 | Oct 2023 | PEP 695 type-parameter syntax (`def f[T](...)`), per-interpreter GIL (PEP 684), `frozen` dataclass perf, eager task factory, `itertools.batched` |
| 3.13 | Oct 2024 | Experimental **free-threaded build** (PEP 703, `--disable-gil`), experimental JIT (PEP 744), REPL overhaul, `typing.TypeIs`, copy.replace, removal of 19 dead batteries (PEP 594) |
| 3.14 | Oct 2025 | Free-threaded build **officially supported** (PEP 779), deferred evaluation of annotations (PEP 649/749), template strings t-strings (PEP 750), multiple interpreters in stdlib (`concurrent.interpreters`), improved error messages |

Exact 3.14 behavior is described as "current"; anything version-specific is marked inline.
