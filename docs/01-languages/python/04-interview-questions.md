# Python Interview Questions

Answer each question aloud before reading the model answer. For senior roles, explain the trade-off and the production consequence, not only the definition.

## Core Python

### Q1: What does it mean that everything in Python is an object?

**Answer:** Names refer to objects, and values such as integers, functions, and classes have identity, type, and behavior. Assignment binds another name to an object; it does not copy it. This matters most for mutable objects, where mutation through one reference is visible through other references.

### Q2: What is the difference between `is` and `==`?

**Answer:** `is` checks object identity; `==` checks value equality through `__eq__`. Use `is None` for the `None` singleton. Do not use identity for ordinary value comparisons, even when small-integer or string interning makes it appear to work.

### Q3: Why are mutable default arguments dangerous?

**Answer:** Default expressions are evaluated once when the function is defined, so a default list or dictionary is shared by calls. Use `None` as the default and create the mutable value inside the function, or use a suitable immutable sentinel.

### Q4: Explain LEGB name lookup.

**Answer:** Python resolves a name through Local, Enclosing, Global, then Built-in scopes. `nonlocal` assigns to an enclosing function variable; `global` assigns to a module-level name. Blocks such as `if` and `for` do not create a new local scope.

### Q5: What is a closure?

**Answer:** A closure is a function together with references to names from its enclosing lexical scope. Closures are useful for callbacks and decorators, but captured mutable state can make behavior harder to reason about. Use `nonlocal` when rebinding a captured name.

### Q6: What does a decorator do?

**Answer:** A decorator receives a function or class and returns a replacement, commonly a wrapper. Decorators execute when the `def` statement is evaluated, usually at import time. Use `functools.wraps` so the wrapper preserves metadata such as the original function's name and docstring.

### Q7: How are iterators and generators different?

**Answer:** An iterator implements `__iter__` and `__next__`; a generator is a convenient iterator created by a function containing `yield`. Generators compute values lazily, which can reduce peak memory, but they are usually consumed once and cannot be indexed like a list.

### Q8: What does a context manager provide?

**Answer:** It brackets setup and cleanup around a `with` block, usually through `__enter__`/`__exit__` or `__aenter__`/`__aexit__`. Cleanup runs when the block exits, including when it raises. Return true from `__exit__` only when intentionally suppressing that exception.

### Q9: What is the difference between a shallow copy and a deep copy?

**Answer:** A shallow copy creates a new outer container but keeps references to nested objects. A deep copy recursively copies objects where possible. Deep copying can be expensive or incorrect for resources such as sockets, locks, and database connections; prefer explicit copying of the state you own.

### Q10: What is a dataclass, and when would you use one?

**Answer:** `dataclasses.dataclass` generates common methods such as `__init__` and `__repr__` for data-focused classes. Use `frozen=True` when instances should be immutable and `slots=True` when reducing per-instance memory is useful. A frozen dataclass is only shallowly immutable if it contains mutable fields.

### Q11: How does Python's exception model differ from checked exceptions?

**Answer:** Python exceptions are unchecked: a function's signature does not declare which exceptions it may raise. Catch exceptions at a boundary where the program can recover or add useful context. Avoid broad `except Exception` blocks that hide programming errors, and preserve causes with `raise NewError(...) from exc`.

### Q12: What do type hints guarantee at runtime?

**Answer:** Usually nothing by themselves. Type hints support readers, IDEs, and static checkers such as mypy or Pyright, but Python does not enforce them automatically. Use runtime validation at untrusted boundaries, for example Pydantic models for request data.

## Runtime and concurrency

### Q13: What is the GIL?

**Answer:** In the standard CPython build, the Global Interpreter Lock normally allows only one thread at a time to execute Python bytecode in an interpreter. Threads can still help I/O-bound work because blocking I/O releases execution, but CPU-bound Python code generally needs processes for parallelism. Python 3.13 introduced optional free-threaded builds; their compatibility and performance trade-offs still need evaluation.

### Q14: When would you choose threads, processes, or asyncio?

**Answer:** Use threads to overlap blocking I/O from synchronous libraries, processes for CPU-bound parallel work or stronger isolation, and asyncio for high concurrency when the I/O stack is async-compatible. Choose based on workload, library support, resource cost, and cancellation needs rather than assuming one model is universally faster.

### Q15: What does `await` do?

