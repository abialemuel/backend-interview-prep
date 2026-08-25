# Traits, Error Handling, and Idiomatic Patterns

## Static dispatch (generics) vs dynamic dispatch (`dyn Trait`)

Rust gives you two distinct ways to write code that works over "anything implementing trait `T`," and the choice has real, measurable runtime consequences — this is one of the most commonly probed design questions in a Rust interview.

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle { radius: f64 }
struct Square { side: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
}

impl Shape for Square {
    fn area(&self) -> f64 { self.side * self.side }
}
```

**Generics (static dispatch):**

```rust
fn print_area<T: Shape>(shape: &T) {
    println!("area: {}", shape.area());
}

fn main() {
    print_area(&Circle { radius: 1.0 });
    print_area(&Square { side: 2.0 });
}
```

The compiler generates a separate, specialized copy of `print_area` for each concrete type it's called with — `print_area::<Circle>` and `print_area::<Square>` — a process called **monomorphization**. Each copy calls `Shape::area` as a direct, inlinable function call: zero indirection, zero runtime dispatch cost. The trade-off is compile time and binary size: more call sites with more distinct type parameters means more generated code ("code bloat"), and generic-heavy libraries are a common source of slow release builds.

**Trait objects (dynamic dispatch):**

```rust
fn print_area_dyn(shape: &dyn Shape) {
    println!("area: {}", shape.area());
}

fn shapes() -> Vec<Box<dyn Shape>> {
    vec![Box::new(Circle { radius: 1.0 }), Box::new(Square { side: 2.0 })]
}

fn main() {
    for shape in shapes() {
        print_area_dyn(shape.as_ref());
    }
}
```

`&dyn Shape` is a **fat pointer**: a data pointer plus a pointer to a vtable (a static table of function pointers, one per trait method). Calling `shape.area()` is one indirect call through that vtable — a small, constant runtime cost, but a real one, and one the compiler generally can't inline through. In exchange, you get exactly what generics can't give you: a single, shared implementation (no per-type code generation) and **heterogeneous collections** — `Vec<Box<dyn Shape>>` can hold circles and squares in the same `Vec`, something `Vec<T>` for a generic `T` cannot do.

| | Generics (`impl Trait` / `<T: Trait>`) | `dyn Trait` |
|--|------------------------------------------|-------------|
| Dispatch | Static — resolved and inlined at compile time | Dynamic — vtable lookup at runtime |
| Cost per call | Zero (as fast as a non-generic function) | One indirect call |
| Binary size | Grows with each instantiation | One shared implementation |
| Compile time | Slower as instantiations multiply | Faster |
| Heterogeneous collections | No (need an enum, or `Box<dyn Trait>`) | Yes |
| Requires "object safety" | No | Yes (no generic methods, no `Self`-by-value returns) |

**Rule of thumb:** default to generics for library-internal, performance-sensitive code where the concrete types are known at compile time. Reach for `dyn Trait` when you genuinely need a heterogeneous collection, a plugin-style extension point, or when you want to avoid the compile-time and binary-size cost of monomorphizing over many types (a common choice in large applications with many trait implementors, e.g., HTTP middleware stacks).

## Error handling idioms

### The `?` operator

`?` on a `Result` either unwraps the `Ok` value or returns the `Err` early, converting the error type via `From` if the surrounding function's error type differs:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username() -> Result<String, io::Error> {
    let mut file = File::open("user.txt")?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

This is Rust's answer to Go's `if err != nil { return err }` — the same control flow, without repeating the check at every call site. `?` also works on `Option<T>` inside functions returning `Option`.

### Custom error types with `thiserror`

Library code should expose a **typed, matchable** error so callers can branch on what went wrong. `thiserror` derives `std::error::Error` and `Display` from annotations, and `#[from]` generates the `From` impl that makes `?` work across error types:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum ServiceError {
    #[error("item not found: {0}")]
    NotFound(String),
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),
}

