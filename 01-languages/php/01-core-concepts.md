# PHP Core Concepts

This file covers the language primitives and the runtime model that every
backend interview candidate should be able to explain cold. Syntax targets
PHP 8.5; where a feature was introduced in 8.3 or 8.4 it is noted inline.

---

## Variables and types

PHP is dynamically typed: a variable's type is determined by its value at
runtime, and a variable can hold different types over its lifetime. Variables
are prefixed with `$` and are case-sensitive (function names are not).

```php
$x = 1;        // int
$x = "1";      // now string
$x = 1.5;      // now float
$x = [1, 2];   // now array
$x = null;     // now null
```

### Value vs reference semantics

PHP has three "families" of semantics:

- **Scalars** (`int`, `float`, `string`, `bool`) are passed by **value**
  (copy on assign). Strings are immutable byte arrays; "modifying" a string
  produces a new one.
- **Arrays** are **copy-on-write value types**, not references. Assigning an
  array to a new variable copies it (lazily until mutation). This is a common
  interview trap: arrays are *not* objects.
- **Objects** are passed by **handle** (object reference). The variable holds
  a handle to an object; assigning or passing it copies the handle, not the
  object. Mutating fields through the handle affects the shared object.
- **Resources** (mostly legacy — streams, sockets, GD) are also handle-like.

```php
$a = [1, 2, 3];
$b = $a;            // copy-on-write: $b is a separate array
$b[] = 4;           // $a is still [1,2,3]

$o1 = new stdClass;
$o1->x = 1;
$o2 = $o1;          // handle copy
$o2->x = 2;         // $o1->x is now 2 as well

$c = &$a;           // true reference: $c and $a are aliases
$c[] = 99;          // $a is now [1,2,3,99]
```

Explicit pass-by-reference for function parameters uses `&`:

```php
function inc(int &$n): void { $n++; }
$n = 5; inc($n);   // $n === 6
```

### Strict types

By default PHP performs coercive type juggling at function boundaries
(`add(1, "1")` runs and coerces `"1"` to `1`). Adding
`declare(strict_types=1)` as the *first* statement in a file makes type
declarations strict in that file's *caller* scope: a `string` argument will
not be coerced from `int`, and `int` will not coerce from `"1"`.

```php
declare(strict_types=1);

function add(int $a, int $b): int { return $a + $b; }

add(1, 2);          // ok
add(1, "2");        // TypeError, even though "2" looks numeric
```

Important subtlety: `strict_types` is determined by **the file that contains
the call**, not the file that declares the function. A library cannot force
strictness on its callers.

## Primitive types

The nine core types: `bool`, `int`, `float` (double-precision IEEE 754),
`string` (binary-safe byte array), `array`, `object`, `null`, `callable`,
`resource`. Plus `never`, `void`, `mixed`, `iterable`, `self`, `static`,
`parent`, `X|Y` unions, `X&Y` intersections, DNF types — covered in
`02-oop-and-modern-php.md`.

Integers are platform-dependent (64-bit on 64-bit builds). `String`s are
bytes, not Unicode code points: native string functions are byte-oriented;
`mb_*` functions handle multi-byte encodings. `Intl` and `grapheme_*` handle
grapheme clusters for collation/case.

## Arrays

PHP has one array type that doubles as a list and a map. Internally it is an
ordered hash table mapping keys to values. Keys are either `int` or `string`
(leading-digit strings without leading zeros are coerced to `int` keys;
`"01"` stays a string; floats truncate to int; `true`→`1`, `false`→`0`).
Insertion order is preserved for iteration — this is a defining property
unlike a typical hash map.

```php
$indexed  = [10, 20, 30];                 // list-like, keys 0,1,2
$assoc    = ['name' => 'Ada', 'age' => 36];
$multi    = [
    ['id' => 1, 'tags' => ['a', 'b']],
    ['id' => 2, 'tags' => ['c']],
];
$mixed    = [1 => 'a', 'b', 5 => 'c'];    // keys: 1, 2, 5
```

### Array operations and common functions

```php
count($arr);                      // length
in_array($x, $arr, true);         // membership; strict=true avoids == juggling
array_search($x, $arr, true);     // key of first match or false
array_key_exists('k', $arr);      // isset() is false on null values
array_merge($a, $b);              // reindex int keys; overwrite string keys
$a + $b;                          // union: $a's keys win, no reindex
array_replace($a, $b);            // like merge but preserves keys
array_slice($arr, 1, 2);          // subsequence
array_map(fn($x) => $x * 2, $arr);
array_filter($arr, fn($x) => $x > 0);
array_reduce($arr, fn($carry, $x) => $carry + $x, 0);
array_column($rows, 'name', 'id');// extract a column, index by 'id'
array_keys($arr); array_values($arr);
sort($arr); rsort($arr);
asort($arr); arsort($arr);        // sort preserving keys
usort($arr, fn($a, $b) => $a['x'] <=> $b['x']);
array_unique($arr);               // strict== by default since 8.0
list($a, $b) = $pair;             // destructuring; also [$a, $b] = $pair
['name' => $n] = $user;           // associative destructuring
```

