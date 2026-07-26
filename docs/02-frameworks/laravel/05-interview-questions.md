# Laravel Interview Questions

Questions are grouped by difficulty, which maps to the level being assessed:

- **Easy (junior)** — a passing answer states the mechanism correctly. Seniors should answer these instantly and add one trade-off.
- **Medium (senior)** — a passing answer explains the mechanism *and* the trade-offs/failure modes. Juniors get partial credit for the mechanism alone.
- **Hard (staff)** — a passing answer covers operational reality: scaling, failure modes, deploy interactions, and when *not* to use the feature. These are usually asked as open-ended design follow-ups.

Each answer is a model answer suitable for repeating in an interview; add code snippets where they clarify the concept. Cross-references point back to the reference files. Answers are current as of **Laravel 13.x** (mid-2026).

## Easy (junior)

### Q1: Walk me through the Laravel request lifecycle from the browser to the response.

**Answer:** The web server hits `public/index.php`, which requires the Composer autoloader and builds the `Application` instance from `bootstrap/app.php`. That file (Laravel 11+) configures routing, middleware, and exception handling fluently, then `index.php` resolves the HTTP kernel from the container, calls `$kernel->handle($request)`, and sends the response. The kernel runs the global middleware pipeline, hands the request to the Router, which matches a route, applies its middleware groups, and resolves the controller (and its dependencies) via the container. The controller returns a `Response`; the kernel runs `terminate()` on the request, firing terminable middleware and `App::terminating()` callbacks. This pipeline is why middleware, form requests, and DI all compose so cleanly.

### Q2: What is the difference between `register()` and `boot()` in a service provider?

**Answer:** `register()` is for putting things into the container — bindings, singletons, configuration, listeners that just register closures. You must not rely on other providers' services here because their `boot()` may not have run yet; only register pure binding logic. `boot()` runs after every provider's `register()` has completed, so it is safe to use any service, attach observers, publish routes, and dispatch events. Order matters: bindings in `register`, consumption in `boot`. Deferred providers implement `DeferrableProvider` and only load when their `provides()` bindings are actually requested, which trims boot time for rarely used services.

### Q3: What is the difference between `$fillable` and `$guarded`?

**Answer:** Both control mass assignment on Eloquent models. `$fillable` is a whitelist of attributes allowed to be mass-assigned through `fill`, `create`, and `update`; anything not listed is silently dropped (or, with `Model::preventSilentlyDiscardingAttributes()`, throws). `$guarded` is a blacklist — every attribute is assignable except those listed. `$guarded = []` disables protection entirely, which is only safe when you fully control input. In practice, prefer explicit `$fillable` whitelists for models exposed to user input, and reserve `$guarded = []` for internal/seed-only models.

### Q4: When would you use the query builder instead of Eloquent?

**Answer:** Eloquent is built on top of the query builder, so they share the fluent API; the trade-off is overhead vs features. Use the query builder (`DB::table(...)`) for cross-table joins where you don't need model lifecycle, fast aggregate reads (`count`, `sum`, `min`), raw SQL reports, and read-heavy endpoints where hydrating thousands of models would be wasteful. Use Eloquent when you need casts, relationships, accessors/mutators, events, observers, scopes, factories, or any domain logic on the rows. You can mix both within a single request: `DB::table` for a fast count, `Model::find` for the few rows you actually need.

### Q5: What is route model binding and how do you customize it?

**Answer:** Route model binding resolves a model instance from a route parameter by type-hinting it in the closure or controller action. Implicit binding uses the parameter name (`{post}` + `Post $post`); Laravel queries by primary key. You can bind on a different column with `{post:slug}` and `Post $post`, set `getRouteKeyName()` on the model, or define an explicit `Route::bind('post', fn ($v) => Post::where('slug', $v)->firstOrFail())`. Enum binding works the same way with a backed enum. `->scopeBindings()` applies the model's global scopes during binding, which is useful for tenant-scoped routes. Implicit binding also supports soft-deleted rows when combined with `withTrashed()` in a custom binding.

