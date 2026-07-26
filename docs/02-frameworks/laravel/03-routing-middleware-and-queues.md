# Routing, Middleware, and Queues

## Routing

Routes are registered in `routes/web.php`, `routes/api.php`, and (in Laravel 11+) `routes/console.php` for closure-based commands. The router is loaded by `bootstrap/app.php`'s `withRouting` call, which also wires middleware groups (`web`, `api`) and the optional health check route.

```php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;

Route::get('/posts', [PostController::class, 'index'])->name('posts.index');

Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');

Route::post('/posts', [PostController::class, 'store'])
    ->middleware(['auth', 'throttle:10,1']);

Route::put('/posts/{post}', [PostController::class, 'update'])->name('posts.update');

Route::delete('/posts/{post}', [PostController::class, 'destroy'])->name('posts.destroy');
```

### Parameters and constraints

```php
Route::get('/users/{id}', fn (int $id) => $id)
    ->whereNumber('id');

Route::get('/posts/{slug}', fn (string $slug) => $slug)
    ->where('slug', '[a-z0-9-]+');

Route::get('/users/{id}/{name?}', fn (int $id, ?string $name = null) => $id);

Route::pattern('id', '[0-9]+');   // global constraint in RouteServiceProvider/bootstrap
```

`whereNumber`, `whereAlpha`, `whereAlphaNumeric`, `whereIn('status', ['a','b'])` are conveniences.

### Named routes and URL generation

```php
route('posts.show', $post);          // /posts/42
route('posts.show', ['post' => $post, 'tab' => 'comments']);
redirect()->route('posts.index');
URL::signedRoute('unsubscribe', ['user' => $user]);
```

Signed routes produce a tamper-proof URL with an HMAC signature.

### Route groups

```php
Route::middleware(['auth', 'verified'])->prefix('admin')->name('admin.')->group(function () {
    Route::get('/dashboard', AdminDashboard::class)->name('dashboard');
    Route::resource('posts', PostController::class);
});

Route::controller(PostController::class)->group(function () {
    Route::get('/posts', 'index');
    Route::post('/posts', 'store');
});

Route::name('admin.')->prefix('admin')->controller(AdminPostController::class)->group(...);
```

### Resource controllers

`Route::resource('posts', PostController::class)` registers `index`, `create`, `store`, `show`, `edit`, `update`, `destroy`. Use `->only([...])`, `->except([...])`, `->shallow()` for shallow nesting, `->names(...)` to rename, and `->parameter('posts', 'post')` to override the parameter name.

```php
Route::resource('users.posts', PostController::class)->shallow();
// GET /users/{user}/posts, GET /posts/{post}, ...
```

`apiResource` skips `create` and `edit`:

```php
Route::apiResource('posts', PostController::class)->only(['index', 'show']);
Route::apiResources([
    'posts' => PostController::class,
    'tags'  => TagController::class,
]);
```

### Route model binding

Implicit binding by type-hinted variable name:

```php
Route::get('/posts/{post}', function (Post $post) {
    return $post;
});
```

Custom key/column:

```php
Route::get('/posts/{post:slug}', function (Post $post) {
    return $post;
});
```

Custom key resolution per-model:

```php
public function getRouteKeyName(): string
{
    return 'slug';
}
```

Explicit binding (formerly `Route::bind` in `RouteServiceProvider`, now anywhere routes are loaded):

```php
Route::bind('post', fn (string $value) => Post::where('slug', $value)->firstOrFail());
```

Enum binding:

```php
enum Category: string { case News = 'news'; case Tech = 'tech'; }

Route::get('/posts/{category}', function (Category $category) {
    return $category->value;
});
```

Default scopes / query customization:

```php
Route::get('/posts/{post}', function (Post $post) { return $post; })
    ->scopeBindings();   // applies the model's global scopes during binding

// Or:
Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) { return $post; })
    ->scopeBindings();
```

### Route caching

```bash
php artisan route:cache
php artisan route:clear
```

