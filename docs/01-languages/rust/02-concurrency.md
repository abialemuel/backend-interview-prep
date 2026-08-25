# Concurrency

Rust's marketing slogan for this topic is "fearless concurrency": the same ownership and borrowing rules that prevent memory bugs at compile time also prevent data races at compile time. This file covers threads, channels, `Send`/`Sync`, async/await, and the trade-offs of Rust's approach versus Go's.

## `Send` and `Sync` — why data races are a compile error

Two marker traits, both auto-implemented by the compiler for most types, govern what can cross thread boundaries:

- **`Send`**: a type is `Send` if it's safe to *transfer ownership* of a value of that type to another thread.
- **`Sync`**: a type is `Sync` if it's safe for multiple threads to hold a *shared reference* (`&T`) to it simultaneously — equivalently, `T` is `Sync` if and only if `&T` is `Send`.

Almost every type is both. The interesting cases are the ones that aren't:

| Type | `Send`? | `Sync`? | Why |
|------|---------|---------|-----|
| `i32`, `String`, `Vec<T>` (where `T: Send`) | Yes | Yes | Plain owned data, no shared mutable state |
| `Rc<T>` | No | No | Its reference count is a plain (non-atomic) integer — two threads incrementing it concurrently is a data race |
| `Arc<T>` (where `T: Send + Sync`) | Yes | Yes | Reference count is atomic; safe to share |
| `RefCell<T>` | Yes (if `T: Send`) | No | Its borrow-tracking flag is not atomic — safe to move to another thread, unsafe to share a reference across threads |
| `Mutex<T>` (where `T: Send`) | Yes | Yes | Internally synchronized; safe to share |
| Raw pointers (`*const T`, `*mut T`) | No | No | No safety guarantees at all — the compiler can't verify anything about them |

This is what "compile error, not runtime bug" means concretely: if you try to move an `Rc<T>` into a spawned thread, or share a plain `RefCell<T>` reference across threads, the code **does not compile**. There is no code path where this reaches production and races under load — the class of bug is eliminated before the binary exists.

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(5);
    // thread::spawn(move || {
    //     println!("{data}");
    // });
    // ^ compile error: `Rc<i32>` cannot be sent between threads safely,
    //   because `Rc<i32>` doesn't implement `Send`.
    // The fix: use `Arc::new(5)` instead.
    println!("{data}");
}
```

Nearly all of `std`'s and the ecosystem's types implement `Send`/`Sync` automatically based on their fields (a struct is `Send`/`Sync` if all its fields are); you rarely write `impl Send for MyType` by hand, and doing so is an `unsafe impl` — you're personally vouching for a guarantee the compiler can't verify.

## Threads

`std::thread::spawn` starts an OS thread running a closure and returns a `JoinHandle`:

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("spawned: {i}");
        }
    });

    for i in 1..3 {
        println!("main: {i}");
    }

    handle.join().unwrap(); // blocks until the spawned thread finishes
}
```

There is no scheduler multiplexing many logical threads onto a few OS threads the way Go's runtime multiplexes goroutines onto a `GOMAXPROCS`-sized pool — `std::thread::spawn` maps 1:1 to an OS thread. That makes threads in Rust relatively heavyweight (megabyte-scale stacks, real context switches), which is exactly why the async ecosystem exists for I/O-bound concurrency at scale — closer to Go's model, but opt-in rather than built into the language runtime.

Closures passed to `spawn` almost always need `move`, because the closure's lifetime isn't tied to the caller's stack frame — the compiler can't prove a borrowed reference will outlive the spawned thread, so it requires the closure to take ownership of everything it uses:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let handle = thread::spawn(move || {
        println!("{data:?}"); // data is owned by this closure now
    });
    handle.join().unwrap();
}
```

## Channels (`mpsc`)

`std::sync::mpsc` (multi-producer, single-consumer) is the channel-based, "share memory by communicating" primitive:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        for i in 0..5 {
            tx.send(i).unwrap();
        }
        // tx is dropped here when the closure ends, closing the channel
    });

    for received in rx {
        println!("got: {received}");
    }
}
```

- `rx` implements `Iterator`; the `for` loop ends automatically once every `Sender` has been dropped and the channel is drained — there's no explicit `close()` call the way there is on a Go channel.
- `tx.clone()` gives you multiple producers feeding one consumer, matching the `mpsc` name.
- `mpsc::sync_channel(n)` gives a bounded channel — `send` blocks once the buffer of size `n` is full, the same backpressure behavior as a buffered Go channel.
- There is no equivalent of Go's `select` in `std`; waiting on multiple channels at once requires either a crate (`crossbeam-channel`'s `select!`) or moving to `async` and `tokio::select!`.

## `Arc<Mutex<T>>` — the standard shared-state pattern