### Q6: What is the difference between `{{ }}` and `{!! !!}` in Blade?

**Answer:** `{{ $value }}` passes the value through `htmlspecialchars` and is XSS-safe in an HTML body context — this is the default and almost always what you want. `{!! $value !!}` outputs raw, unescaped HTML and is only safe for content you generated yourself or sanitized with a library like `mews/purifier`. Note `{{ }}` does not protect against XSS in attribute contexts (e.g., `href="javascript:..."`) — you still need to validate and sanitize inputs, and never put untrusted values into `href`, `src`, event handlers, or `<style>` without filtering.

### Q7: How do factory states work?

**Answer:** A state is a method on a factory that returns `$this->state(fn (array $attrs) => [...])` to override or add attributes. States compose: `Post::factory()->count(5)->published()->withUser($user)->create()` runs each state's closure in order, then persists the result. States can take arguments, depend on the current attributes via the closure, and create nested relations with `for()` or `has()`. They keep test data realistic and remove the need to repeat long arrays across tests.

### Q8: What is the difference between `RefreshDatabase` and `DatabaseTransactions`?

**Answer:** `RefreshDatabase` runs migrations once per test run (skipping them if already migrated) and wraps each test in a transaction that rolls back on completion, so tests start with a clean schema and clean data. `DatabaseTransactions` only wraps the test in a transaction and assumes the schema is already migrated; it is faster but can't recover from migration mutations. Use `RefreshDatabase` for tests that touch schema (migrations, custom columns) or want a guaranteed clean slate, and `DatabaseTransactions` when migrations are expensive and you only insert/update rows. There is also `DatabaseTruncation` for cases where transactions don't work (concurrent connections, Dusk, DDL inside the test).

### Q9: How does `Model::preventLazyLoading` help, and when should you turn it on?

**Answer:** Lazy loading fires a query the moment you access a relationship that was not eager loaded, which causes the N+1 problem whenever you loop a collection. `Model::preventLazyLoading(! app()->isProduction())` makes Laravel throw `LazyLoadingViolationException` on any such access, turning a silent performance bug into a loud test failure. Enable it in `AppServiceProvider::boot()` for non-production environments, and combine with Telescope or Debugbar to see the queries. In production you typically leave it off to avoid breaking requests on edge cases.

### Q10: What changed structurally in Laravel 11 that still matters in 12/13?

**Answer:** Laravel 11 introduced the slim skeleton: `app/Http/Kernel.php` and `app/Console/Kernel.php` are gone, and middleware, exceptions, routing, and console commands are configured fluently in `bootstrap/app.php` via `withRouting`, `withMiddleware`, `withExceptions`, and `withSchedule`. Providers are registered in `bootstrap/providers.php` instead of `config/app.php`, and only `AppServiceProvider` ships by default — `EventServiceProvider` and `RouteServiceProvider` no longer exist (event auto-discovery is on by default, and route loading moved to `bootstrap/app.php`). The default skeleton has no `app/Http/Controllers` folder pre-created, closure commands live in `routes/console.php`, and a `/up` health route ships built in. Laravel 12 and 13 keep this structure unchanged — 12 added new starter kits, 13 (March 2026, PHP 8.3+) added the AI SDK, JSON:API resources, `Queue::route()`, vector search, and declarative PHP attributes, all without touching the skeleton.

## Medium (senior)

### Q11: Compare using the service container directly vs facades. What are the trade-offs?

**Answer:** Facades provide a concise static-style API backed by a container binding — `Cache::get('x')` resolves `cache` from the container and forwards the call. They are easy to use and trivially testable via `Cache::fake()` and `Cache::shouldReceive(...)`, which swap in a mock for that accessor. Their downside is they hide dependencies in the call site and couple your class to a framework global, which makes refactoring harder and obscures what a class actually needs. Constructor DI makes dependencies explicit, is framework-agnostic, and is the idiomatic choice for service/domain classes; you accept a contract (`CacheRepository`) rather than a facade. Rule of thumb: facades in controllers and quick glue code, DI in services, repositories, and domain objects.

