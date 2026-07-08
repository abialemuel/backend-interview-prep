# Testing and Security

## Testing

Laravel ships with PHPUnit support. As of Laravel 11 the default skeleton uses **Pest** optionally but PHPUnit remains fully supported. The `Tests\TestCase` is still the base; Pest provides a thin function layer over it.

### TestCase and feature vs unit tests

- `tests/Feature/` — end-to-end-ish tests that boot the framework, hit routes, and use the database.
- `tests/Unit/` — narrow tests that do not boot the framework (no facades, no DB) unless you change the base class.

The default base class for Feature tests extends `Illuminate\Foundation\Testing\TestCase` and gives access to the booted application, `RefreshDatabase`, `actingAs`, and the test HTTP client.

```php
namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostApiTest extends TestCase
{
    use RefreshDatabase;

    public function test_can_list_posts(): void
    {
        Post::factory()->count(3)->create(['user_id' => $this->user->id]);

        $response = $this->actingAs($this->user)->getJson('/api/posts');

        $response->assertStatus(200)
            ->assertJsonCount(3, 'data')
            ->assertJsonStructure(['data' => [['id', 'title', 'body']]]);
    }
}
```

### HTTP tests

```php
$this->get('/posts')->assertOk();
$this->get('/posts')->assertStatus(200);

$this->postJson('/api/posts', ['title' => 'x'])
    ->assertCreated()
    ->assertJson(['title' => 'x'])
    ->assertJsonPath('data.title', 'x')
    ->assertJsonStructure(['data' => ['id', 'title']])
    ->assertJsonCount(10, 'data')
    ->assertJsonMissing(['error' => true])
    ->assertJsonValidationErrors(['email'])
    ->assertRedirect(route('posts.index'));

$this->withHeaders(['X-Custom' => '1'])->get('/...');
$this->withSession(['foo' => 'bar'])->get('/...');
$this->actingAs($user)->get('/dashboard');
```

`getJson`/`postJson` set `Accept: application/json` and decode the response body into the assertion helpers.

### Database testing

```php
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\DatabaseTransactions;
use Illuminate\Foundation\Testing\DatabaseMigrations;

class XTest extends TestCase
{
    use RefreshDatabase;        // rolls back by wrapping each test in a transaction;
                                // also runs migrations if needed
    // or
    use DatabaseTransactions;   // only wraps the test in a transaction, assumes migrations done
}
```

- **`RefreshDatabase`** wraps the test in a transaction and ensures migrations run (only once per run, skipped if already migrated). Use for tests that mutate schema or rely on a clean DB.
- **`DatabaseTransactions`** just wraps the test in a transaction; faster but assumes the schema is set up. Use when migrations are expensive and tests only insert/update rows.
- **`DatabaseMigrations`** runs migrations up and down per test (slow; rarely needed unless you test migrations themselves).
- **`DatabaseTruncation`** (Laravel 11+) truncates tables between tests for cases where transactions don't work (e.g., concurrent connections or DDL).

Factories fill the database:

```php
$posts = Post::factory()->count(3)->published()->create();

$user = User::factory()->create();
Post::factory()->for($user)->count(2)->create();

$this->assertDatabaseHas('posts', ['title' => 'x', 'user_id' => $user->id]);
$this->assertDatabaseMissing('posts', ['title' => 'y']);
$this->assertSoftDeleted('posts', ['id' => $post->id]);
$this->assertModelExists($post);
$this->assertModelMissing($post->fresh());
```

### Mocking and fakes

Fakes are preferred over mocking because they exercise the real integration paths without side effects.

```php
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Storage;

// Jobs
Bus::fake();
Bus::dispatch(new GenerateReport($user));
Bus::assertDispatched(GenerateReport::class, fn ($job) => $job->user->id === $user->id);
Bus::assertNotDispatched(SendNewsletter::class);
Bus::assertBatched(fn (PendingBatch $b) => $b->jobs()->count() === 3);
Bus::assertChained([JobA::class, JobB::class]);

// Queues (asserts against dispatched jobs even if they were fake-dispatched)
Queue::fake();
Queue::assertPushed(GenerateReport::class);
Queue::assertPushedOn('invoices', GenerateReport::class);

// Events
Event::fake([OrderShipped::class]);  // fake only these; others still fire
OrderShipped::dispatch($order);
Event::assertDispatched(OrderShipped::class, fn ($e) => $e->order->id === $order->id);
Event::assertListening(OrderShipped::class, SendShippingNotification::class);

// Mailables
Mail::fake();
Mail::to($user)->send(new OrderShipped($order));
Mail::assertSent(OrderShipped::class, fn ($m) => $m->hasTo($user->email));
Mail::assertQueued(OrderShipped::class);

// Notifications
Notification::fake();
$user->notify(new InvoicePaid($invoice));
Notification::sent($users, InvoicePaid::class);
Notification::assertSentTo($user, InvoicePaid::class);

// HTTP
Http::fake(['api.stripe.com/*' => Http::response(['ok' => true], 200)]);
Http::preventStrayRequests();
Http::assertSent(fn (Request $r) => $r->hasHeader('Authorization'));

// Filesystem
Storage::fake('local');
$file = UploadedFile::fake()->image('avatar.jpg', 200, 200);
Storage::disk('local')->assertExists('avatars/' . $file->hashName());

// Time
$this->travel(5)->minutes();
$this->travelTo(now()->addDay());
$this->travelBack();
```