For state genuinely shared and mutated across threads, the idiom is an `Arc` (shared ownership, atomic refcount) wrapping a `Mutex` (exclusive access):

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter); // clone the Arc, not the data
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        }); // MutexGuard drops here, releasing the lock
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("result: {}", *counter.lock().unwrap());
}
```

`counter.lock()` returns a `Result<MutexGuard<T>, PoisonError<...>>` — the `Result` exists because a `Mutex` becomes "poisoned" if a thread panics while holding the lock, and by default a later `.lock()` returns `Err` rather than silently handing you data that might be in a half-updated state. `.unwrap()` is the common shortcut in examples and in code where a poisoned mutex should crash the process rather than run on with suspect state; production code sometimes prefers `.lock().unwrap_or_else(|e| e.into_inner())` to recover and keep going.

`RwLock<T>` is the reader/writer equivalent of `RwMutex` in Go — many concurrent readers via `.read()`, one exclusive writer via `.write()`.

## Async/await

`async fn` doesn't run anything by itself — it compiles the function body into a state machine implementing the `Future` trait. Nothing happens until something **polls** that future, which is the job of a runtime (executor).

```rust
async fn fetch_status(id: u64) -> Result<u16, reqwest::Error> {
    let resp = reqwest::get(format!("https://api.example.com/items/{id}")).await?;
    Ok(resp.status().as_u16())
}
```

`.await` suspends the current async function at that point until the inner future resolves, yielding control back to the executor so it can run other tasks in the meantime — conceptually similar to a goroutine blocking on I/O and the Go scheduler running something else on that P, except the yield points in Rust are explicit (`.await`) rather than implicit.

### Why Rust needs an executor and Go doesn't

Go ships a scheduler in every binary: `go f()` is understood by the runtime with no library choice involved. Rust's standard library ships **no async runtime at all** — `async`/`await` is a language feature (the `Future` trait and the state-machine transformation), but *something* has to actually call `poll()` repeatedly to drive a future to completion, and the standard library deliberately doesn't provide that "something." This is a considered design trade-off, not an oversight: different domains want very different executors (multi-threaded work-stealing for a web server, single-threaded for an embedded target, `io_uring`-based for maximum Linux I/O throughput, WASM-compatible for browser/edge targets), and baking one in would force a one-size-fits-all cost onto every Rust binary, including ones that never touch async at all.

**Tokio** is the de facto standard for backend services — a multi-threaded, work-stealing async runtime with first-class `TcpStream`, timers, and a large ecosystem (`axum`, `tonic`, `sqlx`) built on top of it.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("done sleeping");
    });

    println!("doing other work while the task sleeps");
    handle.await.unwrap();
}
```

`#[tokio::main]` is a macro that expands to "create a runtime, then block the current thread running this async function on it" — under the hood it's ordinary Rust, no special language support required. This snippet needs `tokio = { version = "1", features = ["full"] }` in `Cargo.toml`.

### `select!` and timeouts

`tokio::select!` races multiple futures and proceeds with whichever completes first, cancelling the others — the async analog of Go's `select` over channels:

```rust
use tokio::time::{sleep, Duration};

async fn race_two_timers() {
    tokio::select! {
        _ = sleep(Duration::from_millis(50)) => {
            println!("50ms timer fired first");
        }
        _ = sleep(Duration::from_millis(100)) => {
            println!("this branch loses the race");
        }
    }
}
```

`tokio::time::timeout` wraps a single future with a deadline, returning `Err` if it doesn't resolve in time:

```rust
use tokio::time::{timeout, Duration};

async fn fetch_with_deadline() -> Result<u16, &'static str> {
    match timeout(Duration::from_secs(2), fetch_status(1)).await {
        Ok(Ok(status)) => Ok(status),
        Ok(Err(_)) => Err("request failed"),
        Err(_) => Err("timed out"),
    }
}
```

### Structured concurrency with `JoinSet`

Manually pushing `JoinHandle`s into a `Vec` and joining them one by one works, but `tokio::task::JoinSet` tracks a dynamic set of spawned tasks and lets you drive them to completion as each one finishes, in whatever order they complete:

```rust
use tokio::task::JoinSet;

async fn fetch_all(ids: Vec<u64>) -> Vec<u16> {
    let mut set = JoinSet::new();
    for id in ids {
        set.spawn(async move { fetch_status(id).await.unwrap_or(0) });
    }

    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        results.push(res.unwrap());
    }
    results
}
```

Dropping a `JoinSet` aborts every task still running in it — a form of structured concurrency where a task's lifetime is tied to its owning scope, closer to how a Go `errgroup` ties goroutine lifetimes to the group's `Wait()`.

### Blocking in async contexts is a real pitfall

An `async` task cooperatively yields at `.await` points. If a task runs a genuinely blocking call — `std::thread::sleep`, synchronous file I/O, a CPU-heavy loop with no `.await` — it **starves the executor thread**: no other task scheduled on that worker thread runs until the blocking call returns. Unlike a blocked goroutine (which the Go runtime detects and hands the P to another M), a blocked async task just sits on the executor's worker thread doing nothing productive, and the executor has no way to preempt it.