Route caching serializes the route tree. It only works when all routes use controllers (closure routes are not cached) and `app()->routesAreCached()` short-circuits route loading. Always run `optimize` in production deployments.

## Middleware

### Before, after, and terminable

```php
class AuditMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        Log::info('request.in', ['path' => $request->path()]);
        $response = $next($request);
        $response->headers->set('X-Audit-Id', Str::uuid()->toString());
        return $response;
    }

    public function terminate(Request $request, Response $response): void
    {
        Log::info('request.out', ['status' => $response->status()]);
    }
}
```

`terminate` runs after the response is sent and is ideal for non-blocking analytics. Note: under `queue:work` long-running workers, terminate runs per request.

### Middleware parameters

```php
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('role:editor,admin');
```

```php
class EnsureRole
{
    public function handle(Request $request, Closure $next, string ...$roles)
    {
        abort_unless(
            $request->user() && $request->user()->hasAnyRole($roles),
            403
        );
        return $next($request);
    }
}
```

### Middleware priority and aliases

Configure in `bootstrap/app.php` `withMiddleware`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'role' => \App\Http\Middleware\EnsureRole::class,
        'admin' => \App\Http\Middleware\IsAdmin::class,
    ]);

    $middleware->priority([
        \App\Http\Middleware\EnsureUserIsActive::class,
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\EnsureRole::class,
    ]);
})
```

## Controllers and dependency injection

Single-action controllers use `__invoke`:

```php
class ShowDashboard
{
    public function __invoke(Request $request)
    {
        return view('dashboard');
    }
}

Route::get('/dashboard', ShowDashboard::class);
```

Method injection works on any controller method; the `Request` parameter can be in any position:

```php
class PostController
{
    public function __construct(protected PostRepository $posts) {}

    public function show(Request $request, Post $post, CommentService $comments)
    {
        return view('posts.show', [
            'post' => $post,
            'comments' => $comments->forPost($post),
        ]);
    }
}
```

Constructor injection requires the controller be resolved via the container (always true for routed controllers) but the cached instance is reused across requests only under Octane — otherwise each request builds a new controller.

## Form requests

A form request centralizes validation and authorization, removing boilerplate from the controller.

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('create', Post::class);
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body' => ['required', 'string'],
            'published_at' => ['nullable', 'date', 'after_or_equal:today'],
            'tags' => ['array'],
            'tags.*' => ['string', 'distinct'],
        ];
    }

    public function messages(): array
    {
        return ['title.required' => 'A title is mandatory.'];
    }

    public function attributes(): array
    {
        return ['published_at' => 'publish date'];
    }

    protected function prepareForValidation(): void
    {
        $this->merge(['slug' => Str::slug($this->title)]);
    }
}
```

`prepareForValidation` mutates input before rules run; `passedValidation` mutates after. Injected into the controller method, the request's `validated()` returns the validated array.

### Validation

```php
$request->validate(['email' => 'required|email|unique:users,email']);

Validator::make($data, [...])->validate();

$validator = Validator::make($data, [...]);
if ($validator->fails()) { /* ... */ }
```

Custom rule objects:

```php
use Illuminate\Contracts\Validation\ValidationRule;

class ValidStripeCoupon implements ValidationRule
{
    public function validate(string $attribute, mixed $value, \Closure $fail): void
    {
        if (! app(Stripe::class)->isValidCoupon($value)) {
            $fail("The {$attribute} is not a valid coupon.");
        }
    }
}

$request->validate(['coupon' => [new ValidStripeCoupon]]);
```

Older `Rule` interface with `passes`/`message` still works; the `ValidationRule` interface is preferred for stateless rules.

Closures work too: `'coupon' => ['string', fn ($attr, $value, $fail) => $value !== 'invalid' ?: $fail('bad coupon')]`.

Conditional rules use `Rule::when($condition, [...rules])`, `Rule::requiredIf(...)`, `Rule::in([...])`, `Rule::exists('users', 'id')`, `Rule::unique('users', 'email')->ignore($user)`.

## The Queue system

