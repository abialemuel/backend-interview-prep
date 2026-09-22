# Core Concepts

## Data model: everything is an object

Every value in Python is an object — integers, functions, classes, modules, `None`. There are no primitive types. Even integers are heap-allocated objects with reference counting and garbage collection:

```python
x = 42
y = x
print(id(x) == id(y))  # True — same object in memory
```

This has consequences: integer values are never truly "just 42" on the stack. The runtime manages every value with a reference count (and a cyclic garbage collector for reference cycles).

**Integer caching** (CPython implementation detail): small integers (-5 to 256) are pre-allocated singletons:

```python
a = 256
b = 256
print(a is b)  # True — cached range

a = 257
b = 257
print(a is b)  # False — new object each (in most contexts)
```

This is why you compare values with `==`, not `is` (except for `None`, which should always be compared with `is`).

## Mutability and identity

Two core axes: **identity** (`is` / `id()`) vs **equality** (`==`), and **mutable** vs **immutable** types:

| Type | Mutable | Example |
|------|---------|---------|
| `int`, `float`, `complex` | No | `x = 1; x += 1` creates a new object |
| `str` | No | `s = "hi"; s[0]` works, `s[0] = "H"` raises `TypeError` |
| `bytes`, `tuple`, `frozenset` | No | immutable sequences |
| `list` | Yes | `l[0] = 1` works in place |
| `dict`, `set` | Yes | modified in place |
| Custom classes | Yes by default | unless `frozen=True` dataclass or `__slots__` |

**Default arguments are mutable gotcha** — the single most common Python interview trap:

```python
def append_to(item, target=[]):
    target.append(item)
    return target

print(append_to(1))  # [1]
print(append_to(2))  # [1, 2]  ← the default list is shared across calls
```

The fix: use `None` sentinel and create a fresh list inside the function:

```python
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

## Naming, scoping, LEGB

Python resolves names by the **LEGB** rule: Local → Enclosing → Global → Built-in. There is no block scope — `for`, `if`, `with` all execute in the enclosing scope:

```python
x = 10
for i in range(5):
    x = i
print(x)  # 4 — loop variable leaked into enclosing scope
```

This is different from Go (where block-scoping is strict) and is a common source of bugs. Since Python 3.12, list comprehensions no longer leak their iteration variable.

**Walrus operator** `:=` (since 3.8) lets you assign inside an expression — useful in `if` and `while`:

```python
data = fetch()
if (n := len(data)) > 1000:
    print(f"large: {n} items")
```

## Functions are first-class objects

Functions are objects with attributes. You can pass them, assign them, close over variables:

```python
def outer():
    msg = "hello"
    def inner():
        return msg  # closure over `msg`
    return inner

greet = outer()
print(greet())  # "hello"
print(greet.__closure__[0].cell_contents)  # "hello"
```

**Closures and mutable state** — another common interview pattern:

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count  # necessary to mutate the enclosing variable
        count += 1
        return count
    return increment

c = make_counter()
print(c())  # 1
print(c())  # 2
```

Without `nonlocal`, `count += 1` would raise `UnboundLocalError` because `count` would be treated as local (due to the assignment).

## Decorators

Decorators are higher-order functions that wrap another function or class:

```python
def retry(max_attempts=3):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
        return wrapper
    return decorator

@retry(max_attempts=5)
def flaky():
    ...
```

Key points for interviews:
- `functools.wraps` preserves the original function's `__name__`, `__doc__`, and `__module__`. Always use it.
- Decorators are evaluated at **import time**, not call time. This can be surprising.
- `@functools.lru_cache` and `@functools.cache` (3.9+) are built-in memoization decorators.
- Class-based decorators implement `__init__` (takes the decorated function) and `__call__` (the wrapper).

## Generators and iterators

Generators produce values lazily — they don't store the full result in memory:

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

fib = fibonacci()
first_10 = [next(fib) for _ in range(10)]  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

**Generator expressions** (like list comprehensions but lazy):

```python
total = sum(x ** 2 for x in range(1_000_000))  # no list allocated
```

