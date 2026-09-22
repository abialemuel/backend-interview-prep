# Testing and Tooling

## pytest: the default test framework

Nobody uses stdlib `unittest` in modern teams. pytest is the assumed answer in interviews.

```python
# test_booking.py
import pytest

from app.booking import allocate_seats, TripFullError

def test_allocates_seats_on_requested_segment():
    trip = make_trip(available=10)
    booking = allocate_seats(trip, seats=4)
    assert booking.seats == 4
    assert trip.available == 6

def test_raises_when_trip_is_full():
    trip = make_trip(available=2)
    with pytest.raises(TripFullError, match="no seats available"):
        allocate_seats(trip, seats=4)
```

### Fixtures: setup, teardown, and dependency injection

Fixtures are pytest's DI. Declare what a test needs; pytest resolves and caches per test:

```python
import pytest
import asyncpg

@pytest.fixture
def event():
    return make_trip(available=10)          # fresh per test

@pytest.fixture(scope="session")
def database_url():
    return os.environ["TEST_DATABASE_URL"]

@pytest.fixture(scope="session")
async def pool(database_url):
    pool = await asyncpg.create_pool(database_url)
    yield pool                               # everything after yield = teardown
    await pool.close()

@pytest.fixture(autouse=True)
def _require_clean_db(db_snapshot):
    """Runs for every test automatically — guards against dirty database."""
    ...
```

Key semantics:

- **Scope**: `function` (default) → `class` → `module` → `package` → `session`. Session fixtures (DB pool, containers) are created once for the whole run.
- **Teardown**: code after `yield` runs as cleanup — even if the test fails.
- **Fixtures compose**: a fixture can depend on other fixtures. Resolution is by name in the test signature.
- **`conftest.py`**: shared fixtures live here, discovered automatically up the directory tree. No imports needed.
- **`autouse=True`**: applies to every test in scope — use sparingly (guards, env setup).

### Parametrize: one test, many cases

```python
@pytest.mark.parametrize(
    ("seats", "available", "expected_ok"),
    [
        (1, 10, True),      # normal
        (10, 10, True),     # exact fit
        (11, 10, False),    # overbook
        (0, 10, False),     # invalid
    ],
    ids=["normal", "exact-fit", "overbook", "zero"],
)
def test_allocation_rules(seats, available, expected_ok):
    trip = make_trip(available=available)
    if expected_ok:
        allocate_seats(trip, seats=seats)
    else:
        with pytest.raises(Exception):
            allocate_seats(trip, seats=seats)
```

`ids` makes failure output readable. This is the Python equivalent of Go's table-driven tests.

### Parametrizing fixtures (indirect)

```python
@pytest.fixture
def trip(request):
    return make_trip(available=request.param)

@pytest.mark.parametrize("trip", [0, 5, 100], indirect=True)
def test_something(trip): ...
```

### Mocking: monkeypatch, mocker, and fakes

Three layers, in order of preference for senior engineers:

1. **Fakes** (real in-memory implementation) — best: no mocking framework, tests real behavior.
2. **`monkeypatch`** (built-in) — for env vars, attributes, module functions.
3. **`unittest.mock` / `pytest-mock`** — for verifying call interactions.

```python
# 1. monkeypatch — stdlib, clean
def test_reads_timeout_from_env(monkeypatch):
    monkeypatch.setenv("BOOKING_TIMEOUT_MS", "500")
    assert read_timeout_ms() == 500

# 2. mock — when you must verify interactions
from unittest.mock import AsyncMock, MagicMock

def test_payment_called_once(mocker):
    gateway = mocker.AsyncMock(spec=PaymentGateway)  # spec= prevents typo'd method names
    service = BookingService(gateway)
    await service.book(order)
    gateway.charge.assert_awaited_once_with(order.total, idempotency_key=order.id)
```

**Async mocking** (critical for the ASGI stack): plain `MagicMock` doesn't handle `await`. Use `AsyncMock` — `assert_awaited_once_with`, not `assert_called_once_with`, for async methods.

**Over-mocking smell**: if a test asserts `gateway.charge.assert_called_once`, it tests implementation, not behavior. Senior answer: mock at boundaries only (network, DB, time), test real logic against fakes.