### Q12: How do facades work under the hood?

**Answer:** A facade extends `Illuminate\Support\Facades\Facade` and declares a protected `$accessor` (string key, closure, or object). The magic `__callStatic` method resolves that accessor into an instance via `resolveFacadeInstance` — which, for string keys, asks the container for the binding — then forwards the call: `$instance->{$method}(...$args)`. This is why `Cache::remember(...)` is not a true static call; it is forwarded to the bound `CacheManager`. The testability comes from `Facade::swap($mock)` (used by `Cache::fake()`), which replaces the resolved instance for the accessor. Class aliases in `config/app.php` let you write `Cache` instead of the full facade namespace, but they are purely cosmetic — the same forwarding mechanism runs underneath.

### Q13: Explain the N+1 problem in Eloquent and how you fix it.

**Answer:** N+1 happens when you fetch N parent models (1 query) and then access a relationship on each, triggering N more queries — one per parent. The fix is eager loading: `Post::with('author')->get()` issues two queries (parents plus `WHERE author_id IN (...)`) instead of N+1. Use `withCount` to load counts without the rows, `load` to eager load on an already-fetched collection, and `loadMissing` to avoid re-loading. Constrain eager loading with `with(['comments' => fn ($q) => $q->latest()->limit(5)])` and select only needed columns (`with('author:id,name')` — the foreign key must be included so the join matches). Pair this with `Model::preventLazyLoading(true)` in dev to catch regressions automatically.

### Q14: What are polymorphic relationships and when do you use them?

**Answer:** A polymorphic relationship lets a model belong to more than one type of model on a single association — for example, a `Comment` that can belong to a `Post`, a `Video`, or a `Photo` via `commentable_type`/`commentable_id` columns. The "many" side uses `morphMany` or `morphToMany`; the owning side uses `morphTo`. Many-to-many polymorphic uses `morphedByMany` on the shared model (e.g., `Tag` belongs to many `Post`s and `Video`s). It is the right tool when the association is genuinely shared across heterogeneous types; the trade-off is that you lose a hard foreign key (the `type` column is a string), so `Relation::morphMap(['post' => Post::class, ...])` lets you alias the type and rename classes safely.

### Q15: How does mass assignment protection work and why is it important?

**Answer:** Mass assignment lets `::create($array)`, `->fill($array)`, and `->update($array)` set many attributes at once from user input, which is convenient but dangerous — `$request->all()` could include columns like `is_admin` or `role` that you never meant to expose. Eloquent guards against this with `$fillable` (whitelist) and `$guarded` (blacklist); attributes outside the whitelist (or on the blacklist) are silently dropped. In development you can opt into strict mode with `Model::preventSilentlyDiscardingAttributes()` so dropped attributes throw instead of vanishing. The classic Laravel CVE that prompted this was the 2013 mass assignment on `github.com`-style sites; the lesson is to always use `$fillable` for models exposed to user input and to never pass `$request->all()` to `::create` without scoping.

### Q16: How do accessors and mutators work, including the `Attribute` class?

**Answer:** The modern syntax uses a method named after the attribute returning an `Attribute` instance:

```php
protected function firstName(): Attribute
{
    return Attribute::make(
        get: fn (?string $v) => $v ? ucfirst($v) : null,
        set: fn (string $v) => strtolower($v),
    );
}
```

The `get` closure transforms the stored value when accessed (`$model->first_name`); `set` transforms input before persisting. The closure form caches the result on the instance, and derived attributes (no backing column) can be added to `$appends` so they appear in JSON. The older `getFirstNameAttribute`/`setFirstNameAttribute` methods still work but the `Attribute` syntax is more concise, composes better, and is the documented modern style since Laravel 9.

### Q17: What are middleware groups and how are they configured in Laravel 11+?