Queues let you defer slow work (sending email, generating reports, syncing to third-party APIs) to a background worker. Drivers include `sync` (in-process, default), `database`, `redis`, `beanstalkd`, `sqs`, and null.

### Jobs

```php
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

use Illuminate\Queue\Attributes\Backoff;
use Illuminate\Queue\Attributes\Timeout;
use Illuminate\Queue\Attributes\Tries;

#[Tries(5)]
#[Timeout(120)]
#[Backoff([10, 30, 60])]
class GenerateInvoicePdf implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $maxExceptions = 3;

    public function __construct(public Order $order)
    {
        $this->onQueue('invoices');
        $this->delay(now()->addMinutes(2));
    }

    public function handle(): void
    {
        $pdf = app(PdfService::class)->for($this->order);
        Storage::disk('s3')->put($pdf->path, $pdf->content);
    }

    public function middleware(): array
    {
        return [(new RateLimited('invoices'))->releaseAfter(60)];
    }

    public function failed(\Throwable $e): void
    {
        // runs when tries exhausted
    }
}
```

The `#[Tries]`, `#[Timeout]`, `#[Backoff]`, and `#[FailOnTimeout]` attributes are the Laravel 13 declarative style; public properties (`public $tries = 5;`) remain fully supported and are what you'll see in most existing codebases.

### Dispatching

```php
GenerateInvoicePdf::dispatch($order);

GenerateInvoicePdf::dispatch($order)
    ->onQueue('invoices')
    ->delay(now()->addMinutes(5))
    ->afterResponse();

dispatch(new GenerateInvoicePdf($order));

GenerateInvoicePdf::dispatchIf($order->should_invoice, $order);
GenerateInvoicePdf::dispatchUnless($order->skip_invoice, $order);
```

`dispatchSync` runs the job immediately in-process (the old `dispatchNow` is gone); `afterResponse()` runs it after the response is sent but before the worker moves on. For fire-and-forget work that doesn't justify a queued job at all, the `defer(fn () => ...)` helper (Laravel 11.23+) runs a closure after the response with no serialization or worker involved — but it dies with the process, so it offers no retry guarantees.

### Queue routing (Laravel 13)

Instead of scattering `->onQueue()`/`->onConnection()` across dispatch sites or constructors, Laravel 13 lets you declare routing rules centrally:

```php
// AppServiceProvider::boot()
Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');
Queue::route(GenerateInvoicePdf::class, queue: 'invoices');
```

This keeps queue topology (which jobs go where, which workers consume what) in one reviewable place — valuable once you have dedicated workers per queue.

### Job batching

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

$batch = Bus::batch([
    new ProcessReport($userA),
    new ProcessReport($userB),
    new ProcessReport($userC),
])->then(function (Batch $batch) {
    Log::info("Batch {$batch->id} done");
})->catch(function (Batch $batch, Throwable $e) {
    Log::error("Batch failed: {$e->getMessage()}");
})->finally(function (Batch $batch) {
    // runs whether or not the batch succeeded
})->onQueue('reports')->name('monthly-reports')->dispatch();
```

Track progress via the `job_batches` table (`queue:batches-table` migration). `$batch->progress()`, `->pendingJobs`, `->failedJobs`, and `->add([...])` to enqueue more jobs into an existing batch.

### Queued listeners and mailables

Listeners implementing `ShouldQueue` run on the queue; mailables use `Mail::to(...)->queue(new OrderShipped($order))` or `->later(now()->addMinutes(10), new OrderShipped(...))`.

### Workers

```bash
php artisan queue:work redis --queue=default,invoices --tries=3 --backoff=10,30 --timeout=120 --memory=256 --sleep=3 --max-time=3600
php artisan queue:listen
php artisan queue:restart
php artisan queue:retry all
```

- `queue:work` keeps the framework booted across jobs (fast; ideal for production with supervisor). Use `--max-time`/`--max-jobs` so the worker periodically restarts itself to free memory and pick up new code via `queue:restart`.
- `queue:listen` boots the framework per job (slower; useful in development where you want code changes picked up immediately).
- `queue:work` does not see code changes between jobs unless restarted; deploy with `queue:restart` and have supervisor restart the worker.
- `--tries` overrides `$tries`; failed jobs go to `failed_jobs` (`queue:failed-table`).
- `--backoff` controls retry delays; the job's `$backoff = [10, 30, 60]` array applies the Nth delay for the Nth attempt.
- `--timeout` is the worker's max seconds per job — set higher than the job's own internal timeout to avoid double-processing.
- `--max-jobs=N` exits after N jobs; pair with supervisor to recycle workers and release memory.
- `--stop-when-empty` finishes the current queue then exits.

### Retries, timeouts, backoff, rate limiting

Per-job:

```php
public $tries = 5;
public $timeout = 120;
public $backoff = 10;            // seconds, or [10, 30, 60] per attempt
public $maxExceptions = 3;

