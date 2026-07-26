# Eloquent and Database

## Model basics

A model maps a class to a table, with each row hydrated into an instance. By convention the table name is the snake-cased plural of the class; override `$table` to change it.

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class Post extends Model
{
    use HasFactory;

    protected $table = 'blog_posts';

    protected $fillable = ['title', 'body', 'published_at'];

    protected $guarded = [];

    protected $hidden = ['internal_notes', 'metadata'];

    protected $visible = ['id', 'title', 'body', 'published_at'];

    protected function casts(): array
    {
        return [
            'published_at' => 'datetime',
            'metadata' => 'array',
            'is_featured' => 'boolean',
            'price' => 'decimal:2',
            'tags' => 'collection',
            'settings' => AsEncryptedArrayObject::class,
            'status' => PostStatus::class,
        ];
    }
}
```

The `casts()` *method* is the preferred style since Laravel 11 (it can call static methods like `AsCollection::using(...)`); the older `$casts` property still works.

### `$fillable` vs `$guarded`

- `$fillable` is a whitelist: only those attributes are mass-assignable.
- `$guarded` is a blacklist: everything is assignable except those listed.
- `$guarded = []` disables protection entirely (use only when you fully trust the input).
- Mass assignment protection exists because `$request->all()` could otherwise push unexpected columns into the model.

### `$casts` and custom/encrypted casts

Custom casts implement `CastsAttributes`:

```php
namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

class Money implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes): ?int
    {
        return isset($value) ? (int) $value : null;
    }

    public function set($model, string $key, $value, array $attributes): ?array
    {
        return isset($value) ? [$key => (int) round($value * 100)] : null;
    }
}
```

`AsEncryptedArrayObject` and `AsEncryptedCollection` ship built-in to transparently encrypt JSON columns at rest; `encrypted:` casts also exist for scalar values.

## Relationships

```php
class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    public function phone(): HasOne
    {
        return $this->hasOne(Phone::class);
    }

    public function team(): BelongsTo
    {
        return $this->belongsTo(Team::class);
    }

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)
                    ->withPivot(['assigned_at'])
                    ->using(RoleUser::class)
                    ->withTimestamps();
    }

    public function commentsThroughPosts(): HasManyThrough
    {
        return $this->hasManyThrough(Comment::class, Post::class);
    }
}
```

### `hasOneOfMany`

Return the single most recent/first of a `hasMany`:

```php
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->ofMany();
}

public function highestPaidOrder(): HasOne
{
    return $this->hasOne(Order::class)->ofMany(['total' => 'max'], fn ($q) =>
        $q->where('status', 'completed')
    );
}
```

### Polymorphic relationships

```php
class Image extends Model
{
    public function imageable(): MorphTo
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function images(): MorphMany
    {
        return $this->morphMany(Image::class, 'imageable');
    }

    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

class Tag extends Model
{
    public function posts(): MorphToMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }
}
```

The database needs an `imageable_type` and `imageable_id` column. `Relation::morphMap(['post' => Post::class, ...])` lets you alias the type so renaming a class doesn't break rows.

## Eager loading and the N+1 problem

Without eager loading, looping `Post` then reading `$post->author` issues one query per post. Eager loading pre-fetches related rows in two queries (one for parents, one IN(...) for children):

```php
$posts = Post::with(['author', 'comments.user'])->get();

Post::with(['author' => fn ($q) => $q->select('id', 'name')])->get();

Post::withCount(['comments' => fn ($q) => $q->where('approved', true)])->get();