fn load_item(id: &str) -> Result<String, ServiceError> {
    if id.is_empty() {
        return Err(ServiceError::NotFound(id.to_string()));
    }
    let contents = std::fs::read_to_string(id)?; // io::Error auto-converts via #[from]
    Ok(contents)
}
```

### Application-level errors with `anyhow`

At the top of an application (a binary, an HTTP handler), you usually don't care about the exact error type — you want to propagate it, attach context, and log or return it. `anyhow::Result<T>` (an alias for `Result<T, anyhow::Error>`) erases the concrete error type behind a single dynamic container while preserving the chain for debugging:

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let contents = std::fs::read_to_string("config.toml")
        .context("failed to read config.toml")?;
    println!("{contents}");
    Ok(())
}
```

**Rule of thumb, and a common interview question:** `thiserror` in libraries (callers need to match on specific variants), `anyhow` in applications (nothing downstream needs to pattern-match your error type — it just needs to be logged or displayed). Mixing them is normal: a library crate exposes `thiserror` enums; the binary that calls it wraps everything in `anyhow::Result` at the top.

### When to panic vs return `Result`

- **Return `Result`** for anything an operational failure can plausibly cause: malformed input, a network call that failed, a file that doesn't exist, a database constraint violation. The caller might reasonably retry, log, or degrade gracefully.
- **Panic** (`panic!`, `.unwrap()`, `.expect()`, an out-of-bounds index, an integer overflow in debug mode) for conditions that indicate **a bug in your own code**, not a runtime failure external callers should have to handle — an invariant you believe you maintained being violated, or a value you just constructed and know cannot be `None`.
- **`.expect("message")` over bare `.unwrap()`** in production code — the message becomes the panic output and should state *why* the value was assumed to be present, which is what future-you needs when the panic actually fires.
- Library code should almost never panic on bad *input* — that's the caller's data, and the caller should get a `Result` to decide what to do about it. Panicking on your own broken invariants (a `debug_assert!` that would fire) is fine because that's a bug report, not an operational condition.

## Iterators and zero-cost abstractions

Iterator chains compile down to the same machine code as an equivalent hand-written loop — no heap allocation for intermediate steps, no virtual dispatch, because each adapter (`.map`, `.filter`, `.sum`) is generic and gets monomorphized and inlined. This is what "zero-cost abstraction" means concretely: the abstraction (a chain of named operations) costs nothing at runtime versus the imperative version.

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6];
    let sum: i32 = nums.iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * n)
        .sum();
    println!("{sum}"); // 4 + 16 + 36 = 56
}
```

- `.iter()` yields `&T` (borrows); `.into_iter()` yields owned `T` (consumes the collection); `.iter_mut()` yields `&mut T`.
- Iterators are **lazy** — nothing runs until a consuming call (`.sum()`, `.collect()`, `.for_each()`, a `for` loop) drives the chain.

Implementing `Iterator` for a custom type only requires defining `next()`; every adapter (`.map`, `.zip`, `.take`, `.sum`, ...) comes free via the trait's default methods:

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let sum: u32 = Counter::new()
        .zip(Counter::new().skip(1))
        .map(|(a, b)| a * b)
        .filter(|x| x % 3 == 0)
        .sum();
    println!("{sum}"); // 18
}
```

## The builder pattern

Rust has no constructor overloading and no named/default arguments for arbitrary function calls, so builders are the idiomatic way to construct a value with many optional fields:

```rust
#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    max_connections: u32,
}

struct ServerConfigBuilder {
    host: String,
    port: u16,
    max_connections: u32,
}

impl ServerConfigBuilder {
    fn new() -> Self {
        ServerConfigBuilder {
            host: "127.0.0.1".to_string(),
            port: 8080,
            max_connections: 100,
        }
    }

    fn host(mut self, host: impl Into<String>) -> Self {
        self.host = host.into();
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn build(self) -> ServerConfig {
        ServerConfig {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
        }
    }
}

fn main() {
    let cfg = ServerConfigBuilder::new()
        .host("0.0.0.0")
        .port(9090)
        .build();
    println!("{cfg:?}");
}
```

