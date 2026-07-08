# Architecture and Lifecycle

## The request lifecycle

Every HTTP request in Laravel flows through a small, well-defined pipeline. Understanding this pipeline is the single most valuable thing you can do for an interview because almost every other topic (middleware, the container, providers, facades) hangs off it.

```
public/index.php
   └── bootstrap/app.php            (creates the Application, binds kernels & handler)
        └── HTTP Kernel             (App\Http\Http in Laravel 11/12, configured in bootstrap/app.php)
             ├── global middleware   (Pipeline of middleware)
             ├── Router
             │    ├── route middleware / middleware groups
             │    └── controller (resolved via the container)
             └── response
```

`public/index.php` is the only entry point the web server should hit. In Laravel 11+ it is intentionally tiny:

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Foundation\Application;

define('LARAVEL_START', microtime(true));

require __DIR__.'/../vendor/autoload.php';

/** @var Application $app */
$app = require_once __DIR__.'/../bootstrap/app.php';

$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);
```

The Application instance is created in `bootstrap/app.php`, which in Laravel 11/12 is where middleware, exception handling, routing, and console/commands configuration lives instead of in dedicated kernel classes:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->append(App\Http\Middleware\TrimStrings::class);
        $middleware->alias(['admin' => App\Http\Middleware\IsAdmin::class]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->shouldRenderJsonWhen(fn (Request $request, Throwable $e) =>
            $request->is('api/*') || $request->expectsJson()
        );
    })
    ->create();
```

The HTTP kernel does three things:

1. Collects and runs the global middleware stack.
2. Dispatches the request to the Router, which selects the matching route, applies route middleware, and resolves the controller via the container.
3. Receives a `Response` and calls `terminate()` (which fires `terminable` middleware and `app->terminate()` lifecycle hooks).

The bootstrappers that bootstrap the framework (load environment, config, providers, facades) are handled internally; in older versions these were enumerated in `Kernel::bootstrappers`. In 11+ the `Application::configure()` flow runs an equivalent ordered set.

## The Service Container

The container (`Illuminate\Container\Container`) is the heart of Laravel. It is a PSR-11-compatible dependency injection container that knows how to resolve classes via reflection, can hold bindings keyed by interface, and supports contextual and tagged resolution.

### bind / singleton / scoped

```php
$this->app->bind(PaymentGatewayContract::class, function ($app) {
    return new StripeGateway(config('services.stripe.secret'));
});

$this->app->singleton(CacheStore::class, fn ($app) => new ArrayStore());

$this->app->scoped(Cart::class, fn ($app) => new Cart());
```

- `bind` resolves a new instance every time.
- `singleton` resolves once and caches for the entire request/CLI lifetime.
- `scoped` resolves once per "scope" — for HTTP, the request; for CLI, the command. Useful for per-request stateful services.

### Automatic resolution

When a binding is not registered, Laravel resolves the class by reflecting on its constructor and recursively resolving each parameter.

```php
class InvoiceController
{
    public function __construct(
        protected InvoiceRepository $invoices,
        protected LoggerInterface $log,
    ) {}
}
```

For `LoggerInterface` to resolve you must have a binding (typically in a service provider) telling the container which concrete class to use.

### Contextual binding

When class A needs a different cache instance than class B:

```php
$this->app->when(OrderProcessor::class)
    ->needs(CacheStore::class)
    ->give(fn () => new RedisStore());

$this->app->when(ReportService::class)
    ->needs(CacheStore::class)
    ->give(fn () => new ArrayStore());
```

### Primitive binding

```php
$this->app->when(ReportExporter::class)
    ->needs('$format')
    ->give('pdf');
```

### Tagged bindings

```php
$this->app->tag([
    StripeGateway::class,
    PaypalGateway::class,
], 'gateways');

$gateways = $this->app->tagged('gateways');
foreach ($gateways as $gateway) { /* ... */ }
```

### Interface to implementation binding

Bind interfaces so type-hints in controllers resolve to a concrete class:

```php
$this->app->bind(
    \App\Contracts\PaymentGateway::class,
    \App\Services\StripeGateway::class
);
```

## Service Providers

Providers are the only place where you should register things in the container, listeners, listeners, validation rules, and Artisan commands. Everything else (routes, middleware) lives in `bootstrap/app.php` and `routes/`.

Each provider has two phases:

- `register()` — only register things in the container and bind interfaces. Do not use services that depend on other providers here because their boot phase may not have run.
- `boot()` — everything is registered; safe to use any service, dispatch events, publish routes, etc.

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(
            \App\Contracts\PaymentGateway::class,
            \App\Services\StripeGateway::class,
        );
    }

    public function boot(): void
    {
        \App\Models\Transaction::observe(\App\Observers\TransactionObserver::class);
    }
}
```

### Provider registration in 11+

In Laravel 11+ providers are registered in `bootstrap/providers.php` instead of `config/app.php`:

```php
<?php

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\PaymentServiceProvider::class,
];
```

The default skeleton ships only `AppServiceProvider`. `AuthServiceProvider`, `RouteServiceProvider`, `EventServiceProvider` etc. no longer exist; route and event configuration moved to `bootstrap/app.php` and `Event::discover()`.

### Deferred providers

A provider may implement `DeferrableProvider` and declare `provides()` to only be loaded when one of the listed bindings is resolved. This saves boot time for rarely used services.

```php
class HeavyServiceServiceProvider extends ServiceProvider implements \Illuminate\Contracts\Support\DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(HeavyService::class, fn () => new HeavyService());
    }

    public function provides(): array
    {
        return [HeavyService::class];
    }
}
```

## Facades

A facade is a class that extends `Illuminate\Support\Facades\Facade` and provides a static-ish interface to a resolved-by-container service. `Cache::get('foo')` is not a static call; the facade's `__callStatic` resolves the bound service from the container (`CacheManager`) and forwards the call.

```php
namespace App\Services;

use Illuminate\Support\Facades\Cache;

class TaxService
{
    public function getRate(string $country): float
    {
        return Cache::remember("tax.{$country}", 3600, function () {
            // ...
        });
    }
}
```

The protected `$accessor` property declares the container key. The default `Cache` facade uses `'cache'`, which resolves to the bound service.

### Class aliases

Aliases in `config/app.php` (or via `AliasLoader`) let you write `Cache` instead of the full facade namespace. In 11+ aliases are still defined in `config/app.php` `aliases` array.

### How `__callStatic` works (interview gold)

```php
abstract class Facade
{
    public static function __callStatic($method, $args)
    {
        $instance = static::resolveFacadeInstance(static::getFacadeAccessor());
        return $instance->$method(...$args);
    }
}
```

The facade resolves the accessor (a string key, an object, or a closure) into an instance from the container, then forwards the call. This is why facades are easily testable: `Cache::shouldReceive('remember')->once()->andReturn(0.2)` swaps in a mock for that accessor.

### Facades vs DI trade-offs

- **Facades** are concise, easy to swap with mocks via fakes, and decouple your call sites from the container. Their downside is they hide dependencies and can encourage coupling to the framework's global state.
- **DI** makes dependencies explicit in constructors, is easier to refactor and test without framework knowledge, and is the more idiomatic choice for service classes.
- Rule of thumb: controllers and short-lived code often use facades; services and domain objects take dependencies through constructors. You can make a class-testable facade at any time using `Cache::fake()` style fakes — these fakes work via the container.

## Contracts

Contracts are the interfaces facades expose. `Illuminate\Contracts\Cache\Repository` is what `Cache` resolves to. Coding against contracts (constructor-injecting `Repository`) rather than facades gives you the testability of DI and the swappability of interfaces without the static-call appearance.

## The middleware pipeline

Middleware runs as a pipeline: each middleware receives the request, calls `$next($request)`, optionally mutates the response, and returns it.

### Global, group, and route middleware

In 11/12 middleware is configured in `bootstrap/app.php` via the `withMiddleware` closure rather than in `app/Http/Kernel.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    // Append a global middleware that runs after the defaults.
    $middleware->append(\App\Http\Middleware\TrustProxies::class);

    // Remove a default middleware.
    $middleware->remove(\Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class);

    // Disable the api group or stateful API behavior.
    $middleware->api([
        \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    ]);

    // Aliases usable as `->middleware('admin')`.
    $middleware->alias([
        'admin' => \App\Http\Middleware\IsAdmin::class,
        'throttle.api' => \App\Http\Middleware\ThrottleByUser::class,
    ]);
})
```

### Writing middleware

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class EnsureUserIsActive
{
    public function handle(Request $request, Closure $next)
    {
        if (! $request->user()?->is_active) {
            return abort(403, 'Inactive account.');
        }

        $response = $next($request);

        $response->headers->set('X-Audit', now()->toIso8601String());

        return $response;
    }
}
```