$posts->loadMissing('comments');   // lazy, but only if not already loaded
$posts->loadCount('comments');
```

- `with()` eager-loads at query time.
- `load()` eager-loads onto an already-fetched collection.
- `loadMissing()` only loads relations that aren't already loaded.
- `loadCount()` loads counts without the rows.
- `withCount()` loads counts in the original query.
- `with(['rel' => fn ($q) => $q->where(...)])` constrains eager loading.
- `with(['rel:id,post_id,name'])` selects only needed columns (the foreign key must be selected so the join can match).

### Preventing lazy loading

In development, fail loudly on N+1:

```php
// AppServiceProvider::boot()
Model::preventLazyLoading(! app()->isProduction());
```

Any access that triggers a query when the relation was not eager-loaded throws `LazyLoadingViolationException`. This is the single most effective N+1 detector; combine with Telescope/Debugbar (or Pulse/Nightwatch in production) for visibility.

### Inverse hydration with `chaperone()`

A subtle N+1: you eager-load `Post::with('comments')`, then inside each comment access `$comment->post` — Eloquent re-queries the parent even though it already has it in memory. `chaperone()` (Laravel 11.22+) hydrates the inverse automatically:

```php
public function comments(): HasMany
{
    return $this->hasMany(Comment::class)->chaperone();
}
```

## Query Builder vs Eloquent

The query builder (`DB::table(...)`) returns plain `stdClass` rows and has no model lifecycle (no casts, no accessors, no events). Eloquent is built on top of the query builder — every model has a `newQuery()` that returns a builder.

- Use the builder for joins across tables, fast aggregate reads, raw SQL, and reports where you don't need domain logic.
- Use Eloquent when you need casts, relationships, accessors/mutators, events, observers, scopes, or factory/seed integration.

```php
DB::table('posts')
    ->join('users', 'users.id', '=', 'posts.user_id')
    ->where('users.active', true)
    ->select('posts.*', 'users.name')
    ->get();

Post::query()->where('published_at', '<', now())->orderByDesc('id')->cursorPaginate();
```

For very large result sets prefer `cursor()`, `chunk()`, or `lazy()` over `get()`. `lazy()` yields models from the database cursor so memory stays flat.

## Collections

`Illuminate\Database\Eloquent\Collection` extends `Illuminate\Support\Collection`. Base Collection gives you fluent functional methods; the Eloquent collection adds `find`, `load`, `loadCount`, `modelKeys`, `contains(model)`, `fresh`, and `toDictionary`.

```php
$users = User::with('posts')->get();

$active = $users->filter(fn (User $u) => $u->is_active);

$grouped = $users->groupBy(fn (User $u) => $u->role);

[$active, $inactive] = $users->partition(fn (User $u) => $u->is_active);

$totals = $users->reduce(fn (int $carry, User $u) => $carry + $u->post_count, 0);

$names = $users->map(fn (User $u) => $u->name)->unique()->sort()->values();
```

`map` returns a base `Collection`, not an Eloquent one — only methods that reattach the model preserve the Eloquent collection. `eachSpread`, `reduce`, `partition`, `groupBy` are idiomatic for collection pipelines; remember collections are not lazy (use `LazyCollection` for large streams).

## Accessors and mutators

Laravel 9+ introduced the `Attribute` class-based syntax; older `getXAttribute`/`setXAttribute` methods still work.

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function firstName(): Attribute
{
    return Attribute::make(
        get: fn (?string $value) => $value ? ucfirst($value) : null,
        set: fn (string $value) => strtolower($value),
    );
}

protected function fullName(): Attribute
{
    return Attribute::get(fn () => $this->first_name . ' ' . $this->last_name);
}
```

- The closure form auto-caches the result for the model instance.
- Accessors that derive from existing columns (like `fullName` above) need no backing column.
- Add the derived attribute to `$appends` to include it in JSON serialization:

```php
protected $appends = ['full_name'];
```

## Scopes

Local scope — since Laravel 12 the preferred style is the `#[Scope]` attribute on a protected method (no `scope` prefix needed); the classic `scopeX` naming still works:

```php
use Illuminate\Database\Eloquent\Attributes\Scope;

#[Scope]
protected function published(Builder $query, ?bool $published = true): Builder
{
    return $query->where('published_at', $published ? '<=' : '>', now());
}

// Legacy style, still supported:
public function scopePublished(Builder $query): Builder { /* ... */ }

Post::published()->orderBy('id')->get();
```

Global scope:

```php
// AppServiceProvider::boot
Post::addGlobalScope('published', function (Builder $q) {
    $q->whereNotNull('published_at');
});

Post::withoutGlobalScope('published')->get();
```

Or a class-based scope implementing `Scope`:

```php
class ActiveScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('active', true);
    }
}

// in model:
protected static function booted(): void
{
    static::addGlobalScope(new ActiveScope);
}
```

## Soft deletes

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;   // the trait auto-casts deleted_at to datetime
}