**Answer:** Middleware groups bundle middleware that runs together — `web` for browser routes (session, CSRF, cookies, share errors) and `api` for API routes (throttling, stateful if Sanctum). In Laravel 11+ groups and aliases are configured fluently in `bootstrap/app.php` via `withMiddleware`, not in `app/Http/Kernel.php` (which no longer exists). You can `$middleware->api([...])` to override the api group, `$middleware->alias([...])` to add route-level aliases, and `$middleware->priority([...])` to control ordering. A route picks a group with `->middleware('web')` or applies individual aliases. Custom groups can be defined by extending the `Middleware` configuration, but the two built-in groups cover most apps.

### Q18: When should you use a form request instead of inline validation?

**Answer:** Use a form request whenever a controller action has non-trivial rules, authorization, or custom messages — it extracts validation out of the controller, makes rules reusable, and gives you a typed `$request->validated()` array. A form request's `authorize()` gates access (returning `false` throws 403), `rules()` declares the contract, and `prepareForValidation()` lets you normalize input before rules run. Use inline `$request->validate([...])` for trivial one-off rules where a separate class would be overkill — e.g., a simple search endpoint with two fields. Both end up calling the same validator; the difference is organization, reuse, and testability. Injecting a form request type-hint also makes the controller signature self-documenting.

### Q19: What is the difference between queue workers and listeners?

**Answer:** Listeners are objects that respond to events; they run synchronously by default but become asynchronous when they implement `ShouldQueue`, in which case they are serialized and pushed to a queue like any other job. Queue workers (`queue:work`) are long-running processes that pop jobs off a queue and execute them; they keep the framework booted across jobs for speed and must be supervised (Supervisor, systemd, Horizon). A queued listener is dispatched via `Event::dispatch`, serialized with its event payload, and processed by the same workers that process explicit jobs. The distinction is the trigger: jobs are dispatched explicitly (`SomeJob::dispatch($x)`), listeners are dispatched implicitly when their event fires.

### Q20: How do `Bus::fake`, `Queue::fake`, and `Event::fake` differ?

**Answer:** All three short-circuit side-effecting work so you can assert on it without actually performing it. `Bus::fake` intercepts `dispatch` calls (and batches/chains) on the bus facade and lets you assert `assertDispatched`, `assertNotDispatched`, `assertBatched`, `assertChained`. `Queue::fake` intercepts jobs pushed to any queue connection and asserts `assertPushed`, `assertPushedOn`, but does not run them. `Event::fake([SpecificEvent::class])` short-circuits the event dispatcher — and importantly, **also prevents listeners from running** — so it is the choice when listeners have side effects (sending email, hitting APIs). The subtle difference: `Bus::fake` keeps jobs from running, `Queue::fake` keeps them from being pushed, `Event::fake` keeps listeners from running. Choose based on which layer you want to assert against.

### Q21: What did Laravel 13 add, and which additions actually change how you design an app?

**Answer:** Laravel 13 (March 2026, PHP 8.3+) was deliberately low-breakage; the skeleton is unchanged since 11. The headline additions: the first-party **AI SDK** (provider-agnostic text/image/audio generation, agents, `Str::toEmbeddings()`), **vector/semantic search** in the query builder (`whereVectorSimilarTo` over pgvector columns), **JSON:API resources** for spec-compliant API serialization, **`Queue::route()`** for centralizing job-to-queue routing, the **`PreventRequestForgery`** origin-aware CSRF middleware, `Cache::touch()` for extending TTLs, and expanded **PHP attributes** (`#[Middleware]`/`#[Authorize]` on controllers, `#[Tries]`/`#[Backoff]`/`#[Timeout]` on jobs). Design-wise the meaningful ones are: vector search removes the need for a separate vector store in RAG features; `Queue::route()` makes queue topology reviewable in one place; and attributes move cross-cutting config onto the classes it governs. The rest is quality-of-life. A good answer also notes what *didn't* change: the container, Eloquent, and the bootstrap flow are stable, which is why the upgrade is typically a one-day job.

### Q22: How would you roll out a risky feature gradually in Laravel?