Mockery is used for isolating specific services; Laravel auto-resolves Mockery mocks via `Mockery::mock(...)->shouldReceive(...)` or `$this->mock(Service::class)` from the test case:

```php
$this->mock(PaymentGateway::class, fn ($m) =>
    $m->shouldReceive('charge')->once()->andReturn(true)
);

$this->mock(\App\Services\PaymentGateway::class)->shouldReceive('charge')->with(1000)->andReturn(true);
```

`$this->instance(Service::class, $mock)` and `$this->bind(Service::class, fn () => $mock)` also work — they bind in the container.

### Testing jobs and queues

```php
use Illuminate\Support\Facades\Queue;
use App\Jobs\GenerateReport;

public function test_job_enqueued(): void
{
    Queue::fake();
    $this->post('/reports', ['user' => $user->id]);
    Queue::assertPushed(GenerateReport::class, fn ($j) => $j->user->id === $user->id);
}

public function test_job_executes(): void
{
    (new GenerateReport($user))->handle();
    Storage::disk('s3')->assertExists("reports/{$user->id}.pdf");
}

public function test_job_serializes_models(): void
{
    $job = new GenerateReport($user);
    $restored = unserialize(serialize($job));
    $this->assertSame($user->id, $restored->user->id);
}
```

### Dusk

Dusk runs a ChromeDriver-controlled browser for end-to-end tests including JS:

```php
class LoginTest extends DuskTestCase
{
    public function test_login(): void
    {
        $this->browse(function (Browser $b) {
            $b->visit('/login')
              ->type('email', 'a@b.com')
              ->type('password', 'secret')
              ->press('Login')
              ->assertPathIs('/dashboard')
              ->assertSee('Welcome');
        });
    }
}
```

Dusk tests don't share the booted app instance with unit/feature tests; they hit a real server via `php artisan dusk`.

### Snapshots

Laravel doesn't ship snapshot assertions natively; use `spatie/phpunit-snapshot-assertions` (composer require --dev). Snapshots freeze expected output and fail when it drifts:

```php
use Spatie\Snapshots\MatchesSnapshots;

class ExportTest extends TestCase
{
    use MatchesSnapshots;

    public function test_export_json(): void
    {
        $json = Exporter::for($user)->toJson();
        $this->assertMatchesSnapshot($json);
    }
}
```

First run records the snapshot; subsequent runs compare. Update with `--update-snapshots`.

## Security

### Authentication

Laravel's auth is built from three pieces:

- **Guards** — how users are authenticated (session, token, passport, sanctum).
- **Providers** — how users are retrieved (`eloquent` against a model, `database` against raw rows).
- **The `Auth` facade / `auth()` helper** — exposes the manager.

```php
if (Auth::attempt(['email' => $email, 'password' => $password])) {
    $request->session()->regenerate();
    return redirect()->intended('dashboard');
}

Auth::logout();
$request->session()->invalidate();
$request->session()->regenerateToken();

Auth::guard('api')->user();
```

- **Session guard** (`web`): cookie-session based, used for browser apps and SPAs.
- **Token guard**: simple API token in a header; superseded by Sanctum for new code.
- **Sanctum guard**: provides both SPA (cookie) and API token (Bearer) flows.
- **Passport guard**: OAuth2 access token validation.

Guards and providers are configured in `config/auth.php`. In 11+ the file is intentionally small; password resets and notifications live there too.

```php
'guards' => [
    'web' => ['driver' => 'session', 'provider' => 'users'],
    'sanctum' => ['driver' => 'sanctum', 'provider' => null],
],

'providers' => [
    'users' => ['driver' => 'eloquent', 'model' => App\Models\User::class],
],
```

### Authorization: Gates and Policies

Gates are closures; policies are classes scoped to a model. Both return `bool`-ish (`true`, `false`, `null`, or throw).