### Testing FastAPI apps

```python
from fastapi.testclient import TestClient   # sync; wraps httpx + ASGI transport
from httpx import ASGITransport, AsyncClient

# Sync style — fine for most tests
def test_create_booking():
    client = TestClient(app)
    resp = client.post("/bookings", json={"trip_id": "t1", "seats": 2})
    assert resp.status_code == 201

# Async style — needed when you await app state directly
async def test_create_booking_async():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.post("/bookings", json={...})
    assert resp.status_code == 201
```

No real server, no sockets — the ASGI transport calls the app coroutine directly. Fast, deterministic. Dependency overrides for isolation:

```python
app.dependency_overrides[get_booking_service] = lambda: FakeBookingService()
```

### Testing async code

```python
# pytest-asyncio
@pytest.mark.asyncio
async def test_pool_roundtrip(pool):
    async with pool.acquire() as conn:
        assert await conn.fetchval("SELECT 1") == 1
```

- `pytest-asyncio` in `auto` mode (`asyncio_mode = "auto"` in config) removes the need for the marker on every test.
- `anyio` (Starlette's backend) is an alternative: `@pytest.mark.anyio` — tests run on both asyncio and trio.
- **Test that blocking calls don't exist**: run the app with a loop-debug mode (`PYTHONASYNCIODEBUG=1`, or `loop.slow_callback_duration` tuning) in CI — logs any callback that blocks the loop beyond a threshold. Interview gold: "we detect loop-blocking in CI, not in prod."

```python
# Debug pattern — catches blocking calls in tests
import asyncio, warnings

@pytest.fixture(autouse=True)
def _asyncio_debug():
    loop = asyncio.new_event_loop()
    loop.set_debug(True)
    loop.slow_callback_duration = 0.05   # log anything blocking > 50ms
    yield
    loop.close()
```

### Property-based testing (hypothesis)

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()), st.integers(min_value=1))
def test_batching_preserves_order(items, size):
    assert [x for batch in batched(items, size) for x in batch] == items
```

Mention it for pure logic (allocation rules, pricing math, date range overlaps). Interviewers rank this signal highly at senior level.

## Test strategy at service level

| Layer | What | Speed | Count |
|-------|------|-------|-------|
| Unit | Pure logic, allocation rules, pricing | ms | many |
| Integration | Real DB (testcontainers), real broker, HTTP via ASGI transport | 100ms–1s | dozens |
| Contract | Provider/consumer pacts or OpenAPI schema checks | 1s | per endpoint |
| E2E | Few happy paths against a staging deployment | 10s | handful |

- **testcontainers** (Python lib) spins real Postgres/Kafka in Docker for integration tests — preferred over mocking the DB. Story to tell: "our integration suite boots a real Postgres per session; schema migrations run before tests."
- **Freeze time** with `freezegun` or `time-machine` for anything date-dependent (pricing by time-to-departure is exactly this).

```python
@pytest.mark.freeze_time("2026-09-21T10:00:00Z")
def test_price_increases_near_departure():
    dep = datetime(2026, 9, 21, 11, 0, tzinfo=UTC)   # 1h to departure
    assert price_for(dep) == high_price
```

## Profiling and performance

### cProfile + pstats (stdlib)

```python
python -m cProfile -o prof.out -m app.worker

import pstats
pstats.Stats("prof.out").sort_stats("cumulative").print_stats(20)
```

Read: `cumtime` (inclusive) vs `tottime` (self). Start with `tottime` to find hot leaf functions.

### py-spy: production-safe sampling profiler

```bash
py-spy top --pid 12345          # live CPU view, no code changes, no restart
py-spy dump --pid 12345         # stack snapshot — where is it stuck?
py-spy record -o flame.svg --pid 12345   # flame graph
```

This is the answer to "how do you debug a Python service that's pegged at 100% CPU in production" — sampling profiler, attach to the live process, zero restart. `py-spy dump` also catches GIL contention and stuck loops.

### Memory: tracemalloc + objgraph

```python
import tracemalloc
tracemalloc.start()
run_workload()
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:10]:
    print(stat)