**Answer:** Use **Laravel Pennant**, the first-party feature-flag package. Define a feature with a resolver — `Feature::define('new-checkout', fn (User $user) => $user->isInternal() || Lottery::odds(1, 100))` — and Pennant stores each user's resolved value (database or array driver) so a user's experience is sticky rather than re-rolled per request. Check it with `Feature::active('new-checkout')`, `Feature::for($team)->active(...)` for non-user scopes, or the `@feature` Blade directive; activate/deactivate globally with `Feature::activateForEveryone()`. The rollout playbook: internal users → percentage ramp → full activation → remove the flag and `Feature::purge()` the stored values. Trade-offs worth stating: flags are read on the hot path, so use the database driver with care under load (Pennant memoizes per request and supports eager loading values for many scopes to avoid N+1 flag lookups); and stale flags are tech debt — schedule their removal. Rich rule engines or cross-service flag platforms (LaunchDarkly et al.) are where Pennant stops being the right tool.

## Hard (staff)

### Q23: How do retries, backoff, and failed jobs work in Laravel queues?

**Answer:** A job's `$tries` controls max attempts; `$timeout` is the worker's per-job max seconds; `$backoff` is the delay between attempts, either a single int or an array `[10, 30, 60]` applied per attempt. After exhausting `$tries` the job fails and (with a `failed_jobs` table from `queue:failed-table`) is recorded with its payload and exception; the job's `failed(Throwable $e)` hook runs for cleanup. You can override per-attempt timing with a `backoff()` method, cap the total time window with `retryUntil()`, and limit attempts by exception count with `$maxExceptions`. On the worker side `--tries` and `--backoff` override the job's properties. Use `queue:retry <id>` to re-run a failed job, and configure the failed-job driver (database, etc.) in `config/queue.php`.

### Q24: What does Laravel Horizon add, and how do you operate it in production?

**Answer:** Horizon is a Redis-only queue manager that provides a dashboard, master/supervisor worker orchestration, and metrics for throughput, wait times, and recent jobs. You configure supervisors in `config/horizon.php` with queue lists, max workers, balance strategy (`simple`, `auto`, `false`), and per-supervisor `tries`/`timeout`. In production you run Horizon under Supervisor or a process manager — it spawns the workers itself — and on deploy you call `php artisan horizon:terminate` so Horizon gracefully finishes in-flight jobs and restarts with the new code. The `auto` balance mode uses past wait times to dynamically shift workers from idle queues to busy ones, which beats static worker allocation for bursty workloads.

### Q25: Explain Laravel broadcasting: channels, drivers, and the auth flow.

**Answer:** Broadcasting pushes server-side events to clients over WebSockets. The driver is configured in `config/broadcasting.php` — `pusher`, `ably`, `redis`, `log`, `null`, or `reverb` (the first-party server introduced in 11.x). Channels come in three flavors: public (anyone), private (require authorization), and presence (require authorization and expose members). An event implementing `ShouldBroadcast` declares `broadcastOn()` returning `Channel`/`PrivateChannel`/`PresenceChannel` instances and `broadcastWith()` for the payload. Private/presence channel access is authorized via a callback in `routes/channels.php` returning `true` (or a user array for presence) — the `/broadcasting/auth` endpoint issues a signed response. SPAs typically use Sanctum's stateful API middleware so the session cookie authorizes channel access, and the frontend subscribes via Laravel Echo.

### Q26: Why do `withoutOverlapping()` and `onOneServer()` matter for scheduled tasks?

**Answer:** In a multi-server deployment, every server runs `schedule:run` every minute by default, so a job runs N times instead of once. `onOneServer()` uses an atomic cache lock to ensure only one server runs the job per minute; it requires a shared cache store (Redis, database, memcached) so the lock is visible across nodes. `withoutOverlapping($minutes)` uses a similar atomic lock keyed on the job so a long-running task does not start a second instance while the first is still going; it is essential for jobs that mutate a single resource or could collide with themselves. Combine both for safe distributed scheduling: `onOneServer()->withoutOverlapping(30)`. Without `onOneServer` you get duplicate work; without `withoutOverlapping` you get overlapping runs on the same server if a job takes longer than its interval.

