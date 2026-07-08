# OOP and Modern PHP

This file covers the object model, modern type system, and the ecosystem
conventions (PSR, Composer) that a backend candidate is expected to know. PHP
8.5 is the baseline; features introduced in 8.3 or 8.4 are noted.

---

## Classes and objects

```php
final class User {
    use Timestampable;

    private const CREATED_VIA_WEB = 'web';     // typed class constant (8.3)
    public private(set) string $displayName;   // asymmetric visibility (8.5)

    public function __construct(
        public readonly string $id,            // constructor property promotion
        public readonly string $email,
        ?string $displayName = null,
    ) {
        $this->displayName = $displayName ?? substr($email, 0, strpos($email, '@'));
    }

    public function __destruct() {
        // acquire/release rarely belongs here; prefer RAII wrappers
    }
}
```

Key points:

- The `final` keyword (class or method) prevents subclassing / override.
- A class can be `abstract`, `final`, neither, or `readonly`. `readonly class`
  makes all instance properties readonly and forbids dynamic properties.
- Properties may be typed (built-in or class name), nullable (`?T`), or union
  (`A|B`); intersection (`A&B`) and DNF (`(A&B)|C`) are allowed on
  parameters/returns/properties; see § Type declarations.
- Property hooks (8.4) and asymmetric visibility (8.5) reduce boilerplate
  for computed/controlled properties.

### Constructors and constructor property promotion

Constructor property promotion (8.0+) combines parameter declaration and
property initialization in one syntax. Promoted parameters become properties
automatically and are assigned from the constructor arguments.

```php
class Point {
    public function __construct(
        public readonly float $x,
        public readonly float $y,
        public readonly float $z = 0.0,
    ) {}
}
```

`readonly` + promotion is the idiomatic immutable value object form in PHP.
Only one constructor declaration per class is allowed (no overloading);
default args emulate optional construction.

### Destructors

`__destruct()` runs when the object's refcount hits zero (the GC may delay
this for cycle participants). They are useful for close-after-scope patterns
but should be deterministic enough not to depend on time-sensitive state. PHP
resources (fopen, PDO) close on object destruction. Avoid heavy logic in
destructors; prefer explicit `close()`/`dispose()` for resources you truly
care about, because destructor timing is not deterministic with cycles.

## Visibility

`public`, `protected`, `private`. Methods and constants are visibility-bound;
in 8.5 properties can be **asymmetrically** visible (`public private(set)
$prop`). Inside a class, `private` means *this class only*, not "this object"
— objects of the same class see each other's private state (like Java, unlike
some other languages).

## Inheritance

Single inheritance only: a class extends one parent. `extends` is implicit for
the anonymous `stdClass`. Methods can be overridden with compatible
signatures (PHP enforces contravariant parameters and covariant return types
since 7.4; in practice a child method's signature must match the parent's or
be a compatible variant). Properties cannot be redeclared with a different
type.

```php
abstract class Animal {
    abstract public function sound(): string;
    public function describe(): string { return "A {$this->sound()} animal"; }
}

final class Dog extends Animal {
    public function sound(): string { return 'woof'; }
}
```

### static vs self vs parent

- `self::` refers to the class *where the method is literally written*,
  resolved lexically. This is the trap of Late Static Binding.
- `static::` is **late static binding**: it refers to the class on which the
  method was actually called at runtime. Use this when defining extension
  points in a base class — a common interview target.
- `parent::` refers to the parent of the class where the method is written.

```php
class Base {
    public static function make(): static { return new static; }   // LSB: returns caller class
    public static function makeSelf(): self { return new self; }   // always Base
}

class Sub extends Base {}

Sub::make();        // Sub instance
Sub::makeSelf();     // Base instance — silently surprising
```

When to use `static` as a return type: factory methods, "_returns an instance
of the class the caller invoked on_". When to use `self`: when you really
want exactly this class's behavior and not a child override.

### final

`final class` cannot be extended; `final method` cannot be overridden;
`final` constants cannot be redefined by a child. Use `final` aggressively on
value objects and to lock down extension points; it improves static analysis
and signals intent.

## Constants

Class constants can be typed since PHP 8.3:

```php
class Http {
    public const int STATUS_OK = 200;
    protected const array SAFE_METHODS = ['GET', 'HEAD', 'OPTIONS'];
}
```

