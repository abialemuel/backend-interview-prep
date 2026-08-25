# Core Concepts

## Design philosophy

Rust was started at Mozilla to build a safer browser engine (Servo) without giving up C++-level performance. The pillars:

- **Memory safety without a garbage collector.** The compiler tracks ownership of every value at compile time and inserts deallocation calls automatically. There is no runtime GC, no GC pauses, and (outside `unsafe`) no use-after-free, double-free, or data races — these are compile errors, not runtime bugs.
- **Zero-cost abstractions.** High-level constructs (iterators, closures, generics, `async`) compile down to code as fast as the hand-written equivalent. You don't pay a runtime tax for writing idiomatic Rust.
- **Explicitness over magic.** No implicit conversions between numeric types, no null, no exceptions, no implicit sharing of mutable state. Every place a value can fail, alias, or move is visible in the type system.
- **Fearless concurrency.** The same ownership rules that prevent memory bugs also prevent data races. Covered in depth in `02-concurrency.md`.
- **A single, opinionated toolchain.** `cargo` handles building, testing, dependency resolution, and publishing; `rustfmt` and `clippy` are the de facto formatter and linter, both usually run in CI.

Trade-offs worth stating plainly, because interviewers want to hear you know them: the borrow checker has a real learning curve, compile times are slower than Go's (though improving release over release), and the language surface is larger than Go's — traits, generics, lifetimes, and macros all interact. None of this is free; you're trading developer iteration speed for compile-time-verified safety and predictable runtime performance.

## Program structure

A Rust project managed by Cargo has a standard layout:

```
myservice/
├── Cargo.toml       # package metadata + dependencies
├── Cargo.lock       # pinned dependency versions (committed for binaries)
└── src/
    └── main.rs       # binary crate root (or lib.rs for a library crate)
```

```rust
fn main() {
    let name = std::env::args().next().unwrap_or_default();
    println!("running {name}");
}
```

`cargo run` compiles and runs; `cargo build --release` produces an optimized binary. Unlike Go, there is no built-in single-binary-with-embedded-runtime story for cross-compilation — you still get a static-ish binary on Linux with the default `glibc` target, or a fully static one by targeting `musl`.

## Variables and mutability

```rust
let x = 5;          // immutable by default
let mut y = 10;      // must opt into mutability explicitly
y += 1;

let x = x + 1;       // shadowing: a new binding, not a mutation
const MAX_CONN: u32 = 100; // must have an explicit type, evaluated at compile time
```

Immutability is the default — the opposite of most languages. This isn't stylistic; it's load-bearing for the borrow checker, which reasons differently about `&T` vs `&mut T` references depending on whether the underlying binding is `mut`.

## Basic types

| Category | Types |
|----------|-------|
| Integer | `i8/16/32/64/128`, `u8/16/32/64/128`, `isize`/`usize` (pointer-sized) |
| Float | `f32`, `f64` |
| Boolean | `bool` |
| Character | `char` (4-byte Unicode scalar value, not a byte) |
| String | `String` (owned, heap, growable), `&str` (borrowed string slice) |
| Compound | tuple `(T, U, ...)`, array `[T; N]` (fixed size, stack), `Vec<T>` (growable, heap) |

Integer overflow **panics** in debug builds and **wraps** in release builds by default — a frequent interview gotcha. Use `wrapping_add`, `checked_add`, or `saturating_add` when the behavior must be explicit and portable across build profiles.

## Ownership

Rust's ownership rules, stated exactly as the compiler enforces them:

1. Each value has exactly one owner (a variable binding).
2. When the owner goes out of scope, the value is dropped (its destructor runs, memory is freed).
3. Ownership can be **moved** to another binding; after a move, the original binding is no longer valid.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // ownership of the heap data moves to s2
    // println!("{s1}"); // compile error: value borrowed here after move
    println!("{s2}"); // fine — s2 owns it now
}
```

This is different from a shallow copy: after `let s2 = s1`, `s1` is no longer usable at all — the compiler statically forbids the use, it doesn't just leave a dangling copy. Types that implement the `Copy` trait (all integers, floats, `bool`, `char`, and tuples/arrays of `Copy` types) are duplicated instead of moved, because copying them is as cheap as moving them:

```rust
let x = 5;
let y = x; // i32 is Copy — both x and y are valid
println!("{x} {y}");
```

Passing a value to a function moves it (unless the type is `Copy`); returning it moves ownership back out. `.clone()` explicitly performs a deep copy when you need two independent owners of heap data:

```rust
let s1 = String::from("hello");
let s2 = s1.clone(); // explicit, visible cost — both are independently valid
println!("{s1} {s2}");
```

## Borrowing and references

Instead of moving ownership every time you want to use a value, you can **borrow** it with a reference. The compiler enforces exactly one of these at any point in the code:

- Any number of immutable references (`&T`), **or**
- Exactly one mutable reference (`&mut T`)

— never both at once, and never a reference that outlives the data it points to (no dangling references).

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{r1} and {r2}"); // last use of r1/r2

    let r3 = &mut s; // fine — r1/r2's borrows have already ended
    r3.push_str(", world");
    println!("{r3}");
}
```

This compiles because of **non-lexical lifetimes** (NLL, stable since the 2018 edition): a borrow's lifetime ends at its last use, not at the end of its enclosing block. Had `r1` been used *after* `r3` was created, this would be a compile error: `cannot borrow s as mutable because it is also borrowed as immutable`.

Function signatures make borrowing explicit:

```rust
fn calculate_len(s: &String) -> usize {
    s.len() // borrows s, doesn't take ownership; s is still usable by the caller
}

fn append_world(s: &mut String) {
    s.push_str(" world");
}
```

## Lifetimes

A lifetime is the compiler's name for "how long is this reference valid." Most of the time it's inferred; you write an explicit lifetime annotation when the compiler can't determine, from the function signature alone, how the lifetimes of inputs relate to the lifetime of the output.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long string");
    let s2 = String::from("short");
    let result = longest(s1.as_str(), s2.as_str());
    println!("longest: {result}");
}
```

`'a` doesn't change how long anything actually lives — it's a constraint the compiler checks: *the returned reference will not outlive the shorter of `x` and `y`*. Without the annotation, the compiler has no way to know whether the return value borrows from `x`, from `y`, or is unrelated, so it refuses to compile ambiguous cases.

Structs that hold references need a lifetime parameter too, because the compiler must guarantee the struct doesn't outlive the data it borrows:

```rust
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn announce(&self, heading: &str) -> &str {
        println!("Attention: {heading}");
        self.part
    }
}
```

**Elision rules** mean you rarely write lifetimes on plain functions: if there's exactly one input reference, the output gets that lifetime; if `&self` is a parameter, the output gets `self`'s lifetime. You mostly write explicit lifetimes on structs holding references and on functions with multiple, differently-scoped reference inputs.

!!! note
    In interviews, the practical answer to "when do you need an explicit lifetime" is: almost never in typical backend code, because most data crossing function boundaries is owned (`String`, `Vec<T>`) rather than borrowed. Reach for owned types by default; borrow (and annotate lifetimes) when profiling shows the clones are actually expensive.

## Structs

```rust
struct User {
    id: u64,
    email: String,
    active: bool,
}

// Tuple struct — fields have no names, useful for lightweight wrappers ("newtype" pattern)
struct UserId(u64);

// Unit struct — no fields, often used as a marker type
struct Marker;

impl User {
    fn new(id: u64, email: String) -> Self {
        User { id, email, active: true }
    }

    fn deactivate(&mut self) {
        self.active = false;
    }
}

fn main() {
    let mut u = User::new(1, String::from("a@example.com"));
    u.deactivate();
}
```