**Answer:** It suspends the current coroutine until an awaitable produces a result, allowing the event loop to run other ready tasks. `await` does not automatically make a blocking function non-blocking; calling synchronous network or database code inside an async endpoint still blocks that worker's event loop.

### Q16: What is the difference between a coroutine and a task?

**Answer:** Calling an `async def` function creates a coroutine object but does not schedule it. A task wraps a coroutine and schedules it on an event loop. Await the coroutine directly when sequential execution is intended; create tasks or use `TaskGroup` when work should run concurrently.

### Q17: Why is `asyncio.TaskGroup` useful?

**Answer:** It provides structured concurrency: child tasks are awaited before the group exits, and a failing child cancels its siblings. Exceptions may be raised as an `ExceptionGroup`. This makes task lifetime and failure handling clearer than creating detached tasks and hoping they finish.

### Q18: How should blocking work be called from an async endpoint?

**Answer:** Prefer an async-compatible library. For unavoidable blocking I/O, `asyncio.to_thread()` can move the call off the event-loop thread. CPU-bound work generally belongs in a process pool or separate worker service; moving it to a thread does not usually provide Python bytecode parallelism in standard CPython.

### Q19: Does a single-threaded event loop eliminate race conditions?

**Answer:** No. Coroutines can interleave whenever they suspend, so a check-then-act sequence spanning an `await` can observe stale state. Use an `asyncio.Lock` for in-process coordination or, for shared business state, enforce the invariant atomically in the database or another shared system.

### Q20: What are `contextvars` for?

**Answer:** They carry context-local values such as trace IDs across asynchronous calls without relying on thread-local storage. A task receives a copy of the current context when created. They are useful for observability metadata, not as a replacement for explicit business parameters.

### Q21: How should cancellation be handled in async code?

**Answer:** Let `asyncio.CancelledError` propagate after performing necessary cleanup in `finally` or async context managers. Swallowing cancellation can prevent timeouts and shutdown from working. Keep cleanup bounded because servers cannot wait forever for a cancelled task.

### Q22: What is the difference between concurrency and parallelism?

**Answer:** Concurrency is organizing multiple tasks so they make progress over overlapping periods; parallelism is executing at the same instant on multiple cores. Asyncio provides concurrency on an event loop, while multiple processes or a free-threaded runtime can provide CPU parallelism.

## Web, APIs, and deployment

### Q23: What problem does ASGI solve compared with WSGI?

**Answer:** WSGI defines a synchronous request/response callable. ASGI supports async applications and connection types such as HTTP, WebSockets, and lifespan startup/shutdown events. Uvicorn is an ASGI server; FastAPI and Starlette are ASGI frameworks.

### Q24: What happens to an ASGI request in a FastAPI service?

**Answer:** A server such as Uvicorn accepts the network connection and translates it into ASGI messages. Middleware wraps the app; routing selects an endpoint; dependencies and validation run; the endpoint returns a value or response; and the framework serializes it back through ASGI. The server and framework have distinct responsibilities.

### Q25: What does Pydantic do in FastAPI?

**Answer:** Pydantic validates and parses data into typed models, commonly for request bodies and response schemas. Validation failures become client errors at the framework boundary. Validation is not authorization: the application must still check permissions and enforce business invariants.

### Q26: What is FastAPI dependency injection useful for?

**Answer:** Dependencies declare reusable request-scoped or application-scoped needs such as authentication, database sessions, and service objects. They make composition and test overrides convenient. Be explicit about resource lifetime so a request does not accidentally retain a session or connection beyond its intended scope.

### Q27: How should startup and shutdown resources be managed?

**Answer:** Use the framework's lifespan mechanism to create shared clients and connection pools at startup, then close them at shutdown. Avoid creating a new global connection pool per request or relying on destructors for asynchronous cleanup. In a multi-worker deployment, each process normally owns its own pool.

### Q28: How would you deploy an ASGI app in containers?

**Answer:** A common starting point is one application process per container, with the orchestrator managing replicas. Bind to `0.0.0.0`, install from a committed lockfile, run as a non-root user, inject secrets at runtime, and configure resource limits and graceful termination. Multiple workers per container can work, but account for each process's memory and connection pools.

### Q29: How should worker count be chosen?

**Answer:** Measure throughput, latency, CPU, memory, and connection usage under realistic load. Each worker is a separate process with its own event loop and typically its own application state and pools. A fixed workers-equal-CPU-cores rule can oversubscribe memory or connections, especially in containers with quotas.