Type enforcement at declaration time prevents `STATUS_OK = '200'` typos that
would otherwise rip through to a runtime bug. Interface constants work too;
enums have non-backed constants but cannot have backing values on constants.

## Interfaces, abstract classes, traits

### Interfaces

```php
interface Stringable {
    public function __toString(): string;
}

interface Repository {
    /** @return User[] */
    public function all(string $id): array;
    public function find(string $id): ?User;
}
```

- Multiple interface implementation: `class X implements A, B, C`.
- Interfaces can declare constructors (controversial; restricts
  implementations).
- `interface` methods are implicitly `public`.

### Abstract classes

Abstract classes carry partial implementation, single inheritance. Use an
abstract class when you have a true is-a relationship and shared default
behavior; use an interface when you want a contract that can be satisfied by
unrelated classes.

**Interfaces vs abstract classes — when to use which (common question):**

| | abstract class | interface |
| --- | --- | --- |
| Inheritance | single | many |
| State | fields allowed | no state (constants only) |
| Concrete methods | yes | no (interfaces have no default methods in PHP — only class and trait methods can carry bodies) |
| Constructors | yes | yes (but constrains implementers) |
| Vocabulary preference | "is-a" with shared skeleton | "can-do" capability contract |

### Traits

Traits are compiler-level horizontal code reuse — copy-paste at compile time
into the using class. They cannot be instantiated, but can have abstract
methods, static state, and properties (state in traits is per-class, with
subtle collision semantics — prefer stateless traits).

```php
trait Equalable {
    public function equals(self $other): bool {
        return $this == $other;          // == juggling here; prefer value compare
    }
}

final class Money {
    use Equalable;
    public function __construct(public readonly int $cents, public readonly string $currency) {}
}
```

Trait conflict resolution with `insteadof` and `as`:

```php
trait A { public function hello(): string { return 'A'; } }
trait B { public function hello(): string { return 'B'; } }

class Greet {
    use A, B {
        B::hello insteadof A;
        A::hello as greetFromA;       // alias
    }
}
```

You can change visibility with `as` (e.g. `A::hello as protected`).
Trait static methods are reachable through the using class.

## Enums

Covered at length in `01-core-concepts.md` § Enums. Recap the interview
essentials:

- `enum X` (pure) and `enum X: int|string` (backed). Backed cases MUST have
  literal scalar values, unique within the enum.
- Enums may implement interfaces, use traits (stateless), declare constants.
- No instantiation, no state, no cloning. Cases are singleton instances.
- `Status::from('active')` throws on miss; `tryFrom` returns null.
- `Status::cases()` returns all cases in declaration order.
- Methods on enums go on the enum type itself; `match($this)` is the idiomatic
  per-case logic. For complex per-case behavior prefer implementing methods
  per-case via `match` returning closures, or via the "smart enum" package
  when state is needed.

## Type declarations

### Parameter and return types

```php
function f(int $x, ?string $name = null): ?User { /* ... */ }
function g(iterable $items): Traversable { /* ... */ }
function h(): never { throw new \RuntimeException('unreachable'); }
```

- `void` — no return value; implicit `return;` allowed; no implicit null.
- `never` (8.1+) — function either throws or terminates the script;
  covariant with `void` and any other return type.
- `mixed` — any value, including null; equivalent to no declaration but
  explicit. Cannot be made nullable (`mixed` already includes null).
- `callable` and closures: a `callable` accepts functions, static method
  strings `Class::method`, instance method strings `[$obj, 'method']`, and
  closures. Better type-checking is `Closure` if you specifically want a
  closure object.
- `self`, `parent`, `static` are class-contextual return/parameter types.

### Union types (8.0+)

```php
function score(int|float $x): int|float { /* */ }
function parse(string|Stringable $s): string { /* */ }
function get(mixed $x): int|string|false { /* */ }   // `false` here is literal

class Prop {
    public int|string $id;
}
```

- A union literal `false` and `null` are allowed in unions (8.0+ for `null`,
  8.1+ for standalone `false` and `true`).
- Unions are NOT ordered and not merely "any type at runtime". Variance is
  *contravariant* in parameters and *covariant* in returns; a child can
  narrow.

### Intersection types (8.1+)