**`yield from`** delegates to a sub-generator (3.3+):

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item
```

**Generators and context managers** — generators can be used as context managers via `contextlib.contextmanager`:

```python
from contextlib import contextmanager

@contextmanager
def timer(label: str):
    import time
    start = time.perf_counter()
    yield  # the `with` body runs here
    elapsed = time.perf_counter() - start
    print(f"{label}: {elapsed:.3f}s")
```

Interview trap: forgetting that `yield` in a `@contextmanager` splits the function into setup (before `yield`) and teardown (after `yield`), and that `yield` must produce exactly one value.

## Context managers (the protocol)

Context managers implement `__enter__` and `__exit__`:

```python
class DatabaseTransaction:
    def __init__(self, conn):
        self.conn = conn

    def __enter__(self):
        self.tx = self.conn.begin()
        return self.tx

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            self.tx.rollback()
        else:
            self.tx.commit()
        return False  # don't suppress exceptions
```

**`__exit__` return value**: if it returns `True`, the exception is suppressed. Almost always return `False` (or `None`). Suppressing exceptions silently is almost always a bug.

## OOP and dunder protocols

Python has no interfaces in the Go/Java sense. Behavior is defined by **dunder (double-underscore) methods** — the "protocols":

```python
class Money:
    def __init__(self, amount: float, currency: str):
        self.amount = amount
        self.currency = currency

    def __repr__(self):           # repr(money)
        return f"Money({self.amount}, '{self.currency}')"

    def __eq__(self, other):      # money == other
        return (self.amount == other.amount
                and self.currency == other.currency)

    def __lt__(self, other):      # money < other (needed for sorting)
        return self.amount < other.amount

    def __add__(self, other):     # money + other
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

    def __hash__(self):           # needed for dict/set membership
        return hash((self.amount, self.currency))
```

Key dunder methods for backend interviews:
- **Comparison**: `__eq__`, `__lt__`, `__le__`, `__gt__`, `__ge__`. Use `@functools.total_ordering` to reduce boilerplate.
- **String representation**: `__repr__` (unambiguous, for developers), `__str__` (human-readable).
- **Container protocols**: `__len__`, `__getitem__` (indexing/slicing), `__iter__`, `__contains__`.
- **Callable protocol**: `__call__` makes an instance callable like a function.
- **Attribute access**: `__getattr__` (called when attribute not found), `__getattribute__` (called on every access).

## Dataclasses and attrs

`@dataclass` (since 3.7, improved in 3.10+) reduces boilerplate for data-holding classes:

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)  # immutable — creates __hash__, safe for dict keys
class SeatAllocation:
    trip_id: str
    stop_from: str
    stop_to: str
    seats: int = 1

    def __post_init__(self):
        if self.seats <= 0:
            raise ValueError("seats must be positive")
```

**`__post_init__`** runs after `__init__` — use for validation or derived fields.

**`frozen=True`** makes the instance immutable (raises `FrozenInstanceError` on attribute set). Gives you `__hash__` for free, safe for dict keys.

**`slots=True`** (since 3.10) generates `__slots__` — reduces memory usage, faster attribute access. Combine with frozen for best performance: `@dataclass(frozen=True, slots=True)`.

**Performance note**: `frozen=True` + `slots=True` dataclass instances can be 10–20x faster to create and use less memory than plain class instances, because CPython can optimize attribute access with `__slots__`.

## Enums

```python
from enum import Enum, auto

class BookingStatus(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    CANCELLED = auto()

status = BookingStatus.CONFIRMED
print(status.value)  # 2
print(status.name)   # "CONFIRMED"
```

Use `StrEnum` (3.11+) for JSON-serializable string enums — no custom encoder needed.

## Exceptions

Exceptions in Python are values — you raise them with `raise` and catch them with `try/except`. There are no checked exceptions.

```python
try:
    result = process()
except ValueError as e:
    log.error("bad input: %s", e)
except (ConnectionError, TimeoutError) as e:
    log.error("network failure: %s", e)
    raise  # re-raise after logging
except Exception as e:
    log.error("unexpected: %s", e)
    raise  # always re-raise unknown exceptions
else:
    print("success, no exception")  # runs only if no exception
finally:
    cleanup()  # always runs
```