```php
// AppServiceProvider::boot or AuthServiceProvider (Laravel 10)
Gate::define('admin-panel', fn (User $u) => $u->is_admin);

Gate::before(fn (User $u, string $ability) => $u->isSuperAdmin() ? true : null);
Gate::after(fn (User $u, string $ability, bool $result, array $args) => /* ... */);
```

Policies:

```php
class PostPolicy
{
    public function viewAny(User $user): bool { /* ... */ }
    public function update(User $user, Post $post): bool
    {
        return $post->user_id === $user->id;
    }
}
```

Register policies in `app/Policies` (auto-discovered) or explicitly via `Gate::policy(Post::class, PostPolicy::class)`. Use them:

```php
if ($user->cannot('update', $post)) abort(403);
$this->authorize('update', $post);
request()->user()->can('update', $post);
Gate::authorize('update', $post);

// In a controller:
public function update(Post $post) {
    $this->authorize('update', $post);   // throws 403
    // ...
}

// In Blade:
@can('update', $post) <a>Edit</a> @endcan
@canany(['update', 'delete'], $post) ... @endcanany
@cannot('update', $post) ... @endcannot
```

Resource authorization in controllers:

```php
public function __construct()
{
    $this->authorizeResource(Post::class, 'post');
}
```

This wires `viewAny`, `view`, `create`, `update`, `delete` to policy methods on each resource action.

### CSRF protection

`VerifyCsrfToken` (in the `web` middleware group) rejects POST/PUT/DELETE without a valid `_token` matching the session. For APIs, the `api` group does not include CSRF middleware because API clients are typically stateless tokens.

Exempt specific routes from CSRF (rarely needed; use sparingly):

```php
// bootstrap/app.php withMiddleware or in a custom middleware
protected $except = ['webhook/stripe'];
```

For SPAs served from the same domain, use Sanctum's stateful API which sets an `XSRF-TOKEN` cookie. The frontend reads it from JS (`document.cookie`) and sends `X-XSRF-TOKEN` (Laravel's Axios does this automatically).

For programmatic clients, expose an endpoint that returns the token and have the client send `X-CSRF-TOKEN: <token>`.

### Encryption

`Illuminate\Encryption\EncryptionServiceProvider` binds a `Illuminate\Contract(encryption\Encrypter)` backed by `OpenSSL` AES-256-CBC. `APP_KEY` is the encryption key; never expose it.

```php
use Illuminate\Support\Facades\Crypt;

$encrypted = Crypt::encryptString('secret');
$plain = Crypt::decryptString($encrypted);

$encrypted = Crypt::encrypt(['user' => $user->id, 'exp' => time() + 300]);
```

Encrypted casts in Eloquent: `'ssn' => 'encrypted'`, `'settings' => AsEncryptedArrayObject::class`. Use these for sensitive columns at rest. If you rotate `APP_KEY`, you must first decrypt and re-encrypt existing data.

### Hashing

`Hash::make($password)` salts and hashes; salts are embedded. Default driver from `config/hashing.php`:

```php
'driver' => 'bcrypt',   // or 'argon', 'argon2id'

Hash::make('secret');
Hash::check('secret', $user->password);   // bool
Hash::needsRehash($user->password);        // true if needs upgrading (e.g., cost change)
```

- **bcrypt** — default, configurable rounds.
- **argon2(i)** — memory-hard; requires libsodium.
- Never store plain hashes of passwords; never roll your own.

### Mass assignment

`$fillable` whitelist or `$guarded` blacklist prevents unintended column writes during `::create`/`->fill`/`->update`. With `$guarded = []` you accept the risk; with `Model::preventLazyLoading` analog there is no mass assignment guard "strict" flag, but Laravel 12 provides `Model::preventSilentlyDiscardingAttributes()` (added in 10.x as `Model::preventSilentlyDiscardingAttributes` strict mode) that throws if you assign an attribute not in `$fillable` and not a real column.

```php
// AppServiceProvider::boot
Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
```

### SQL injection prevention

- The query builder and Eloquent always bind parameters via PDO prepared statements: `->where('email', $email)` is safe.
- Raw methods `whereRaw`, `selectRaw`, `DB::raw`, `DB::statement`, `->orderByRaw` do **not** bind. Only interpolate values you trust; for user input pass them as bindings.

```php
// Safe:
Post::where('title', $input)->get();
DB::select('SELECT * FROM posts WHERE title = ?', [$input]);

// Dangerous: do NOT do this
Post::whereRaw("title = '{$input}'")->get();

// Safe raw:
Post::whereRaw('title ilike ?', ['%' . $input . '%'])->get();
```

### XSS in Blade