`A&B` requires a value to be both `A` and `B`. Only class/interface types;
no scalars; no `null`/`false`/`iterable`.

```php
function process(Iterator&Countable $x): void { /* $x has both interfaces */ }
```

### DNF types (8.2+)

Disjunctive Normal Form: a union of intersections, in canonical form.

```php
function dispatch((HasId&HasName)|AnonymousGuest $x): void { /* */ }
```

Parenthesized intersections inside unions: `(A&B)|(C&D)` allowed;
`A|(B&C)` yes; `(A|B)&C` is normalized to the equivalent DNF if it can be —
in practice you write the DNF yourself.

### Nullable shorthand

`?T` is sugar for `T|null` in declarations; with a union you can also expand
inline: `int|string|null`. `?T` cannot be combined with intersection types
(`?(A&B)` is not valid syntax); use `(A&B)|null` DNF instead.

## SOLID principles with PHP examples

### S — Single Responsibility

A class should have one reason to change. Coarse test:

```php
// Smelly: orchestrates persistence, e-mail, and validation.
class RegistrationService {
    public function register(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) { /* */ }
        $this->pdo->exec("INSERT INTO users ...");
        $this->mailer->send($email, 'Welcome');
    }
}

// Cleaner: each collaborator owns one concern.
final class RegisterUser {
    public function __construct(
        private UserRepository $users,
        private WelcomeMailer $mailer,
        private EmailValidator $validator,
    ) {}

    public function __invoke(string $email): User {
        $this->validator->assert($email);
        $user = $this->users->store(new User($email));
        $this->mailer->send($user);
        return $user;
    }
}
```

### O — Open/Closed

Open for extension, closed for modification. Achieve via interfaces and
polymorphism rather than `switch` on type.

```php
interface ShippingRate {
    public function cost(Money $base): Money;
}

final class StandardRate implements ShippingRate {
    public function cost(Money $base): Money { return $base; }
}
final class ExpressRate implements ShippingRate {
    public function cost(Money $base): Money { return $base->add(Money::usd(900)); }
}

final class Checkout {
    public function __construct(private ShippingRate $rate) {}
    public function total(Money $items): Money { return $this->rate->cost($items); }
}
```

Add a shipping variant = add a class; the existing `StandardRate` is not
modified.

### L — Liskov Substitution

Subtypes must be substitutable for their base types without surprising the
caller. Most commonly violated by tighter preconditions, looser
postconditions, throwing new exception types the caller cannot expect, or
narrowing nullability contracts on returns.

```php
abstract class Bird {
    abstract public function fly(): void;
}
final class Penguin extends Bird {           // violates LSP: cannot fly
    public function fly(): void { throw new \LogicException(); }
}
```

Better: model the variation as a capability interface (`Flyable`) so that
non-flying birds don't implement it.

### I — Interface Segregation

Clients shouldn't depend on methods they don't use. Prefer small focused
interfaces.

```php
interface Readable { public function read(): string; }
interface Writable { public function write(string $data): void; }

final class ReadOnlyStream implements Readable { /* only read() */ }
final class File implements Readable, Writable { /* both */ }
```

### D — Dependency Inversion

Depend on abstractions, not concrete classes. Wire at the composition root.

```php
interface InvoiceStore { public function save(Invoice $i): void; }

final class DbInvoiceStore implements InvoiceStore { /* pdo */ }
final class InMemoryInvoiceStore implements InvoiceStore { /* array */ }

final class IssueInvoice {
    public function __construct(private InvoiceStore $store) {}
    public function __invoke(Invoice $i): void { $this->store->save($i); }  
}
```

The high-level `IssueInvoice` doesn't know about PDO. Tests inject
`InMemoryInvoiceStore`.

## Composition over inheritance

Inheritance ("is-a") couples to a parent's API and lifecycle. Composition
("has-a") pulls capabilities together without that coupling, and lets
behavior change at runtime.

```php
// Composition: behaviors plugged in.
final class Duck {
    public function __construct(
        private QuackStrategy $quack,
        private FlyStrategy $fly,
    ) {}
    public function quack(): string { return $this->quack->quack(); }
    public function fly(): string   { return $this->fly->fly(); }
}
```