Post::find($id)?->delete();   // sets deleted_at
Post::withTrashed()->get();
Post::onlyTrashed()->get();
$post->restore();
$post->forceDelete();
```

Soft-deleted rows are excluded from default queries by a global scope. Querying `withTrashed` opts in; `onlyTrashed` restricts to them.

## Factories and states

Factories build model instances for tests/seeding. Define attributes in `database/factories/ModelFactory.php`:

```php
namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title' => fake()->sentence(),
            'body' => fake()->paragraphs(3, true),
            'user_id' => User::factory(),
            'published_at' => null,
        ];
    }

    public function published(): static
    {
        return $this->state(fn (array $attrs) => [
            'published_at' => now()->subDays(1),
        ]);
    }

    public function withUser(?User $user = null): static
    {
        return $this->state(fn () => ['user_id' => $user->id ?? User::factory()]);
    }
}

Post::factory()->count(5)->published()->create(['user_id' => $user->id]);
```

States are methods returning `$this->state(...)`. Use `factory()->for(User::factory())` to attach relations on creation. `Model::factory()` requires the `HasFactory` trait.

## Migrations

Anonymous migrations are the default since Laravel 9; the class name collision issue is gone.

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $t) {
            $t->id();
            $t->foreignId('user_id')->constrained()->cascadeOnDelete();
            $t->string('title');
            $t->longText('body')->nullable();
            $t->enum('status', ['draft', 'published', 'archived'])->default('draft');
            $t->decimal('price', total: 8, places: 2)->unsigned();
            $t->json('metadata')->nullable();
            $t->timestamp('published_at')->nullable()->index();
            $t->softDeletes();
            $t->timestamps();

            $t->index(['user_id', 'published_at']);
            $t->unique(['slug']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

Common modifiers: `->nullable()`, `->default()`, `->unsigned()`, `->unique()`, `->index()`, `->after('col')` (MySQL), `->useCurrent()` for `timestamps`.

Foreign keys: `foreignId('user_id')->constrained('users')` (table name optional if it follows convention). Add `cascadeOnDelete()` / `restrictOnDelete()` / `nullOnDelete()` for behavior.

Index methods: `index`, `unique`, `fullText` (MySQL), `spatialIndex` (Postgres/MySQL).

Older code used `->onDelete('cascade')` chaining on a `foreign()` call; the fluent `cascadeOnDelete()` is preferred.

### Vector columns and semantic search (Laravel 13)

Laravel 13 added first-party vector support for AI/embedding workloads (PostgreSQL + `pgvector`):

```php
Schema::create('documents', function (Blueprint $t) {
    $t->id();
    $t->text('content');
    $t->vector('embedding', dimensions: 1536);
    $t->timestamps();
});

// Query builder similarity clause — generates embeddings for the search phrase
$docs = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

Embeddings can be generated via the AI SDK (`Str::of($text)->toEmbeddings()`). For interviews, the takeaway is that RAG-style semantic search no longer requires a third-party package — but it does require a database with vector support.

## Database transactions

```php
DB::transaction(function () {
    $order = Order::create([...]);
    OrderItem::insert([...]);
}, attempts: 3);
```

The closure form retries on deadlock; raise the exception if all attempts fail. Manual form:

```php
DB::beginTransaction();
try {
    // nested transactions become SAVEPOINTs
    DB::beginTransaction();
    // ...
    DB::commit();
    DB::commit();
} catch (Throwable $e) {
    DB::rollBack();
    throw $e;
}
```

- `DB::transaction()` rolls back automatically on exception.
- Nested `transaction()` calls produce nested savepoints; a thrown exception rolls back to the nearest savepoint.
- `DB::afterCommit(fn () => dispatch(...))` queues work to fire only after the outermost transaction commits — crucial for jobs/listeners that should not run on rolled-back work.
- Eloquent's save/update/delete is transactional only if you wrap it; by default each statement is its own implicit transaction.

## Mass assignment and timestamps

- Mass assignment (`->fill()`, `::create()`, `::update()`) respects `$fillable`/`$guarded`.
- `$timestamps = true` by default; set `$timestamps = false` to disable.
- Override `CREATED_AT` / `UPDATED_AT` constants to rename columns, or define `getCreatedAtColumn()`.
- Use `$touches = ['parent']` to bump a parent's `updated_at` when a child changes.

## Relationship loading summary

| Goal | Method |
|------|--------|
| Pre-load at query time | `with()` |
| Pre-load counts at query time | `withCount()` |
| Load on existing collection | `load()`, `loadMissing()` |
| Load counts on collection | `loadCount()` |
| Avoid N+1 in dev | `Model::preventLazyLoading(true)` |
| Constrain eager loaded relation | `with(['rel' => fn ($q) => ...])` |
| Load only specific columns | `with(['rel:id,parent_id,name'])` |