public function retryUntil(): DateTime
{
    return now()->addDay();
}

public function backoff(): array
{
    return [10, 30, 60];
}
```

Job middleware:

```php
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\Middleware\ThrottlesExceptions;

public function middleware(): array
{
    return [
        (new RateLimited('reports'))->releaseAfter(20),
        new WithoutOverlapping($this->order->id, releaseAfter: 60),
        (new ThrottlesExceptions(5, 5))->releaseAfterOneMinute(),
    ];
}
```

`WithoutOverlapping` uses an atomic lock keyed on the job's unique ID — critical for jobs that mutate a single resource.

### Failed jobs

```bash
php artisan queue:failed-table && php artisan migrate
php artisan queue:failed
php artisan queue:retry <id>
php artisan queue:retry all
php artisan queue:forget <id>
php artisan queue:flush
```

`failed_jobs` records the exception, payload, and queue. Jobs can also implement `failed(Throwable $e)` to run cleanup.

### Horizon

Horizon (Redis only) provides a dashboard, master/supervisor worker management, and metrics for throughput/wait times. Configure with a `config/horizon.php` supervisors block; deploy with `php artisan horizon:terminate` on deploys so Horizon gracefully restarts supervisors under supervisor/Docker.

```php
'production' => [
    'supervisor-1' => [
        'connection' => 'redis',
        'queue' => ['default', 'invoices'],
        'balance' => 'auto',
        'minProcesses' => 1,
        'maxProcesses' => 10,
        'balanceMaxShift' => 3,
        'balanceCooldown' => 3,
        'tries' => 3,
    ],
],
```

`balance: 'auto'` scales worker processes per queue based on current workload/wait times (bounded by `minProcesses`/`maxProcesses`, with `balanceMaxShift`/`balanceCooldown` controlling how aggressively); `'simple'` splits processes evenly; `false` treats the queue list as strict priority order.

### Scaling queues (common senior/staff question)

- **Isolate by latency class.** Put fast, user-facing jobs (emails, notifications) on their own queue with dedicated workers so a burst of slow report jobs can't starve them. `Queue::route()` centralizes this mapping.
- **Scale horizontally with more worker processes/nodes**, not bigger jobs. Workers are stateless; the bottleneck is usually Redis/DB contention or the downstream API, not PHP.
- **Make jobs idempotent.** At-least-once delivery is the contract: a worker crash after work-but-before-delete replays the job. Use unique keys, upserts, or `ShouldBeUnique`/`WithoutOverlapping` where duplication hurts.
- **Watch `retry_after` vs `timeout`.** `retry_after` (connection config) must exceed the longest job `timeout`, or a still-running job gets handed to a second worker — the classic double-processing bug.
- **Database driver caveat:** fine at low volume, but polling and row locking make it a contention hotspot; move to Redis (+ Horizon) or SQS as volume grows.
- **Backpressure:** monitor queue depth and wait time (Horizon metrics, Pulse) and alert on wait time, not depth alone.

## Events and listeners

Register listeners via auto-discovery (Laravel 11+ default), or `Event::listen`, or `Event::subscribe`.

```php
namespace App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderShipped
{
    use Dispatchable, SerializesModels;