Why prefer it: easier to test (mock each behavior), no fragile inheritance
hierarchies, no LSP traps, supports multiple behaviors (single inheritance
can't), avoids the "diamond" problem.

## Dependency injection and the service container

Manual DI is fine for small apps; service containers (DIC) wire dependencies
centrally. The container resolves entries by ID/string or by class name,
instantiates them once (singleton scope is typical but per-request and
configurable), and injects constructor dependencies.

```php
interface Container {
    public function get(string $id): mixed;            // PSR-11
    public function has(string $id): bool;
}

final class AppContainer implements Container {
    public function __construct(
        private array $factories = [],
        private array $singletons = [],
    ) {}

    public function get(string $id): mixed {
        if (array_key_exists($id, $this->singletons)) { return $this->singletons[$id]; }
        $this->singletons[$id] = ($this->factories[$id])($this);
        return $this->singletons[$id];
    }

    public function has(string $id): bool { return isset($this->factories[$id]); }
}
```

Production frameworks (Symfony DI, PHP-DI, Laravel container) provide:

- **Autowiring** — reflect constructor parameters and resolve by type.
- **Service providers / definitions** — bound interfaces to concrete classes.
- **Decorators and aliases**, lazy proxies, parameter injection.
- **Compiled containers** for performance (Symfony dumps a PHP file at build
  time that wires everything in C-speed array access).

The cardinal sin of DI: calling `new Concrete()` deep in business logic. The
**composition root** (the application bootstrap, one place) should know about
everything; everything else receives its dependencies through its
constructor or factory.

## PSR standards

PSRs are interoperability standards published by the PHP-FIG. Notable ones:

- **PSR-1 / PSR-12** — coding style. PSR-12 (the successor to PSR-2) is the
  current baseline; tools like `php-cs-fixer` and `phpcs` enforce it.
- **PSR-4** — autoloading. A fully-qualified class name `\Acme\Foo\Bar`
  maps to a file at `<root>/Acme/Foo/Bar.php` (or whatever the namespace
  prefix maps to in `composer.json`).
- **PSR-3** — logging. The `LoggerInterface` from
  `psr/log`; concrete loggers (Monolog) implement it.
- **PSR-7 / PSR-15 / PSR-17** — HTTP messages, server request handlers
  (middleware), and HTTP factories. Used by Slim, PSR-7 middleware stacks,
  Nyholm PSR-7.
- **PSR-11** — container interface (`ContainerInterface`).
- **PSR-6** — caching (`CacheInterface`).
- **PSR-14** — event dispatcher.
- **PSR-20** — clock, modern time abstraction for testability.

### PSR-4 autoloading

`composer.json`:

```json
{
    "autoload": {
        "psr-4": { "Acme\\": "src/" }
    }
}
```

`\Acme\Order\Id` resolves to `src/Order/Id.php`. PSR-4 is *the* standard for
class-based packages; PSR-0 is deprecated. Classmaps (`classmap` autoload
section) are great for non-PSR-4 source; `files` autoload is for plain
function files (e.g. `symfony/polyfill-*`).

## Composer

Composer is the de-facto dependency manager. Key commands:

```bash
composer require psr/log                 # add runtime dep
composer require --dev phpunit/phpunit   # dev-only dep
composer install                          # install from composer.lock (CI, prod)
composer update                           # update composer.lock (CI controlled)
composer dump-autoload --optimize         # production: optimized classmap = speed
composer audit                            # security audit installed packages
composer outdated --direct                # show outdated direct deps
```

`composer.json` essentials:

```json
{
    "require":     { "psr/log": "^3.0" },
    "require-dev": { "phpunit/phpunit": "^11.0" },
    "autoload":    { "psr-4": { "Acme\\": "src/" } },
    "autoload-dev": { "psr-4": { "Acme\\Tests\\": "tests/" } },
    "scripts": {
        "test":    "phpunit",
        "lint":    "php-cs-fixer fix"
    }
}
```

Caret `^3.0` allows minor/patch updates within major 3; tilde `~3.0` allows
patch updates within minor. Lock file (`composer.lock`) pins exact versions
and hashes; commit it for applications, leave it out for libraries.

### Composer scripts

`composer test` runs the `test` script and `@` patterns (e.g. `post-autoload-dump`).
Useful hooks include `pre-install-cmd`, `post-install-cmd',
`pre-update-cmd`, `post-update-cmd` — but prefer `post-autoload-dump` for
accelerating deploy. Be careful: scripts run untrusted plugin code only if
allowed.

### Autoload performance

`composer dump-autoload --optimize --classmap-authoritative`:

- `--optimize` writes a classmap (the mapping of class name to file). With
  a classmap, autoload does **not** scan PSR-4 directories per request.
  Production should *always* use `--optimize`.
- `--classmap-authoritative` makes the classmap the single source of truth;
  the autoloader refuses to look at filesystem folders if a class isn't in
  the map. Prevents silent class drift and is faster; requires the map to be
  complete (run dump-autoload after any new class).
- `--apcu` caches the classmap in APCu to skip even the opcache lookup;
  combined with OPcache and JIT this is the conventional production baseline.

## Namespaces

```php
namespace Acme\Order;

use Acme\Money\Money;
use function Acme\Util\slugify;
use const Acme\Util\SEPARATOR;
```

Namespaces are declared at the top of the file; only one namespace clause per
file; `namespace { }` block syntax is allowed but discouraged in libraries.
Importing classes lets you refer to them by short name (`Money` instead of
`Acme\Money\Money`); key imports are **function and const imports** which
resolve at compile time based on the imported name rather than the global
namespace lookup rule. In a namespaced file, `strlen('x')` falls back to a
root lookup; explicitly `use function strlen;` (or `\strlen`) makes it
faster and unambiguous.

## Attributes (8.0+)

Attributes are structured metadata: classes annotated with
`#[Attribute]`, instantiated by the engine when you read them via reflection.
They replace doc-comment annotations in modern frameworks (Symfony, Doctrine
ORM 3).

```php
#[\Attribute(\Attribute::TARGET_METHOD)]
final class Route {
    public function __construct(public readonly string $path, public readonly array $methods = ['GET']) {}
}

final class Controller {
    #[Route('/users/{id}', methods: ['GET'])]
    public function show(string $id): void { /* ... */ }
}

$ref = new \ReflectionMethod(Controller::class, 'show');
foreach ($ref->getAttributes(Route::class) as $attr) {
    $route = $attr->newInstance();        // instance of Route
}
```

Attributes from the engine itself:

- `#[\Override]` (8.3+) — asserts the method overrides a parent; if it does
  not, the engine raises a compile-time fatal error. It is a static-analysis
  guard against typos like `setAlue` instead of `setValue` and against future
  removals of a parent method.
- `#[\AllowDynamicProperties]` — relic for legacy code; dynamic
  properties are disabled by default in 8.2+.

## Generics-ish patterns

PHP has no first-class generics syntax. Approximately sort yourself out with:

1. **PHPDoc generics** (`@template T`). Project-wide static analysis tools
   (Psalm, PHPStan) accept them; `phpdoc` works as a runtime-correct hint
   for humans and IDEs.
2. **Discriminated collections** at runtime: a `Collection<T>` class that
   carries a type filter or asserts on add.

```php
/**
 * @template T
 */
final class TypedCollection {
    /** @var list<T> */ private array $items = [];
    /** @param class-string<T> $class */
    public function __construct(private readonly string $class) {}

    /** @param T $item */
    public function add(mixed $item): void {
        if (!($item instanceof $this->class)) {
            throw new \InvalidArgumentException();
        }
        $this->items[] = $item;
    }

    /** @return list<T> */
    public function all(): array { return $this->items; }
}
```

Static-analysis-time generics via PHPStan/Psalm give you the bulk of the
safety without runtime cost.

## Fibers for cooperative concurrency (recap)

See `01-core-concepts.md` § Fibers for the API. From the architecture side:

- Fibers power event loops in AMP v3 / Swoole / Workerman / Revolt's
  scheduler. The application code reads sync; the event loop switches fibers
  on I/O readiness rather than blocking the thread.
- The pattern is: register the operation with the loop, `Fiber::suspend()`
  the current fiber; the loop later resumes the fiber with the result.
- Critical caveat: every library call from inside a fiber's stack must be
  non-blocking or you stall the entire process. Distinct from "true" thread
  parallelism. Native OS threads come from `ext-parallel` or from the SAPI
  itself (Swoole runs many fibers on worker pthreads).