### Terminate middleware

A `terminate` method runs after the response is sent to the browser but before the process exits — useful for logging, analytics, and any work that doesn't need to delay the user:

```php
public function terminate(Request $request, Response $response): void
{
    // still runs during queue:work cycles
}
```

### Middleware parameters

```php
public function handle(Request $request, Closure $next, string $role)
{
    abort_unless($request->user()?->hasRole($role), 403);
    return $next($request);
}
```

Register as `'role' => \App\Http\Middleware\EnsureRole::class`, then use `->middleware('role:admin')`.

### Middleware priority

Defaults have a fixed order. To override, set `$middleware->priority` array or use `append`/`prepend` semantics via the closure API.

## Directory structure (Laravel 11+ slim skeleton)

Laravel 11 introduced a deliberately thin default structure. Notable changes versus 10.x:

- No `app/Http/Kernel.php`, no `app/Console/Kernel.php`. Configuration is done fluently in `bootstrap/app.php`.
- No `app/Http/Controllers` directory by default; controllers are still available, just not pre-created.
- No `app/Providers/EventServiceProvider` or `RouteServiceProvider`. Route registration moved to `bootstrap/app.php` `withRouting`. Event auto-discovery is enabled by default.
- `bootstrap/providers.php` lists providers instead of `config/app.php` `providers`.
- `routes/console.php` is the home for closure-based commands; command classes can live anywhere.
- A `/up` health check route is built in.
- Config files are still in `config/` but a number ship with fewer comments and defaults.

A typical fresh skeleton:

```
app/
  Models/
  Providers/AppServiceProvider.php
  ...
bootstrap/
  app.php
  providers.php
config/
  app.php
  ...
database/
  migrations/
  factories/
  seeders/
public/
  index.php
routes/
  web.php
  api.php
  console.php
```

## Configuration

- `config/*.php` — PHP arrays of configuration values.
- `.env` — environment-specific values, loaded by `Dotenv` during boot; read via `env('KEY', 'default')`.
- `config:cache` caches all config into a single file and `env()` calls return `null` after caching — never call `env()` outside of config files.

```php
// config/services.php
return [
    'stripe' => [
        'secret' => env('STRIPE_SECRET'),
    ],
];

// anywhere else:
config('services.stripe.secret');
```

`php artisan config:cache`, `route:cache`, `view:cache`, `event:cache`, and `optimize` prebuild these layers. In production use `optimize` and avoid `env()` in app code.

## Lifecycle hooks

- **Providers' `boot()`** — fires after all `register()` phases.
- **`App::booted`** — runs once the framework is fully booted.
- **`RequestReceived`, `RequestHandled`** — telescope/telemetry events.
- **`terminable middleware`** and `App::terminating()` callbacks run after the response is sent.

```php
$this->app->terminating(function () {
    // runs during $kernel->terminate(...)
});
```

For long-running processes (queue workers, Octane), the `register`/`boot` cycle runs once per worker start, not per job; per-job resets happen via `App::scoped` bindings and explicit resetting.