Blade `{{ $value }}` runs through `htmlspecialchars` and is XSS-safe for HTML contexts. `{!! $value !!}` outputs raw; only use it for trusted HTML you generated. For user-provided rich text, sanitize with a library like `mews/purifier`.

```blade
{{ $post->title }}             {{-- escaped --}}
{!! $post->body_html !!}        {{-- raw, only if trusted --}}
```

Note `{{ }}` does not prevent XSS in attribute contexts (e.g., `href="javascript:..."`), so still validate inputs.

### File upload validation/storage

```php
$request->validate([
    'avatar' => ['required', 'image', 'mimes:jpg,png,webp', 'max:2048'],
]);

$path = $request->file('avatar')->store('avatars', 's3');
// stores under a generated filename; use storeAs() for a custom name
$size = $request->file('avatar')->getSize();
$mime = $request->file('avatar')->getMimeType();
```

For tests use `UploadedFile::fake()->image(...)` and `Storage::fake('s3')`. Validate mime and size on the server; never trust the user-supplied filename for storage location.

### Rate limiting

The `throttle` middleware uses named rate limiters from `RateLimiter`. In 11+ define limiters via `Limit::perMinute(...)` callbacks:

```php
// AppServiceProvider::boot or routes
RateLimiter::for('api', fn (Request $r) => Limit::perMinute(60)
    ->by($r->user()?->id ?: $r->ip())
    ->response(fn () => response('Too many requests', 429));

RateLimiter::for('uploads', fn (Request $r) => Limit::perMinute(5)->by($r->user()->id));

Route::middleware('throttle:api')->group(...);
Route::middleware('throttle:10,1')->post('/contact', ...);   // 10 per minute
Route::middleware('throttle:uploads')->post('/upload', ...);
```

Custom limiter keyed on something other than IP lets you apply per-user or per-tenant limits. `RateLimiter::attempt` runs code with a limit:

```php
$ok = RateLimiter::attempt('send-sms:'.$user->id, $maxAttempts = 5, function () use ($user) {
    SmsService::to($user->phone)->send();
}, decaySeconds: 60);
```

### Sanctum vs Passport

**Sanctum** (simple): two flows in one package.

1. **SPA tokens** — for first-party SPAs on the same domain: cookie session + CSRF, no tokens stored client-side.
2. **API tokens** — for mobile apps or third-party clients: issue per-token tokens stored in `personal_access_tokens`, scoped with abilities (`$user->createToken('mobile', ['read-orders'])->plainTextToken`). Tokens are SHA-256 hashed at rest.

```php
$token = $user->createToken('mobile', ['orders:read']);
$plain = $token->plainTextToken;

// middleware
Route::middleware('auth:sanctum')->group(...);

// authorization
if ($user->tokenCan('orders:read')) { /* ... */ }
```

**Passport** (OAuth2): full OAuth2 server — authorization code grant (with PKCE), client credentials grant, password grant (deprecated), personal access tokens, scopes. Use when you must issue OAuth2 tokens to third parties; use Sanctum when you just need first-party API access.

Passport relies on the `league/oauth2-server` package and exposes `/oauth/authorize`, `/oauth/token`, etc.

### Password reset flows

Laravel ships `ForgotPassword` and `ResetPassword` notifications and the `Illuminate\Auth\Notifications\ResetPassword` link. The flow:

1. User submits email to `/forgot-password` → broker creates a signed token in `password_resets`/`password_reset_tokens`.
2. Email links to `/reset-password/{token}` with the email.
3. User submits new password; broker validates token and hashes the new password.

```php
use Illuminate\Support\Facades\Password;

$status = Password::sendResetLink(['email' => $request->email]);
$status = Password::reset($request->only('email', 'token', 'password', 'password_confirmation'), function (User $user, string $password) {
    $user->forceFill(['password' => Hash::make($password)])->setRememberToken(Str::random(60));
    $user->save();
    event(new PasswordReset($user));
});
```

Customize the broker, notification, validation rules, and redirect routes via `config/auth.php` `passwords` array and overriding notifications. Tokens expire after `expire` minutes (default 60) and are single-use.

### Other essentials

- `signed` route middleware (`URL::signedRoute`) prevents tampering with links such as unsubscribe or one-click.
- `throttle` on login routes mitigates brute force; combine with `Password::broker()`.
- `VerifyEmail` middleware (`verified`) gates routes behind email confirmation.
- `2FA` is not first-party; use `laravel/fortify` or third-party.
- `password.confirm` middleware forces re-entry of password before sensitive actions.
- `Pest` plugins like `pest-plugin-laravel` add `actingAs`, `withoutExceptionHandling`, etc.