    public function __construct(public Order $order) {}
}
```

```php
namespace App\Listeners;

use Illuminate\Contracts\Queue\ShouldQueue;

class SendShippingNotification implements ShouldQueue
{
    public function handle(OrderShipped $event): void
    {
        // runs on the queue; $event->order is re-fetched from its serialized id
    }

    public function shouldQueue(OrderShipped $event): bool
    {
        return $event->order->should_notify;
    }

    public $tries = 3;
    public $backoff = 10;
}
```

Dispatch with `OrderShipped::dispatch($order)`. Auto-discovery scans `app/Listeners` and `app/Events` (or configured paths) for type-hinted `handle` arguments.

```php
// AppServiceProvider::boot()
Event::listen(OrderShipped::class, [SendShippingNotification::class, 'handle']);

Event::listen('event.name', fn ($payload) => /* ... */);
```

## Broadcasting

Broadcasting pushes server events to browser clients over WebSockets. Drivers: `log`, `null`, `pusher`, `ably`, `redis`, and `reverb` — the first-party WebSocket server (introduced in the 11.x cycle, now the default choice for self-hosted broadcasting). Reverb speaks the Pusher protocol, so Echo works unchanged, and it scales horizontally by fanning out messages across nodes via Redis pub/sub.

Channels:

- **Public**: anyone may subscribe.
- **Private**: requires authorization via `Broadcast::auth` callback (default route `/broadcasting/auth`).
- **Presence**: like private, but exposes who is in the channel.

```php
// routes/channels.php
Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->ownsOrder($orderId);
}, guards: ['web', 'sanctum']);

Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
}, PresenceChannel::class);
```

Broadcast from events:

```php
class OrderShipped implements ShouldBroadcast
{
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('orders.' . $this->order->user_id),
            new \Illuminate\Broadcasting\Channel('orders'),
        ];
    }

    public function broadcastWith(): array
    {
        return ['id' => $this->order->id, 'total' => $this->order->total];
    }

    public function broadcastAs(): string
    {
        return 'order.shipped';
    }
}
```

Frontend subscribes via Laravel Echo. The `/broadcasting/auth` endpoint issues signed auth tokens for private/presence channels. Sanctum's stateful API middleware (`EnsureFrontendRequestsAreStateful`) is typically used so the SPA session authorizes channel access.

## Task scheduling

In Laravel 11+ the scheduler is configured in `routes/console.php` via the `Schedule` facade (or a closure passed to `->withSchedule()` in `bootstrap/app.php`). The system relies on a single cron entry:

```
* * * * * cd /app && php artisan schedule:run >> /dev/null 2>&1
```

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('reports:send')->dailyAt('06:00')->timezone('America/New_York');

Schedule::job(new ProcessReports)->hourly()->onOneServer();

Schedule::call(fn () => Cache::flush())->weekly()->sundays()->at('02:00');

Schedule::exec('node /app/jobs/cleanup.js')->everyFifteenMinutes();
```

`onOneServer()` requires a shared cache store (Redis/memcached/database) so a single server runs the job even if multiple servers run `schedule:run`. Without it, every server runs the same job.

`withoutOverlapping()` uses an atomic lock so a job that hasn't finished by the next run-time is skipped. Use `withoutOverlapping(10)` to set the lock TTL in minutes.

```php
Schedule::command('reports:generate')
    ->daily()
    ->onOneServer()
    ->withoutOverlapping(30)
    ->runInBackground()
    ->evenInMaintenanceMode();
```

Other helpers: `->cron('* */4 * * *')`, `->everyTenMinutes()`, `->between('08:00','18:00')`, `->when(fn () => ...)`, `->skip(fn () => ...)`, `->environments(['production'])`, `->name(...)`/`->description(...)`, `->appendOutputTo($path)`, `->emailOutputTo(...)`.

For distributed scheduling prefer `schedule:work` in development (runs the schedule loop in foreground); in production always use the cron entry plus, if you have multiple servers, `onOneServer()` with a shared cache driver.