```

Common memory bugs: unbounded caches (`functools.lru_cache` without `maxsize`), storing request bodies globally, growing task sets, retaining exceptions with tracebacks (reference cycles).

### Performance levers, in order

1. **Fix algorithm/data structures** — dict/set lookups vs list scans; generators vs materialized lists.
2. **Fix I/O pattern** — N+1 queries → batched queries; sequential awaits → `gather`/`TaskGroup`; missing connection pool.
3. **Cache** — `lru_cache` for pure functions; Redis for shared state; know invalidation strategy.
4. **Pydantic v2 / msgspec** — if validation dominates CPU profiles, msgspec (or `model_validate` with pre-parsed JSON) is faster than FastAPI's default path.
5. **uvloop** — drop-in event loop, ~2x loop throughput.
6. **Free-threaded 3.14 / multiprocessing** — only for genuinely CPU-bound parallel work; measure first.

Interview framing: never say "add threads" first. Profile → identify → fix cause → re-measure. Threads are the answer only for the specific case of blocking C libraries you can't replace.

## Tooling: the modern stack

### uv — package and environment manager

Replaced pip + virtualenv + pyenv + poetry-lock for most new projects (Astral, Rust, 10–100x faster):

```bash
uv init                      # new project (pyproject.toml)
uv add fastapi uvicorn       # add deps, updates lockfile
uv sync                      # exact install from uv.lock
uv run pytest                # run in project venv
uv python install 3.14       # manage Python versions
```

Mentioning `uv` + `uv.lock` in CI signals current practice. Legacy equivalents: `pip` + `requirements.txt`, `poetry`, `pipenv`.

### ruff — lint + format

One Rust tool replacing flake8, isort, pyupgrade, black (formatting via `ruff format`):

```toml
[tool.ruff]
target-version = "py314"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC", "S"]   # ASYNC rules catch blocking-call patterns!
```

The `ASYNC` ruleset (flake8-async) flags blocking calls in async functions (`requests.get` inside `async def`, etc.) — static detection of the #1 asyncio bug. Also `S` (bandit) for security linting.

### Type checking: mypy / pyright

```toml
[tool.mypy]
python_version = "3.14"
strict = true                # full strictness: no implicit Any, all functions typed
plugins = ["pydantic.mypy"]
```

- `mypy`: the default in most teams.
- `pyright`/`basedpyright`: faster, used by VS Code Pylance.
- Strict mode is the senior-level expectation: `disallow_untyped_defs`, `warn_return_any`, `strict_equality`.
- CI gate: `ruff check && mypy . && pytest` — state this pipeline shape when asked about CI.

```bash
# Typical CI steps for a Python service (GitHub Actions)
- run: uv sync --frozen            # reproducible install
- run: uv run ruff check . && uv run ruff format --check .
- run: uv run mypy .
- run: uv run pytest --cov=app --cov-fail-under=80
```

### Other tools worth naming

| Tool | Purpose |
|------|---------|
| `pytest-cov` + `coverage` | Coverage gates (aim for meaningful coverage, not the number) |
| `testcontainers` | Real dependencies in integration tests |
| `hypothesis` | Property-based testing |
| `time-machine` / `freezegun` | Time control in tests |
| `pre-commit` | Local hook runner (ruff, mypy fast subset) |
| `bandit` / `pip-audit` / `safety` | Dependency + code security scanning |
| `granian` | Alternative Rust-based ASGI server (RSGI) — worth knowing the name |

## Observability hooks

- **OpenTelemetry**: `opentelemetry-instrumentation-fastapi` auto-instruments ASGI apps — spans per request, contextvars-based trace propagation.
- **structlog**: structured logging with contextvars binding (`request_id` bound once per request, appears in every log line).
- **Prometheus**: `prometheus-fastapi-instrumentator` for RED metrics; async in-flight gauges need custom collection (loop-lag metric = scheduler health).

Loop-health metric worth volunteering: measure `asyncio` loop lag (schedule a callback every second, record drift) — detects event-loop starvation in prod before users do.

---

*Next: [04-interview-questions.md](04-interview-questions.md) — 40+ questions with model answers.*