### Q27: Compare Gates and Policies. When do you use each?

**Answer:** Gates are closure-based authorization checks, good for cross-cutting abilities unrelated to a model ("admin-panel", "view-dashboard"). Policies are classes that group authorization logic for a specific model (`PostPolicy` with `view`, `create`, `update`, `delete` methods), auto-discovered from `app/Policies` and bound to the model via convention or `Gate::policy()`. Both can have `before` callbacks (a global short-circuit), return `true`/`false`/`null` (null falls through to other checks), and are invoked via `$user->can(...)`, `$this->authorize(...)`, `@can` in Blade, or `$this->authorizeResource(Post::class)` in controllers. Use a policy whenever the authorization is about a specific Eloquent model and a gate when it's a global permission. Both ultimately route through the `Gate` facade, so they share the same evaluation pipeline.

### Q28: Should APIs be CSRF-protected? How does Laravel handle this?

**Answer:** Stateless APIs consumed by external clients with tokens should not require CSRF — the `api` middleware group omits `VerifyCsrfToken`, and clients authenticate with Bearer tokens (Sanctum/Passport) which are not vulnerable to CSRF the way cookies are. SPAs served from the same domain as the API *are* cookie-authenticated and therefore vulnerable to CSRF, which is why Sanctum's stateful API middleware (`EnsureFrontendRequestsAreStateful`) re-enables session + CSRF for same-origin requests — the SPA gets an `XSRF-TOKEN` cookie and Laravel's Axios sends it back as `X-XSRF-TOKEN`. So the rule is: cookie-based = CSRF required; token-based = CSRF not required (and not in the `api` group by default). If you must exempt a webhook route from CSRF, do so explicitly and narrowly via the middleware's `$except` array.

### Q29: Compare Sanctum and Passport. Which would you choose and when?

**Answer:** Sanctum is a lightweight package providing two flows in one: SPA tokens (cookie session + CSRF, no tokens stored client-side) for first-party SPAs, and per-user API tokens (hashed at rest in `personal_access_tokens`, scoped with abilities) for mobile apps or simple third-party clients. Passport is a full OAuth2 server (authorization code with PKCE, client credentials, refresh tokens, scopes) built on `league/oauth2-server`, exposing `/oauth/authorize` and `/oauth/token`. Choose Sanctum for first-party APIs and simple token issuance; choose Passport when you must issue OAuth2 tokens to third parties (e.g., you are an OAuth2 provider for partners). Sanctum's API tokens are simpler to revoke and reason about than OAuth2 flows, so don't reach for Passport unless you genuinely need OAuth2.

### Q30: How do config and route caching improve performance, and what are the gotchas?

**Answer:** `php artisan optimize` runs `config:cache`, `route:cache`, `event:cache`, and `view:cache` to prebuild serialized lookup tables. `config:cache` serializes all config arrays and crucially **makes `env()` return null outside config files** because the environment is no longer reloaded — so never call `env()` in app code, only in `config/*.php`. `route:cache` serializes the route tree and only works if all routes use controllers (closure routes are not cached and silently fail to cache). Both must be rebuilt on every deploy; in CI run `optimize` after `composer install --no-dev`. Gotchas: cached config will use stale `APP_KEY`/credentials if you change `.env` without re-running `config:cache`, and cached routes will use old middleware aliases if you add new ones without re-running `route:cache`. Always run `config:clear`/`route:clear` if you suspect stale cache.

### Q31: What is a terminable middleware and how does it relate to `App::terminating()`?

**Answer:** A terminable middleware defines a `terminate(Request $request, Response $response)` method that runs after the response is sent to the client but before the worker process moves on — useful for analytics, audit logs, and any work that should not delay the user. `App::terminating(fn () => ...)` registers the same lifecycle callback at the container level; both run during `$kernel->terminate()`. Under `queue:work` long-running workers, terminate hooks still run per request. The gotcha: anything in `terminate` shares the booted app state but the request is already gone, so use `$request`/`$response` arguments rather than global state. This is the layer Horizon, Telescope, and many logging integrations hook into.