`#[derive(...)]` generates common trait implementations without boilerplate:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}
```

`Debug` enables `{:?}` formatting, `Clone` enables `.clone()`, `PartialEq` enables `==`. This is the Rust equivalent of Go's implicit interface satisfaction — except here you opt in explicitly per trait, per type.

## Enums and pattern matching

Rust enums are **algebraic data types**: each variant can carry its own data, not just a name.

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn describe(addr: &IpAddr) -> String {
    match addr {
        IpAddr::V4(a, b, c, d) => format!("{a}.{b}.{c}.{d}"),
        IpAddr::V6(s) => s.clone(),
    }
}
```

`match` must be **exhaustive** — the compiler rejects a `match` that doesn't cover every variant (or lacks a `_` catch-all). This is enforced at compile time, unlike a Go `switch` on an enum-like constant, which silently does nothing if you forget a case.

```rust
let n = 4;
match n {
    0 => println!("zero"),
    1..=3 => println!("small"),
    _ if n % 2 == 0 => println!("even"),
    _ => println!("other"),
}
```

`if let` and `while let` are shorthand for matching a single pattern and ignoring the rest:

```rust
let config_max = Some(3u8);
if let Some(max) = config_max {
    println!("max is {max}");
}
```

## `Option` and `Result` — why Rust has no null or exceptions

Rust has no `null`. The absence of a value is modeled explicitly with the `Option<T>` enum, which the standard library defines as:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

You cannot accidentally dereference a "null" `Option<T>` — the compiler forces you to `match`, `if let`, or call a combinator (`.unwrap_or()`, `.map()`, `.and_then()`) before you can get at the inner `T`. This eliminates null-pointer-dereference bugs by construction, at the cost of writing `Some`/`None` handling explicitly wherever a value might be absent.

```rust
fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn main() {
    match divide(10.0, 0.0) {
        Some(result) => println!("result: {result}"),
        None => println!("cannot divide by zero"),
    }
}
```

Recoverable errors are modeled the same way, via the standard library's `Result<T, E>` — shaped like `Option`, but the "nothing" case carries an error value instead of being empty:

```rust
// enum Result<T, E> { Ok(T), Err(E) } — from std, shown for shape only.

fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
    s.parse::<u16>()
}
```

Rust has no exceptions for ordinary error handling. A function that can fail returns `Result<T, E>`; the caller is statically forced to handle both branches (or explicitly propagate/discard, which is visible in the code). `panic!` exists for unrecoverable programmer errors — the equivalent of a Go panic, not the equivalent of a Java/Python exception used for control flow. Error-handling idioms (the `?` operator, `thiserror`, `anyhow`, when to panic) get a full treatment in `03-traits-error-handling-and-patterns.md`.

## Traits and generics (preview)

A trait is Rust's interface: a set of methods a type can implement.

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, {}", self.name)
    }
}

fn print_greeting(g: &impl Greet) {
    println!("{}", g.greet());
}
```

Unlike Go interfaces, trait implementation is **explicit** (`impl Greet for Person`) rather than structural — a type only satisfies a trait if someone wrote the `impl` block, either in the type's own crate or in the trait's crate (the "orphan rule"). Generics are constrained by trait bounds:

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

Static dispatch, monomorphization, `dyn Trait`, and trait objects are covered in depth in `03-traits-error-handling-and-patterns.md`.

## The module system

```rust
mod network {
    pub mod server {
        pub fn start() {
            println!("server starting");
        }
    }

    fn internal_helper() {} // private by default, even to sibling modules unless pub
}