Each builder method takes `self` by value and returns `Self`, so calls chain fluently; `build()` consumes the builder and produces the final, immutable value. Other idioms worth knowing by name: the **newtype pattern** (`struct UserId(u64)`, covered in `01-core-concepts.md`) for zero-cost type-safety around a primitive, and the **typestate pattern** (encoding a state machine's valid transitions in distinct types, so calling a method on the wrong state is a compile error rather than a runtime check).

## `unsafe` Rust

`unsafe` unlocks exactly five abilities the compiler otherwise forbids:

1. Dereferencing a raw pointer (`*const T` / `*mut T`).
2. Calling an `unsafe fn` or `unsafe` method.
3. Accessing or mutating a mutable `static` variable.
4. Implementing an `unsafe trait`.
5. Accessing a field of a `union`.

Crucially, `unsafe` does **not** turn off the borrow checker or type checking for the surrounding safe code — it only lets you perform those five specific operations, each of which comes with invariants the compiler cannot verify and that you are now personally responsible for upholding (valid pointers, no aliasing violations, no data races, properly initialized memory).

```rust
fn main() {
    let mut num = 5;
    let r1 = &raw const num; // creating the raw pointer is itself safe
    let r2 = &raw mut num;

    unsafe {
        // dereferencing it is the unsafe operation
        println!("r1 is: {}", *r1);
        *r2 += 1;
        println!("r2 is: {}", *r2);
    }
}
```

The most common legitimate use in backend code is **FFI** — calling into a C library:

```rust
unsafe extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("abs(-3) = {}", abs(-3));
    }
}
```

(Since the 2024 edition, `extern` blocks must themselves be marked `unsafe`, making explicit that *declaring* a foreign function signature is also something the compiler can't verify.)

**When `unsafe` is justified:** FFI boundaries, implementing a safe abstraction whose internals genuinely need manual memory control (this is how `Vec<T>`, `Box<T>`, and most of the standard library's collection types are actually built — safe on the outside, `unsafe` inside), and hot-path performance work where profiling has proven that a safe abstraction's bounds checks or synchronization are the bottleneck (`get_unchecked`, atomics with relaxed ordering).

**What you're now responsible for:** every invariant the safe language normally enforces for you — no data races, no dangling or misaligned pointers, no aliased mutable references, no reading uninitialized memory. Getting any of this wrong is **undefined behavior**, exactly as in C, and the failure mode is the same too: it might work fine in testing and corrupt memory in production under a different allocation pattern. The professional norm is to keep `unsafe` blocks as small and as few as possible, wrap them behind a safe public API, document the invariant being upheld directly above the block, and run `cargo miri test` (an interpreter that detects a wide class of undefined behavior) on any code that touches raw pointers.

## Pitfalls to remember

- `dyn Trait` requires the trait to be **object-safe** — a trait with a generic method (`fn foo<T>(&self)`) or a method returning `Self` by value cannot be made into a trait object; the compiler error names the offending method.
- Forgetting `#[from]` on a `thiserror` variant means `?` won't auto-convert into it — you'll get a type-mismatch compile error at the `?` site, not a runtime surprise.
- `anyhow::Error` erases the concrete type; if a caller needs to branch on the specific error variant, `anyhow` was the wrong choice for that boundary — use a `thiserror` enum instead.
- `.unwrap()` on `Result`/`Option` in a request-handling path turns any failure into a full process panic (or, in a web framework, a panicking request handler) — reserve it for states you've actually proven cannot occur, not "the happy path I tested locally."
- `unsafe` doesn't mean "the compiler stops checking anything" — most of the code inside an `unsafe` block is still fully type- and borrow-checked; only the five listed operations bypass compiler verification.