```rust
// BAD — blocks the entire worker thread the executor is running on
async fn bad_handler() {
    std::thread::sleep(std::time::Duration::from_secs(1));
}

// GOOD — yields to the executor, letting other tasks run during the wait
async fn good_handler() {
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
}
```

For work that's unavoidably blocking or CPU-bound (a synchronous database driver, heavy computation, a legacy C library call), hand it to a dedicated blocking thread pool instead of running it inline:

```rust
async fn compute_report() -> String {
    tokio::task::spawn_blocking(|| {
        // CPU-heavy or blocking work runs on a separate thread pool,
        // not on the async executor's worker threads
        expensive_computation()
    })
    .await
    .unwrap()
}

fn expensive_computation() -> String {
    "report".to_string()
}
```

## Concurrency pitfalls specific to Rust

- **Deadlocks are still possible.** Ownership rules prevent data races, not deadlocks — a thread locking `Mutex A` then waiting on `Mutex B`, while another thread locks `B` then waits on `A`, deadlocks exactly as it would in any other language. Always acquire locks in a consistent global order.
- **`Mutex` poisoning.** A panic while holding a lock poisons it; every later `.lock()` returns `Err` by default. Decide up front whether a poisoned mutex should crash the process (usually right for a genuine invariant violation) or be recovered from (`into_inner()`).
- **Blocking the async executor** (above) is the single most common async-Rust production bug — it manifests as latency spikes and timeouts under load that are hard to reproduce locally with low concurrency.
- **`Rc`/`RefCell` accidentally leaking into async code.** These types aren't `Send`, so if an async task needs to hold one across an `.await` point, the *future itself* becomes non-`Send`, which fails to compile the moment you try to `tokio::spawn` it (spawned tasks must be `Send` because the executor may move them between worker threads). The fix is almost always `Arc`/`Mutex` instead of `Rc`/`RefCell` in any type that might cross an `.await`.
- **Holding a `MutexGuard` across an `.await`.** Doing so keeps the lock held for the entire duration of whatever you're awaiting — if that's a network call, you've turned a fast in-memory lock into a lock held for however long the network takes, and any other task needing that lock queues up behind it. `clippy::await_holding_lock` catches this. Scope the guard tightly and drop it before the `.await`.
- **Unbounded channels as an implicit memory leak.** An `mpsc::channel()` (unbounded) with a producer that outpaces its consumer grows without limit — prefer `sync_channel(n)` (or a bounded async channel) when producer and consumer rates aren't guaranteed to match, exactly the backpressure argument for buffered channels in Go.

## Go vs Rust concurrency primitives at a glance

| Go | Rust equivalent | Key difference |
|----|------------------|----------------|
| goroutine | `std::thread::spawn` (OS thread) or `tokio::spawn` (async task) | Go's runtime schedules green threads by default; Rust has no built-in scheduler — a "thread" is a real OS thread unless you opt into an async runtime |
| channel | `std::sync::mpsc` (blocking) / `tokio::sync::mpsc` (async) | Rust has separate sync and async channel types; there's no single channel type that works in both worlds |
| `select` | `tokio::select!` (async only) | No `select` over blocking `std` channels without a third-party crate (`crossbeam-channel`) |
| `sync.Mutex` | `std::sync::Mutex<T>` / `tokio::sync::Mutex<T>` | Rust's mutex **owns** the data it protects (`Mutex<T>`), so you cannot touch the data without holding the guard — in Go the mutex and the data are separate fields, and nothing stops you from reading the field without locking |
| `sync.WaitGroup` | Collect `JoinHandle`s, or `tokio::task::JoinSet` | No single-call equivalent; you track handles yourself or use structured-concurrency helpers |
| `context.Context` cancellation | Dropping a future/task cancels it; `tokio_util::sync::CancellationToken` for explicit signaling | Cancellation isn't a first-class idiom baked into every function signature the way `ctx context.Context` is in Go |
| `GOMAXPROCS` | `Runtime::new().worker_threads(n)` / `#[tokio::main(worker_threads = n)]` | Both default to the number of visible CPUs |

## Pitfalls to remember

- `thread::spawn` closures almost always need `move`; forgetting it produces a lifetime error, not a silent bug, because the compiler refuses to compile a closure that might outlive borrowed data.
- `Arc::clone(&x)` clones the reference-counted pointer (cheap, atomic increment), not the underlying data — a common early confusion coming from languages where `.clone()` always means deep copy.
- `tokio::spawn` requires the future to be `'static` and `Send` — no borrowed data with a lifetime shorter than the task, and no non-`Send` types held across an await point.
- Mixing `std::thread` and `tokio` runtimes is fine and common (`spawn_blocking` exists exactly for this), but calling `.await` from plain synchronous code without an active runtime panics at runtime with "there is no reactor running" — a `#[tokio::main]` or an explicit `Runtime::block_on` must be on the call stack.