fn main() {
    network::server::start();
}
```

- Everything is **private by default**; `pub` opts a module, function, struct, or field into visibility from outside its defining module.
- A **crate** is the unit of compilation (a binary or a library); a **package** (what `Cargo.toml` describes) can contain one library crate and multiple binary crates.
- `use` brings a path into scope, analogous to an import; `pub use` re-exports it, letting a crate flatten its public API surface independently of its internal module layout.
- A **workspace** (a top-level `Cargo.toml` with a `[workspace]` table listing member crates) is how a multi-service Rust repo shares a single `Cargo.lock` and target directory across crates — the rough equivalent of a Go multi-module workspace.

## Smart pointers

Ownership is single-owner by default, but real programs need shared ownership, heap indirection, and interior mutability. The standard library provides these as ordinary types, not language features:

| Type | Use when | Thread-safe? | Cost |
|------|----------|---------------|------|
| `Box<T>` | You need heap allocation with a single owner — recursive types, or a trait object (`Box<dyn Trait>`). | Yes (if `T: Send`) | One allocation, otherwise free |
| `Rc<T>` | Multiple owners of the same data, single-threaded. | No | Reference-count increment/decrement, no atomics |
| `Arc<T>` | Multiple owners of the same data, across threads. | Yes | Reference-count via atomics — slightly more expensive than `Rc` |
| `RefCell<T>` | You need to mutate data behind a shared (`&`) reference, single-threaded, with borrow rules checked at runtime instead of compile time. | No | A runtime borrow-flag check per access; panics on violation |
| `Mutex<T>` / `RwLock<T>` | Same as `RefCell`, but safe across threads (blocks instead of panicking on contention). | Yes | Lock acquisition; covered in `02-concurrency.md` |

`Box<T>` unlocks recursive data structures, because the compiler needs to know a type's size at compile time and a type that directly contains itself has no finite size:

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    let _ = list;
}
```

`Rc<T>` gives shared ownership when it's genuinely unclear which owner should be responsible for dropping the value — common in graph-like structures (a cache entry referenced by multiple indexes, a tree node with a parent pointer):

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("shared"));
    let b = Rc::clone(&a); // bumps the refcount; no deep copy
    println!("refcount = {}", Rc::strong_count(&a));
    println!("{a} {b}");
}
```

`RefCell<T>` is how you get "mutate through a shared reference" in single-threaded code — the borrow-checker's compile-time rule (one mutable XOR many immutable) is deferred to a runtime check that panics on violation instead of refusing to compile:

```rust
use std::cell::RefCell;

struct Cache {
    value: RefCell<Option<i32>>,
}

impl Cache {
    fn get_or_compute(&self, compute: impl FnOnce() -> i32) -> i32 {
        if let Some(v) = *self.value.borrow() {
            return v;
        }
        let v = compute();
        *self.value.borrow_mut() = Some(v);
        v
    }
}
```

The combination `Rc<RefCell<T>>` — shared ownership *and* interior mutability — is common enough to be a named pattern for single-threaded shared mutable state:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
    let clone1 = Rc::clone(&shared);
    clone1.borrow_mut().push(4);
    println!("{:?}", shared.borrow());
}
```

Its multi-threaded equivalent, `Arc<Mutex<T>>`, is one of the most common patterns you'll write and be asked about in a Rust backend interview — covered fully in `02-concurrency.md`.

## Pitfalls to remember

- Moving a value inside a loop or closure often surprises newcomers: `let v = data; for _ in 0..3 { use(v); }` fails to compile on the second iteration's *would-be* move — the compiler catches it on the first pass, not silently at runtime.
- `Vec<T>` indexing (`v[i]`) panics on out-of-bounds; `v.get(i)` returns `Option<&T>` for a checked, non-panicking read. Prefer `.get()` at any boundary handling untrusted input (e.g., request-derived indexes).
- Integer overflow panics in debug builds, wraps silently in `--release` by default — don't assume your dev-build behavior matches production without checking `overflow-checks` in `Cargo.toml`'s release profile.
- `RefCell::borrow_mut()` while another borrow is live panics with `already borrowed: BorrowMutError` — this is the runtime cost of deferring the borrow check; keep borrow scopes as short as possible (drop the `Ref`/`RefMut` before the next borrow).
- `Rc<T>` is not thread-safe and does not implement `Send`; passing one across a `std::thread::spawn` boundary is a compile error, not a runtime race — this is exactly the "fearless concurrency" guarantee in action, covered next.