PHP 8.4 added `array_find`, `array_find_key`, `array_any`, `array_all`:

```php
$found = array_find($users, fn($u) => $u->active);    // first match or null
$any  = array_any($users,  fn($u) => $u->admin);
$all  = array_all($users,  fn($u) => $u->verified);
```

Heavy nested array manipulation quickly becomes hard to read; for domain
data prefer typed value objects / DTOs.

## Strings

Strings are byte arrays. Double-quoted strings interpolate; single-quoted do
not (except `\\` and `\'`).

```php
$name = 'Ada';
$s1 = "Hello $name";             // 'Hello Ada'
$s2 = "Hello {$name}!";          // braces for complex expressions
$s3 = 'Hello $name';             // literal
$heredoc = <<<TXT
Hello $name
TXT;
$nowdoc  = <<<'TXT'              // no interpolation
Hello $name
TXT;
```

Common functions: `strlen` (bytes), `mb_strlen` (chars given encoding),
`substr`/`mb_substr`, `strpos`/`stripos`, `str_contains`/`str_starts_with`/
`str_ends_with` (8.0+), `explode`/`implode`, `trim`/`ltrim`/`rtrim`,
`str_replace`, `preg_*`, `sprintf`/`vsprintf`, `number_format`.

For correctness with non-ASCII, default to `mb_*` and set
`mb_internal_encoding('UTF-8')`.

## Operators

 arithmetic: `+ - * / % **`. `/` is float division; `intdiv()` for integer.
Bitwise: `& | ^ ~ << >>`. Comparison: `==` loose, `===` strict, `<=>`
spaceship (returns -1/0/1). Logical: `&& || ! and or xor` (the word forms
have lower precedence — usually avoid). Assignment: `=`, combined `+=` etc.,
null coalescing `??` and null-safe `?->`. Spread `...` in arrays/calls.

```php
$x = $data['name'] ?? 'default';     // null coalescing (isset-safe)
$city = $user?->address?->city;      // null-safe method/property chain
$result = match(true) {
    $x < 0  => 'neg',
    $x === 0 => 'zero',
    default => 'pos',
};
```

Error suppression `@` exists but is discouraged; prefer explicit checks or
`set_error_handler`.

## Control structures

`if/elseif/else`, `while`/`do-while`, `for`, `foreach`, `switch`, `match`,
`try/catch/finally`. `foreach` supports by-reference:

```php
foreach ($items as &$item) { $item['seen'] = true; }   // mutates in place
unset($item);                                            // break the ref
```

### match expression

`match` is an expression (returns a value), uses strict `===`, supports
single-arm conditions, and throws `\UnhandledMatchError` if no arm matches
and no `default` is present.

```php
$status = match($code) {
    200, 201      => 'ok',
    404            => 'not found',
    500, 502, 503  => 'error',
    default        => 'unknown',
};
```

### match vs switch

| Aspect | `switch` | `match` |
| ------ | -------- | ------ |
| Returns value | no | yes |
| Comparison | loose `==` (JIT-safe but juggling) | strict `===` |
| Fall-through | yes (needs `break`) | no, single arm |
| No-match behavior | falls through silently | throws `UnhandledMatchError` |
| Multiple values per arm | stacked `case` | comma list |
| Conditions as expressions | no | yes (`match(true) {...}`) |

Use `match` for value selection; keep `switch` for fall-through side effects
(rare) or when matching against complex expressions needing statement bodies.

## Functions

```php
function add(int $a, int $b = 0): int { return $a + $b; }

function sum(int ...$nums): int { return array_sum($nums); }  // variadic
sum(1, 2, 3);                                                  // 6

function html(string $tag, string $content, array $attrs = []): string { /* ... */ }
html('a', 'click', href: '/x', target: '_blank');              // named args (8.0+)
```

**Named arguments** (8.0+) let you skip defaults and self-document call sites.
Names are part of the API contract — renaming a parameter is a breaking
change.

**First-class callable syntax** (8.1+) creates a closure from any callable:

```php
$doubler = fn($n) => $n * 2;
$mapper = array_map(...);                 // Closure
$mapped = $mapper([1, 2, 3], $doubler);   // [2, 4, 6]
$strlen = strlen(...);                    // partial application only via closure
```

Passing `strlen(...)` produces a `Closure` that wraps `strlen`; you cannot
partially bind positional args with `...` alone, but you can with closures
and arrow functions.

### Variadic and unpacking

```php
function f(int ...$xs): array { return $xs; }
$args = [1, 2, 3];
f(...$args);                              // unpacking, equivalent to f(1,2,3)
```

## Closures and anonymous functions

An anonymous function is a `Closure` object. Closures capture by value from
the enclosing scope automatically; capturing by reference requires `use (&$x)`.

```php
$counter = function () {
    static $count = 0;       // static var persists across calls of this closure
    return ++$count;
};

$multiplier = 3;
$mult = function ($x) use ($multiplier) { return $x * $multiplier; };
```

`Closure::bind` and `Closure::bindTo` rebind `$this` and the scope (class).
This is how you make a closure execute in the context of an object:

```php
$getter = Closure::bind(function () { return $this->secret; }, $obj, ObjClass::class);
```

Binding also exposes private/protected fields when the scope matches. Useful
for serializers, hydrators, and tests; a footgun for security boundaries.

## Arrow functions

`fn(...) => expr` is a single-expression closure that auto-captures by *value*
from the surrounding scope (no `use` needed). For multi-statement bodies use
the standard `function () { ... }` form for multi-statement bodies.

```php
$multiplier = 3;
$mult = fn($x) => $x * $multiplier;          // captures $multiplier by value
```

There is no arrow-function-by-reference; for recursion bind a variable and
update via `use (&$x)`.

## Generators (yield)

A generator is a function that uses `yield`. Instead of computing a whole
array, it returns a `Generator` that is iterable and suspends between yields.
This makes lazy, possibly infinite, and memory-light pipelines possible.

```php
function naturals(): Generator {
    $n = 1;
    while (true) { yield $n++; }
}

function take(Generator $g, int $n): Generator {
    for ($i = 0; $i < $n && $g->valid(); $i++) { yield $g->current(); $g->next(); }
}

$firstFive = [];
foreach (take(naturals(), 5) as $v) { $firstFive[] = $v; }   // [1,2,3,4,5]
```

`yield from` delegates to another generator or traversable; `send($value)`
pushes a value into the generator (the `yield` expression evaluates to the
sent value); `return $x` inside a generator sets the return value retrievable
via `getReturn()` after completion.

Generator trade-offs vs arrays: lower memory and laziness, but per-yield
function-call cost; not indexable by position; not cache-friendly for random
access. Use generators for streaming/pipeline cases; use arrays when you
need random access or repeated iteration.

## Fibers (8.1+)

A `Fiber` is a stackful, cooperative concurrency primitive: a code block that
can suspend itself (and its whole call stack) and be resumed later from
outside. Unlike generators, fibers let you suspend from *any* depth of nested
calls, and they expose their full call stack so async frameworks (e.g.,
Revolt, AMP v3, Swoole coroutines) can build promise-free non-blocking I/O
without coloring every function `async`.

```php
$fiber = new Fiber(function (): void {
    $value = Fiber::suspend('from fiber 1');
    echo "resumed with: $value\n";
    Fiber::suspend('from fiber 2');
});

$start = $fiber->start();        // runs until first suspend; returns 'from fiber 1'
$fiber->resume('payload');       // resumes; $value = 'payload'; suspends again
$fiber->resume();                // fiber runs to end
```

Key truths:

- Fibers do **not** run in parallel threads; only one runs at a time within a
  PHP thread/process. They are about *cooperative suspend/resume*, not
  parallelism.
- Fibers share the single-threaded execution model; you still need an event
  loop / non-blocking I/O layer to actually achieve concurrency (multiple
  in-flight I/O operations).
- PHP's standard streams and PDO are blocking; the `ext-parallel` extension
  provides real OS threads for CPU parallelism, but is limited and finicky.

### Fibers vs generators

| | generators | fibers |
| --- | --- | --- |
| Stack | single frame (suspends one function) | full stack (suspend from any depth) |
| Resume mechanism | `->next()`, `->send()` | `->resume($val)` |
| Primary use | lazy iteration | cooperative async I/O / coroutines |
| Bidirectional data | yes (yield/send) | yes (suspend/resume) |
| adopted by frameworks | broadly | Swoole, AMP v3, Revolt-based |

## Enums (8.1+)

Enums are a special class hierarchy rooted at `\UnitEnum`. Two flavors:

- **Pure enums** (no value attached).
- **Backed enums** (scalar "backing value" of `int` or string`, useful for DB/API serialization).

```php
enum Status: string {
    case Active   = 'active';
    case Disabled = 'disabled';
    case Banned   = 'banned';

    public function isActive(): bool { return $this === self::Active; }

    public function label(): string {
        return match($this) {
            self::Active   => 'Active user',
            self::Disabled => 'Temporarily disabled',
            self::Banned    => 'Permanently banned',
        };
    }
}

Status::from('active');            // Status::Active
Status::tryFrom('nope');           // null
Status::cases();                   // [Active, Disabled, Banned]
```

To look up a case by name (no native API exists, contrary to some blog
posts), derive it yourself:

```php
function enumFromName(string $enum, string $name): ?\UnitEnum {
    foreach ($enum::cases() as $case) {
        if ($case->name === $name) { return $case; }
    }
    return null;
}
```

Enums can implement interfaces, have constants (non-backed), use static
methods, and use traits — **but cannot have properties** (state). Traits in
enums: useful for shared helper methods; cannot add state.

```php
trait Labelled {
    public function upper(): string { return strtoupper($this->label()); }
}

enum Priority: int {
    use Labelled;
    case Low = 1; case Med = 2; case High = 3;
    public function label(): string { return match($this) { /* ... */ }; }
}
```

Important enums gotchas:

- Backed values must be unique and literal (not expressions) within the enum.
- Enums cannot be instantiated, cloned, serialized destructively, or have
  constructor state.
- `instanceof` and `Enum::case === $x` are the canonical comparisons.
- Constants are allowed but static state on the enum class itself is unusual
  and discouraged for tests.

## readonly properties and classes

`readonly` (8.1) on a property prevents reassignment after initialization; the
property must be typed. `readonly class` (8.2) makes all instance properties
readonly and forbids dynamic properties.

```php
final class Money {
    public function __construct(
        public readonly int $cents,
        public readonly string $currency,
    ) {}
}

$m = new Money(100, 'USD');
// $m->cents = 200; // Error: cannot modify readonly
```

Readonly is not full immutability:

- Writes through the property are blocked entirely, including
  ` $obj->arr[] = $x` even when the property is an array — readonly applies
  to the *storage location*, not just to plain assignment.
- A readonly property holding an object can still have its inner object
  mutated (`$m->currency` is `string` and immutable, but if the value were a
  mutable object you could call its setters). Readonly guards the reference,
  not deep state.
- A readonly property cannot be unset or re-initialized within the class
  after construction. (PHP 8.3 added a narrow exception: a readonly property
  may be re-initialized exactly once during `__clone`, enabling
  clone-with-modification patterns.)

Deep immutability requires value semantics all the way down (readonly + value
objects/DTOs that themselves only hold readonly fields).

## Property hooks (PHP 8.4)

Property hooks let you override read and/or write behavior at the property
declaration site, removing most boilerplate getters/setters. A hook can
`get` and/or `set`; the set hook can validate or transform the value. The
virtual property may have no backing store at all (a "virtual hook").

```php
class Temperature {
    public function __construct(
        public private(set) float $celsius,    // asymmetric visibility (8.5) on a promoted prop
    ) {}

    public float $fahrenheit {                 // virtual property, computed from celsius
        get => $this->celsius * 9 / 5 + 32;
        set (float $f) { $this->celsius = ($f - 32) * 5 / 9; }
    }
}
```

Notes:

- Hooks are defined in the property declaration block. The `get` hook is an
  expression `{ get => expr; }` or a body `{ get { return ...; } }`.
- "Virtual" hooks (without a backing field) compute on each access; no storage.
- Backed hooks can use `$this->name` (the implicit backing field) inside the
  hook body. The compiler allocates the field automatically only if needed.
- Hooks are independent of asymmetric visibility; you can use them together
  (8.5+) for a public read / private write property with validation on the
  set hook.

## Asymmetric visibility (PHP 8.5)

Asymmetric visibility modifiers `protected(set)` and `private(set)` allow
the *write* scope of a property to be narrower than its read scope. The
read scope is taken from the leading visibility keyword (the `public` in
`public private(set)`); when no leading visibility is present, the read
scope defaults to `public`, so `public private(set)` and a bare
`private(set)` expose the same behavior. The explicit form is recommended.
Combined with hooks, this replaces the boilerplate of "public get / private
set" handwritten properties and is the idiomatic way in 8.5 to expose
read-only-to-the-outside state.

```php
final class User {
    public function __construct(
        public private(set) string $id,           // anyone can read; only this class can write
        public protected(set) string $email,      // this class + subclasses can write
    ) {}

    public function rename(string $email): void { $this->email = $email; }
}
```

Before 8.5, the same effect required either a private backing property plus a
public accessor method, or a virtual property hook whose `get` body wrapped a
private field while leaving writes scoped to that field:

```php
class Pre85User {
    private string $email;                  // private backing field
    public string $emailView {              // virtual hook: no own storage
        get => $this->email;
    }
    public function __construct(string $email) { $this->email = $email; }
    public function setEmail(string $email): void { $this->email = $email; }
}
```

Asymmetric visibility in 8.5 makes the intent declarative for the common
case. Asymmetric visibility works with **readonly**: readonly restricts
writes to the constructor itself, asymmetric visibility restricts the write
scope to a subset of visibility after construction (letting service methods
within the class mutate the value later).

## The request lifecycle: SAPI, FPM, shared-nothing

PHP is invoked per request by a **SAPI** (Server API): apache2handler,
cgi-fcgi, fpm-fcgi, cli, phpdbg, embed. Each SAPI adapts PHP to a host: it
provides `$_GET`/`$_POST`/`$_SERVER` globals, the output channel, and the
lifecycle hooks (module init, request init, request shutdown, module
shutdown).

### FPM (FastCGI Process Manager)

The dominant production SAPI for HTTP. A master process manages a pool of
**worker processes** that wait for FastCGI requests from a front-end web
server (nginx) over a socket. Each request is dispatched to an idle worker,
which runs the application once, then returns to the pool.

- Workers are stateless across requests: globals (`$_GET`, `$_SESSION` aside),
  static state, and *autoloaded classes* do not survive between requests by
  default (OPcache persists at the bytecode level, but runtime in-memory
  state is per-process and torn down per request).
- **Shared-nothing architecture**: each request starts with a fresh process
  state except what shared extensions bridge (OPcache bytecode, APCu user
  cache, shared memory, ext-redis, persistent DB connections via the
  pconnect-style pools). This is why PHP scales horizontally with simple
  semantics: there are no global long-lived application objects to corrupt
  across requests.
- Memory leaks within a request are reclaimed at request end; the per-request
  lifetime bounds accumulated state and is one reason PHP's practical leak
  tolerance is high.

### Per-request flow inside PHP

1. SAPI receives the request; minit modules (once at process start, not per
   request) and rinit modules (per request).
2. PHP compiles scripts to opcodes (cached by OPcache after first hit) and
   executes the requested file.
3. shuts down: rshutdown modules, free request memory, send response.
4. Worker returns to the pool.

### Tuning FPM

FPM pools have static/dynamic/ondemand modes. Typical dynamic config:
`pm = dynamic`, `pm.max_children`, `pm.start_servers`,
`pm.min_spare_servers`, `pm.max_spare_servers`. Size `max_children` so that
`max_children * per_request_memory < RAM` while leaving headroom for the
master, OPcache, APCu, and the OS. Each child is a separate process: heavy
RAM per child limits concurrency; OOM kills are the classic failure mode of
oversized pools. For CPU-bound work add more children; for I/O-bound
consider async (Swoole, FrankenPHP, RoadRunner) instead of more children.

## OPcache

OPcache is the shared bytecode cache built into PHP since 5.5. It caches
compiled opcodes keyed by file path (and timestamp or signature, depending
on `opcache.validate_timestamps`). On a warm FPM pool the second request for
a script skips the compiler entirely; this is the single largest performance
lever in most PHP deployments.

Key settings:

- `opcache.enable=1` (CLI: `opcache.enable_cli` for workers that benefit).
- `opcache.memory_consumption` — shared memory size for cached scripts.
- `opcache.max_accelerated_files` — hash table bucket count (rounds up to a
  prime); size above your script count + headroom.
- `opcache.validate_timestamps` — `1` (default, revalidate per request; safe
  but slower) vs `0` (must manually `opcache_reset()` or restart FPM on
  deploy, much faster). Production often sets `0` and clears cache on
  deploy.
- `opcache.revalidate_freq` — when timestamp validation is on, the check
  cadence in seconds.
- `opcache.preload` and `opcache.preload_user` — preload a script at startup
  (see `03-performance-and-security.md`).
- `opcache.jit*` — JIT toggles (see `03-performance-and-security.md`).

OPcache is per-host (shared memory) and per-binary: you cannot share across
different PHP builds. In shared hosts, APCu complements it for *user-level*
in-memory key-value caching since OPcache only stores bytecode, not app data.