**Exception groups** (since 3.11) — handle multiple concurrent exceptions (e.g., from `TaskGroup`):

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch_user())
        tg.create_task(fetch_orders())
except* ValueError as eg:
    # eg is an ExceptionGroup — handles all ValueError sub-exceptions
    for exc in eg.exceptions:
        print(f"bad value: {exc}")
except* ConnectionError as eg:
    ...
```

**`except*`** (3.11+) catches exceptions from an `ExceptionGroup` without flattening the group. This is critical for async code using `TaskGroup` (see 02-asyncio-asgi-and-web.md).

**`raise` vs `raise e`**: always use bare `raise` to re-raise — it preserves the original traceback. Using `raise SomeError()` creates a new exception with a misleading traceback.

## Type hints and protocols

Python's type system is optional but increasingly expected. Since 3.12, the syntax is cleaner:

```python
# Before 3.12:
from typing import TypeVar
T = TypeVar("T")
def first(items: list[T]) -> T: ...

# Since 3.12 (PEP 695):
def first[T](items: list[T]) -> T:
    return items[0]
```

**Protocols** (structural subtyping, since 3.8) — Python's answer to interfaces:

```python
from typing import Protocol

class Repository(Protocol):
    def get(self, id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

def process_order(repo: Repository, order_id: str) -> dict:
    order = repo.get(order_id)
    # ...
    repo.save(order)
    return order
```

Any class with `get` and `save` with compatible signatures satisfies `Repository` — no explicit inheritance needed. This is duck typing formalized.

**Key typing constructs for backend work**:

| Type | Use |
|------|-----|
| `list[str]`, `dict[str, int]` | Generic containers |
| `Optional[str]` / `str \| None` (3.10+) | Nullable |
| `Callable[..., Any]` | Functions |
| `AsyncGenerator[YieldType, SendType]` | Async generators |
| `TypedDict` | Dict with known keys/types |
| `Literal["a", "b"]` | Exact values |
| `TypeIs` (3.13+) | Narrowing in type guards |
| `Unpack` / `**kwargs: Unpack[TypedDict]` | Typed kwargs |
| `ParamSpec` (3.10+) | Decorator type preservation |
| `dataclass` / `NamedTuple` | Structured data |

**`TypedDict`** — dict with compile-time key/type checking:

```python
from typing import TypedDict

class BookingRequest(TypedDict):
    trip_id: str
    seats: int
    passenger_name: str

def book(req: BookingRequest) -> None:
    if req["seats"] > 0:
        ...
```

## The Global Interpreter Lock (GIL) and free-threading

**Classic GIL story** (pre-3.13): CPython's GIL is a mutex that allows only one thread to execute Python bytecode at a time. This means CPU-bound Python threads don't parallelize — you need `multiprocessing` for true parallelism. I/O-bound code is fine with threads (the GIL is released during I/O waits).

**Free-threaded Python** (3.13+ experimental, 3.14 officially supported): PEP 703 made the GIL optional. Build CPython with `--disable-gil` or use the free-threaded binary (3.14+ ships as a separate binary alongside the default build). With free-threading enabled:

- Multiple threads can execute Python bytecode in parallel
- Thread safety becomes the programmer's responsibility — you need `threading.Lock` or `threading.RLock` for shared mutable state
- C extensions need to be updated to opt into free-threading compatibility (many major libraries have done this as of 2025)
- Single-threaded performance has a small regression (~5–10%) due to the overhead of disabling the GIL

**Interview answer for "the GIL":** The GIL serializes CPU-bound Python threads. It was practical when threads were the main concurrency primitive, but modern code uses async I/O for concurrency (one thread, many tasks) and multiprocessing for parallelism. Free-threaded Python (3.14+) makes threads truly parallel, but it's a opt-in mode and most async/await code doesn't need it — async is still the right tool for I/O-bound services.

## Imports and modules

Python imports are **eager** — all top-level code in a module runs at import time. This is different from Go (where packages are compiled together) and catches people off guard.

```python
# database.py
import psycopg2  # this runs when anyone does `import database`
conn = psycopg2.connect("postgres://...")  # this also runs at import time!
```

This is why connection setup goes inside functions or class methods, not at module level.

**`if __name__ == "__main__":`** guards module-level executable code:

```python
def main():
    ...

if __name__ == "__main__":
    main()
```

**Relative vs absolute imports**: prefer absolute imports for clarity:

```python
# preferred
from app.services.booking import BookService

# less clear (relative)
from ..services.booking import BookService
```

## Generics (3.12+ syntax)

Since Python 3.12 (PEP 695), generic syntax is cleaner — type parameters inline:

```python
# Before 3.12:
from typing import TypeVar, Generic
T = TypeVar("T")
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

# 3.12+:
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

# 3.12+ functions:
def first[T](items: list[T]) -> T:
    return items[0]
```

TypeVar constraints (constrained type parameters):

```python
# Before 3.12:
Numeric = TypeVar("Numeric", int, float, complex)
def add(a: Numeric, b: Numeric) -> Numeric: ...

# 3.12+:
def add[Numeric: (int, float, complex)](a: Numeric, b: Numeric) -> Numeric:
    return a + b
```

## Packaging and environments

**`uv`** (since 2024, Astral) has replaced pip + venv + poetry for many teams:

```bash
# Create venv and install dependencies
uv venv
uv pip install -r requirements.txt

# Or use as project manager (replaces poetry)
uv init my-project
uv add fastapi uvicorn
```

`uv` is a single Rust binary that is 10–100x faster than pip. It handles venvs, dependency resolution, locking, and Python version management. As of 2025, it is the dominant tool in new Python projects.

**`pyproject.toml`** is the standard project config (PEP 621):

```toml
[project]
name = "booking-service"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "pydantic>=2.9",
    "sqlalchemy[asyncio]>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-asyncio>=0.24", "ruff>=0.8", "mypy>=1.13"]