### Q30: What should liveness and readiness checks mean?

**Answer:** Liveness should indicate whether the process is functioning and avoid depending on every downstream service. Readiness should indicate whether the instance can currently take traffic; bounded dependency checks can be appropriate. If a shared database outage makes every pod fail liveness, restart loops can worsen the incident.

### Q31: How do you handle database migrations during deployment?

**Answer:** Run migrations as a controlled release step or one-off task, not independently from every application worker's startup hook. During rolling deploys, use an expand-and-contract approach: first add backward-compatible schema, deploy code that can use it, then remove obsolete schema in a later release.

### Q32: How should forwarded headers be handled behind a proxy?

**Answer:** Configure the server or middleware to trust forwarded headers only from known proxy addresses or networks. If arbitrary clients can supply trusted `X-Forwarded-For` or `X-Forwarded-Proto` values, they can spoof client addresses or schemes and affect security decisions and generated URLs.

### Q33: What is the purpose of graceful shutdown?

**Answer:** On termination, the server should stop accepting new work, allow in-flight requests a bounded completion period, and close application resources. The orchestrator's termination grace period must exceed the server's graceful-shutdown timeout. Long-running jobs should use a durable queue or worker system rather than depend on a web process surviving termination.

### Q34: What is the difference between authentication and authorization?

**Answer:** Authentication establishes who the caller is; authorization decides what that identity may do to a resource. A valid token does not automatically grant access to every record. Enforce authorization on the server for each relevant operation and resource.

## Testing, performance, and operations

### Q35: What belongs in unit, integration, and end-to-end tests?

**Answer:** Unit tests cover fast isolated logic; integration tests exercise real boundaries such as a database or broker; end-to-end tests validate a small number of critical user flows through a deployed system. Keep most tests fast, but do not mock away the database behavior or protocol semantics that the test is meant to verify.

### Q36: How do you test a FastAPI endpoint without starting a server?

**Answer:** Use `TestClient` for synchronous tests or HTTPX `AsyncClient` with `ASGITransport` for async tests. Override dependencies to supply fakes or test resources. This exercises the ASGI application in-process without opening a listening socket.

### Q37: How would you find a Python performance bottleneck?

**Answer:** Establish a representative workload and measure first. Use a profiler such as `cProfile` for deterministic call costs or `py-spy` for low-overhead sampling of a live process. Determine whether the constraint is CPU, event-loop blocking, database latency, network waits, allocation, or lock contention before changing code.

### Q38: How would you investigate an event loop that stalls in production?

**Answer:** Check loop-lag and request-latency metrics, inspect traces and logs, and use sampling or stack dumps to find synchronous calls or long CPU sections on the loop thread. Replace blocking I/O with async clients or offload unavoidable calls; move CPU-heavy work to a separate execution path. Re-measure after the change.

### Q39: What does a reproducible Python build require?

**Answer:** Declare supported Python versions and dependencies in project metadata, commit the lockfile, and install from it in CI and image builds. Pin or control the runtime base image and build inputs as the team's release policy requires. A lockfile cannot guarantee identical behavior across incompatible operating systems, Python versions, or native libraries by itself.

### Q40: What should a CI pipeline check for a Python backend?

**Answer:** Install locked dependencies, run formatting and lint checks, run static type checks at the chosen strictness, and run unit and integration tests. Add dependency and image vulnerability scanning where appropriate. Keep deployment artifacts tied to the tested revision and promote the same artifact between environments.

### Q41: How should secrets and configuration be handled?

**Answer:** Keep environment-specific configuration outside the application image and load secrets from the deployment platform's secret mechanism. Do not commit secrets or pass them through Docker build arguments, where they can persist in build metadata or layers. Validate required settings at startup and avoid logging secret values.

### Q42: What is your approach to diagnosing a failing deployment?

**Answer:** Identify the failing stage first: image build, process startup/import, readiness, traffic handling, or dependency access. Compare the deployed image and configuration with the last healthy revision; inspect structured logs, events, and health-check results; then reproduce with the same runtime and environment shape. Roll back when user impact is ongoing, and follow with a targeted fix and a regression check.

---

*Previous: [03-testing-and-tooling.md](03-testing-and-tooling.md) — pytest, profiling, and the modern Python toolchain.*
