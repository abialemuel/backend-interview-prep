# Asyncio, ASGI, and the Web Stack

This is the file that matters most for senior Python backend interviews. Modern Python services are built on `asyncio` + ASGI + FastAPI (or Starlette directly), and most senior-level questions probe whether you understand what happens under the `await`.

## The asyncio model in one paragraph

`asyncio` runs your code on a **single thread** inside an **event loop**. Concurrency comes from **coroutines** — functions marked `async def` that can pause at `await` points and let other coroutines run while they wait for I/O. It is cooperative scheduling, not preemptive: a coroutine only yields control when it hits `await` (or explicitly yields). There is no preemption like goroutines — if your coroutine runs CPU-heavy code without awaiting, **the entire loop stalls and every other request waits**.

Compare with Go: goroutines are preemptively scheduled across multiple OS threads by the runtime; Python coroutines are cooperatively scheduled on one thread (by default). This is the single most important difference to articulate in interviews.

## Coroutines, awaitables, tasks

Three kinds of awaitables:

1. **Coroutines** — `async def` functions (or objects returned by calling them).
2. **Tasks** — coroutines wrapped and scheduled on the loop (`asyncio.create_task`).
3. **Futures** — low-level awaitable result holders (you rarely create these directly).

```python
import asyncio

async def fetch_data() -> dict:
    await asyncio.sleep(1)  # simulates I/O
    return {"id": 1}

async def main():
    # WRONG pattern (sequential — 2 seconds):
    a = await fetch_data()
    b = await fetch_data()

    # RIGHT pattern (concurrent — 1 second):
    t1 = asyncio.create_task(fetch_data())
    t2 = asyncio.create_task(fetch_data())
    a = await t1
    b = await t2

    # Or gather:
    a, b = await asyncio.gather(fetch_data(), fetch_data())
```

**Critical distinction**: calling `fetch_data()` does **not** run it — it creates a coroutine object. Nothing executes until it's awaited or wrapped in a task. Forgetting to await is a classic bug:

```python
async def handler():
    fetch_data()  # BUG: coroutine created, never run, RuntimeWarning
```

**`asyncio.gather` vs `TaskGroup`**:

```python
# gather (older): returns list of results; on exception, behavior depends on return_exceptions
results = await asyncio.gather(f1(), f2(), return_exceptions=False)

# TaskGroup (3.11+, preferred): structured concurrency
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(f1())
    t2 = tg.create_task(f2())
# both awaited when block exits; if either raises, the group cancels the other
# and raises ExceptionGroup
```

`TaskGroup` is **structured concurrency**: tasks are guaranteed to be awaited before the `async with` exits, and failure of one child cancels siblings. This mirrors Go's `errgroup`. Use `TaskGroup` in new code.

**Timeouts** (3.11+):

```python
async with asyncio.timeout(5.0):
    await slow_operation()  # raises TimeoutError if it exceeds 5s

# per-call:
result = await asyncio.wait_for(slow_operation(), timeout=5.0)
```

**Shielding** — protect a task from cancellation:

```python
await asyncio.shield(critical_cleanup())  # continues even if parent cancelled
```

## The event loop: what actually happens

The event loop is a scheduler that:

1. Maintains queues of ready callbacks and timers.
2. Uses a **selector** (`epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows) to wait on file descriptors with I/O readiness.
3. When a coroutine awaits an I/O operation, the underlying machinery registers the FD with the selector, and the coroutine's `Future` is marked pending.
4. When the FD becomes ready, the loop resumes the coroutine by calling `coro.send()` on it.

You don't manage this directly, but you must know:

- **One loop per thread**, and the loop runs on the thread that started it.
- **Blocking calls freeze everything.** `time.sleep(5)`, `requests.get(...)`, heavy CPU work, or a synchronous DB driver inside a coroutine blocks the loop — all requests on that worker hang.
- **How to run blocking code without freezing the loop**:

```python
# In a thread (I/O-bound blocking library):
result = await asyncio.to_thread(sync_db_call, query)  # 3.9+

# In a process pool (CPU-bound):
from asyncio import get_running_loop
from concurrent.futures import ProcessPoolExecutor
loop = get_running_loop()
result = await loop.run_in_executor(ProcessPoolExecutor(), cpu_fn, arg)
```

Interview rule: **`async def` endpoint + synchronous blocking call = production incident.** This is the most common real-world asyncio bug and a favorite interview probe.

## What `await` does mechanically

`await` compiles to yields on a generator-like protocol:

1. `await x` calls `x.__await__()`.
2. The coroutine suspends, returning control to the loop with a `Future`.
3. The loop resumes it when the future resolves, via `coro.send(value)`.

You can demonstrate coroutine machinery manually:

```python
async def coro():
    return 42

c = coro()
try:
    c.send(None)          # starts execution
except StopIteration as e:
    print(e.value)        # 42 — return value comes via StopIteration
```

Rarely needed in practice, but knowing it separates "used async" from "understands async."

## Context variables (async-safe context)

`contextvars` replaces thread-locals for async code — task-local storage for request IDs, trace context, auth:

```python
import contextvars

request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id")

async def handle():
    request_id.set("abc-123")
    await child()  # child sees the value; each Task copies the context

async def child():
    print(request_id.get())  # "abc-123"
```

Every task gets a **copy** of the context at creation time — changes in the parent after task creation are not visible to the task. This is how observability libraries (OpenTelemetry) propagate trace context across awaits.

## WSGI → ASGI: why ASGI exists

**WSGI** (Web Server Gateway Interface, ~2003) is the synchronous contract between web servers and Python apps: one callable, one request, one thread. `Flask`, `Django` (classic), `gunicorn` sync workers are WSGI.

```python
# WSGI app: synchronous callable
def application(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"Hello"]
```

WSGI cannot express: WebSockets, long-lived connections, server-sent events, async I/O — it's inherently one-thread-per-request, one request per call.

**ASGI** (Asynchronous Server Gateway Interface, ~2018) generalizes WSGI to async: the app is a **single coroutine** receiving a stream of scoped **messages** over three channels — `receive` (incoming events), `send` (outgoing events), and a `scope` dict (connection metadata). It supports:

- HTTP requests (same as WSGI)
- **WebSockets** (bidirectional, long-lived)
- **Lifespan** protocol (startup/shutdown hooks — where you open DB pools)
- Multiple concurrent requests per worker on one thread

```python
# Raw ASGI app — no framework
async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        while True:
            message = await receive()
            if message["type"] == "lifespan.startup":
                await startup()          # open DB pool etc.
                await send({"type": "lifespan.startup.complete"})
            elif message["type"] == "lifespan.shutdown":
                await shutdown()
                await send({"type": "lifespan.shutdown.complete"})
                return

    elif scope["type"] == "http":
        await receive()                  # the request body event
        body = b'{"ok": true}'
        await send({
            "type": "http.response.start",
            "status": 200,
            "headers": [(b"content-type", b"application/json")],
        })
        await send({"type": "http.response.body", "body": body})
```

**ASGI message flow for HTTP**: server → app `http.request` events (body chunks), app → server `http.response.start` (status + headers) then `http.response.body` (possibly multiple chunks — that's how streaming works).

**The stack layers**:

| Layer | Examples | Role |
|-------|----------|------|
| Server | Uvicorn, Hypercorn, Granian | Speaks HTTP/WebSocket, translates to ASGI messages, manages the event loop and workers |
| Framework | Starlette, FastAPI, Litestar, Django (ASGI mode) | Routing, middleware, request/response objects, DI |
| Validation | Pydantic v2, msgspec | Schema, (de)serialization, validation |

**Uvicorn** specifics worth knowing:

- Runs an event loop per worker process. Typically deployed as `uvicorn --workers N app:app` or behind gunicorn with uvicorn workers (older pattern; uvicorn's own multi-worker mode is now standard).
- One worker = one process = one loop = one core. Scale = worker count (CPU cores) × loop efficiency (I/O-bound requests).
- `--loop uvloop` (libuv-based loop) gives a significant throughput boost over the default asyncio loop — standard in production.

## Starlette: the foundation

FastAPI is built on **Starlette**, which provides the actual ASGI machinery. Knowing Starlette helps because senior interviews often ask what FastAPI adds on top:

- `Route`, `Mount` (sub-apps), `WebSocketRoute`
- Middleware system (pure ASGI middleware: `app = Middleware(SomeMiddleware, ...)` wrap)
- `Request`/`Response` objects, streaming responses, background tasks
- Test client (`TestClient` wraps `httpx` + ASGI transport — no real server needed)

```python
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route

async def homepage(request):
    return JSONResponse({"hello": "world"})

app = Starlette(routes=[Route("/", homepage, methods=["GET"])])
```

## FastAPI: the framework layer

FastAPI = Starlette + Pydantic + dependency injection + OpenAPI generation.

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel, Field

app = FastAPI()

class BookingRequest(BaseModel):
    trip_id: str
    from_stop: str
    to_stop: str
    seats: int = Field(gt=0, le=10)

class BookingResponse(BaseModel):
    booking_id: str
    status: str

async def get_booking_service() -> "BookingService":
    # resolves per-request; can open transactions, load auth context
    return BookingService(pool=app.state.pool)

@app.post("/bookings", response_model=BookingResponse, status_code=201)
async def create_booking(
    req: BookingRequest,
    service: "BookingService" = Depends(get_booking_service),
):
    try:
        return await service.book(req)
    except TripFullError as e:
        raise HTTPException(status_code=409, detail="no seats available") from e
```

**What to know at senior level:**

- **Request body validation happens before your handler runs.** Pydantic rejects invalid payloads with a 422 and structured error details. Your handler only sees validated types.
- **`Depends` is a DI system**: dependencies can be async, can be cached per-request (`use_cache=True` default), and can be nested. Auth, DB sessions, pagination all become dependencies.
- **`response_model`** filters/validates the output — prevents leaking internal fields (e.g., password hashes) accidentally.
- **Lifespan** (replaces `@app.on_event`):

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.pool = await asyncpg.create_pool(dsn)
    yield                     # app runs
    await app.state.pool.close()

app = FastAPI(lifespan=lifespan)
```

- **Pydantic v2 is Rust-backed** (`pydantic-core`): validation is 5–50x faster than v1. Know `model_validate`, `model_dump(mode="json")`, `field_validator`, `model_config = ConfigDict(frozen=True)`.

```python
from pydantic import BaseModel, ConfigDict, field_validator

class SeatHold(BaseModel):
    model_config = ConfigDict(frozen=True)   # immutable, hashable

    trip_id: str
    seats: int

    @field_validator("seats")
    @classmethod
    def positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("seats must be positive")
        return v
```

## Middleware: ASGI vs framework level

Two middleware styles:

```python
# 1. Framework-level (Starlette/FastAPI BaseHTTPMiddleware)
from starlette.middleware.base import BaseHTTPMiddleware

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        response.headers["x-process-time"] = f"{time.perf_counter() - start:.4f}"
        return response

# 2. Pure ASGI middleware (faster, sees raw messages)
class TimingASGIMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        start = time.perf_counter()
        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                message["headers"].append(
                    (b"x-process-time", str(time.perf_counter() - start).encode())
                )
            await send(message)
        await self.app(scope, receive, send_wrapper)
```

Pure ASGI middleware is preferred for hot paths — `BaseHTTPMiddleware` historically had buffering issues with streaming responses (largely fixed but still adds overhead). Know both forms; interviewers ask.

## Streaming and SSE

ASGI makes streaming natural — send multiple body chunks:

```python
from fastapi.responses import StreamingResponse

async def event_stream():
    while True:
        event = await queue.get()
        yield f"data: {event.json()}\n\n"   # Server-Sent Events format

@app.get("/events")
async def events():
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

Use cases: LLM token streaming, live dashboards, long-running job progress. This is a first-class ASGI feature WSGI never had.

## Async database access

The stack for PostgreSQL:

```python
import asyncpg  # or: sqlalchemy[asyncio] + asyncpg driver

# asyncpg: fast, raw SQL
pool = await asyncpg.create_pool(dsn, min_size=5, max_size=20)
async with pool.acquire() as conn:
    rows = await conn.fetch(
        "SELECT id, seats FROM trips WHERE origin = $1 AND depart_at >= $2",
        "BER", datetime.now(),
    )

# SQLAlchemy 2.0 async ORM:
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
engine = create_async_engine("postgresql+asyncpg://...")
Session = async_sessionmaker(engine, expire_on_commit=False)
async with Session() as session:
    async with session.begin():
        session.add(booking)
    # commit happens on block exit
```

Rules that come up in interviews and code review:

- **Never share a single connection across concurrent tasks** — use a pool, acquire per unit of work.
- **ORM lazy loading breaks under async** (implicit I/O can't `await`) — use eager loading (`selectinload`/`joinedload`) or explicit queries. This is the SQLAlchemy async gotcha.
- **Transactions**: acquire connection → `async with conn.transaction():` → commit/rollback automatic on exception.

## Background work in ASGI services

Options, in order of preference:

1. **`FastAPI BackgroundTasks`** — fire-and-forget within the same process, runs after response. Only for trivial, lossy-tolerant work. Dies with the worker — no durability.
2. **Task queues** (Celery, Dramatiq, ARQ, Taskiq) — separate worker processes, broker (Redis/RabbitMQ), retries, durability. Default choice for anything that must not be lost.
3. **Stream/event-driven** (Kafka + consumer service) — for high-volume, ordered, multi-consumer workloads.

```python
from fastapi import BackgroundTasks

@app.post("/send-welcome")
async def send_welcome(user_id: str, tasks: BackgroundTasks):
    tasks.add_task(send_email, user_id)  # runs after response is sent
    return {"status": "queued"}
```

Interview framing: BackgroundTasks is **not** a job system — no retry, no persistence, dies with the process. If the JD mentions outbox/idempotency, they want durable async processing, and the answer is broker + worker (or transactional outbox).

## Common async pitfalls (memorize these)

1. **Blocking the loop** — sync `requests`, `time.sleep`, CPU work, sync DB drivers inside coroutines. Fix: `asyncio.to_thread`, async libraries, executors.
2. **Un-awaited coroutines** — calling `coro()` without `await`/`create_task` does nothing (RuntimeWarning, lost work).
3. **Fire-and-forget tasks without holding a reference** — `asyncio.create_task(work())` result may be garbage-collected mid-flight; store the task or use a task set. Exceptions vanish silently if never awaited.

```python
task = asyncio.create_task(work())
background_tasks.add(task)
task.add_done_callback(background_tasks.discard)
```

4. **Shared mutable state across requests** — one loop, one thread, but `await` points are interleaving points. Two coroutines can interleave around any `await`, so check-then-act races are real:

```python
async def hold_seats(trip_id: str, n: int):
    if trips[trip_id].available >= n:      # check
        await asyncio.sleep(0)             # ← interleaving happens here
        trips[trip_id].available -= n      # act — another coroutine already decremented
```

Fix: `asyncio.Lock` around check-then-act, or make the operation atomic in the database (`UPDATE ... WHERE available >= n`), which is the correct production answer — single-threaded loop does not remove the need for transactions.

5. **Sync libraries in async endpoints** — the entire dependency tree must be async (httpx not requests, asyncpg not psycopg2, redis.asyncio not redis) — or wrapped in `to_thread`.
6. **`TaskGroup` exception groups** — child exceptions surface as `ExceptionGroup`; use `except*` or unwrap in the parent.
7. **Awaiting in `finally` or destructors** — may run at surprising points (loop shutdown); cleanup belongs in lifespan/context managers.

## Deployment shape

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4 --loop uvloop
```

- Each worker: one process, one loop, one core. `--workers` ≈ CPU core count.
- Behind a reverse proxy (nginx/ALB) or a container orchestrator (K8s, with a per-pod worker count).
- Health/readiness endpoints must **not** touch the DB loop-blocking; use async checks.
- Graceful shutdown: lifespan `yield` exit runs on SIGTERM — close pools, drain tasks; Uvicorn stops accepting connections first.

Mental model for capacity: async Python shines for **many concurrent slow-I/O requests** (thousands of waiting connections on one worker). It is **not** faster for CPU-bound work — for that, more processes, and since 3.14 optionally free-threaded builds.

---

*Next: [03-testing-and-tooling.md](03-testing-and-tooling.md) — pytest, mocking async code, profiling, and modern tooling.*