[tool.ruff]
target-version = "py312"
line-length = 88

[tool.mypy]
python_version = "3.12"
strict = true
```

## Common pitfalls for Go → Python developers

| Concept | Go | Python |
|---------|----|----|
| Mutability | Slices/maps are reference types but can be re-assigned; values in structs are copies | Everything is a reference to an object; lists/dicts are shared by default |
| Concurrency | Goroutines + channels are the default | `asyncio` for I/O, `multiprocessing` for CPU parallelism, `threading` (limited by GIL) |
| Error handling | `if err != nil` explicit, everywhere | `try/except` blocks, exceptions flow through call stack |
| Type system | Static, compile-time | Optional type hints, checked by mypy/pyright, not enforced at runtime |
| Scope | Block-scoped (strict) | Function-scoped (LEGB), `for`/`if` don't create new scope |
| Null | Zero values, `nil` pointers | `None` — compare with `is`, not `==` |
| Tooling | `go fmt`, `go test`, `go vet` built-in | Separate tools: `ruff` (lint+format), `pytest`, `mypy`, `uv` (package manager) |
| Default behavior | No default arguments | Mutable default arguments are shared across calls |
| Immutability | `const` is the only guarantee | `frozen=True` dataclass, `tuple`, `frozenset` |

## Performance characteristics

- **List vs generator**: list comprehensions allocate the full list in memory; generator expressions yield lazily. Use generators for large/streaming data.
- **Dict vs list for lookups**: dict lookup is O(1) amortized; list `.index()` is O(n). Use sets/dicts for membership tests.
- **String concatenation**: use `"".join(parts)`, not `s += part` in a loop. String concatenation is O(n²) in the worst case because strings are immutable and may be reallocated.
- **`__slots__`**: eliminates per-instance `__dict__`, saving memory and speeding attribute access. Combined with frozen dataclass: significant for large numbers of instances.
- **Generator vs list comprehension**: list comprehension is faster when you need all items (no yield overhead). Generator is faster for memory when you have millions of items and don't need them all at once.

---

*Next: [02-asyncio-asgi-and-web.md](02-asyncio-asgi-and-web.md) — the async runtime, ASGI, and the web stack.*