### Q32: How do nested database transactions and `DB::afterCommit()` work?

**Answer:** Laravel's transaction layer maps nested `DB::transaction()` calls onto database savepoints: a nested call creates a savepoint, and an exception rolls back only to the nearest savepoint while leaving the outer transaction intact. The closure form of `DB::transaction(fn () => ..., $attempts)` also retries on deadlock up to `$attempts` times. `DB::afterCommit(fn () => dispatch(...))` defers a callback until the outermost transaction commits — critical for jobs, listeners, and notifications that should not fire if the transaction rolls back (e.g., "send order confirmation" should only email when the order is actually saved). Without `afterCommit`, a queued listener dispatched mid-transaction could run before the row is visible, leading to "model not found" failures.

### Q33: You move an app to Octane and weird bugs appear in production. What are the likely causes?

**Answer:** Octane keeps the framework booted in long-lived workers (FrankenPHP/Swoole/RoadRunner), so anything assuming "fresh process per request" breaks. The usual suspects: **singletons holding request state** — a singleton that captured the authenticated user, the `Request`, or request-derived config leaks it into subsequent requests served by the same worker (fix: `scoped()` bindings, resolve at call time, or reset in an Octane `RequestReceived` listener); **constructor-injected `Request`/container in singletons** — the reference is stale after the first request; **static properties and memoized caches** growing unbounded, causing cross-request bleed and memory creep (mitigate with `--max-requests` worker recycling); and **runtime `config()` mutations** persisting across requests. Operationally, deploys must run `octane:reload` or workers keep executing old code. The staff-level framing: Octane trades per-request isolation for throughput, so it demands the same statelessness discipline as any long-lived app server — and if the team can't audit for that, php-fpm is the safer default.

### Q34: Design the queue layer for a system processing ~10k jobs/minute with mixed latency requirements.

**Answer:** Start by segmenting jobs into latency classes and giving each its own queue: `notifications` (user-facing, sub-second wait target), `default`, and `bulk` (reports, syncs). Centralize the mapping with `Queue::route()` so topology is reviewable. Run Redis as the backing store with **Horizon** managing supervisors — one supervisor per latency class with `balance: 'auto'` and `minProcesses`/`maxProcesses` bounds, so workers shift toward busy queues; alert on *wait time* per queue (Horizon metrics/Pulse), not depth. Make every job idempotent (at-least-once delivery is the contract) and guard single-resource mutations with `WithoutOverlapping` or `ShouldBeUnique`; protect flaky downstreams with `RateLimited` and `ThrottlesExceptions` job middleware plus bounded `backoff`. Ensure `retry_after` on the connection exceeds the longest job `timeout` to prevent double-processing, dispatch jobs with `afterCommit` so they never race the transaction, and recycle workers (`--max-jobs`/`--max-time`, `horizon:terminate` on deploy). At the point Redis on one node becomes the bottleneck or you need cross-region durability, move the heavy queues to SQS — the job API is driver-agnostic, so the migration cost is mostly operational.

### Q35: What has to happen on a zero-surprise Laravel production deploy, and why?

**Answer:** In order: `composer install --no-dev --optimize-autoloader`; run migrations (expand/contract pattern for zero-downtime — add columns first, deploy code, drop later); then rebuild the cached layers with `php artisan optimize` (`config:cache`, `route:cache`, `event:cache`, `view:cache`) because all of them serialize state that is stale the moment code changes — and cached config means `env()` returns null outside `config/`, so a deploy is where that bug surfaces. Then restart every long-lived PHP process, because none of them see new code by themselves: `queue:restart` (workers finish the current job then exit; supervisor restarts them), `horizon:terminate` if using Horizon, and `octane:reload` under Octane. If you skip these, old code keeps processing new jobs — the classic "deployed the fix but the bug continues" incident. Finish with the `/up` health check before shifting traffic. Bonus points for mentioning `php artisan down --secret` maintenance mode for breaking migrations, and that `schedule:run`'s cron needs no restart since it's a fresh process each minute.
