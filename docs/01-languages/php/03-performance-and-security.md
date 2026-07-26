# Performance and Security

This file covers the performance and security topics most often probed in
backend PHP interviews. PHP 8.5 baseline; features from 8.3/8.4 are noted.

---

# Performance

## OPcache recap

OPcache (configuration in `01-core-concepts.md`) is the single most effective
performance lever. Before chasing anything else, confirm:

- `opcache.enable=1` on the web SAPI workers.
- `opcache.memory_consumption` is big enough to hold your full script set
  (~200-512 MiB for mid-size apps; measure with
  `opcache_get_status()['memory_usage']`).
- `opcache.max_accelerated_files` exceeds your script count. PHP rounds up to
  the next prime; pick a value larger than your total file count.
- For production: `opcache.validate_timestamps=0` and clear the cache as part
  of the deploy (restart FPM or call `opcache_reset()` via a CLI/worker
  script with `opcache.enable_cli`). This eliminates the per-request stat
  syscall that `validate_timestamps=1` causes.
- `opcache.file_cache` persists opcache to disk (helpful in containers and
  CLI scripts that run with `opcache.enable_cli`).
- `opcache.preload` and `opcache.preload_user` — preload a script at startup
  (see § Preloading).

## JIT

The JIT (Just-In-Time compiler, 8.0+) compiles hot opcodes to native machine
code via the OPcache shared extension and DynASM. It targets CPU-bound,
tight-loop code — most web workload benefits are smaller than people expect
because typical request time is dominated by I/O (DB, network, templates).

Key settings:

- `opcache.jit` — mode. Use the named modes: `tracing` (recommended;
  numeric equivalent `1254`) or `function`; `disable` / `off` to turn it
  off. The four digits of the numeric form (CRTO):
  - First digit: CPU-specific optimization flags.
  - Second digit: register allocation.
  - Third digit: JIT trigger (`0` compile on load, `1` on first exec, `2`
    profile-based hot functions, `5` tracing hot loops).
  - Fourth digit: optimization level.
- `opcache.jit_buffer_size` — JIT code arena. `0` disables. Set ~64-256 MiB.
- **PHP 8.4 flipped the defaults**: `opcache.jit_buffer_size` now defaults
  to a non-zero size while `opcache.jit=disable`, so enabling JIT is the
  one-liner `opcache.jit=tracing` (pre-8.4 it shipped `jit=tracing` but a
  zero buffer, i.e. effectively off).
- `opcache.jit_debug` — verbose, dev only.

When JIT helps: tight numerical loops (image processing, math-heavy domains,
template engines). When it does not help (and may even slightly slow):
I/O-bound request handlers; an ORM that isn't hot in CPU. Always benchmark.
A common interview answer shape: "JIT primarily benefits CPU-bound code;
most web apps benefit more from caching, query optimization, and async I/O
layers than from JIT."

## Memory management and garbage collection

PHP uses reference counting with a cycle-collecting GC for object/array
cycles. Reachable values are freed eagerly when their refcount drops to 0;
cycles (A references B references A with no external references) are handled
by `zend_gc` periodically via composite runs.

```php
$a = new stdClass; $b = new stdClass;
$a->link = $b; $b->link = $a;
unset($a, $b);                // refcount nonzero due to cycle; GC eventually frees
```

GC configuration:

- `zend.enable_gc` toggles the cycle collector (default `On`).
- `gc_collect_cycles()` forces a cycle pass; `gc_enabled()` /
  `gc_enable()` / `gc_disable()` at runtime.
- `gc_mem_caches()` reclaims collector-internal caches (relevant only for
  long-running CLI / Swoole / RoadRunner processes).
- For long-running workers (Swoole, FrankenPHP persistent mode), explicitly
  `gc_collect_cycles()` periodically or on lifecycle hooks; per-request
  model apps don't worry.

`WeakReference` (7.4+) and `WeakMap` (8.0+) support cache and observer
patterns without preventing GC — the referenced object can be collected, and
the corresponding `WeakMap` entry disappears with it.

```php
$cache = new WeakMap();
$cache[$object] = [...];
unset($object);              // cache entry reaped; no manual eviction
```

Memory leaks matter primarily in long-running processes (queue workers,
daemons). In shared-nothing FPM, request teardown reclaims everything.

## Profiling

Three tool families:

- **Xdebug** — stepping debugger + function-call profiler. Disable in
  production; it adds substantial overhead and disables JIT. Use for
  development-time deep dives.
- **XHProf and descendants** — instrumenting profiler; Tideways is the
  maintained commercial descendant, usable in production at low overhead.
- **Blackfire** — sampling profiler, low overhead, full call graph, callable
  on-demand via CLI or HTTP endpoints. Compare cold start vs warmed cache.
- **Excimer** (Wikimedia) — pure sampling profiler extension; near-zero
  overhead, good for always-on production profiling and flame graphs.
- **APM / tracing** (Tideways, Sentry, Datadog, OpenTelemetry PHP) — spans
  across HTTP/DB/Redis; answers "where did the 2 s go" across services.
- **OPcache statistics** — `opcache_get_status()` exposes hit rate, free
  memory, num cached scripts.
- Built-in `xdebug_get_function_stack`, `memory_get_usage`,
  `memory_get_peak_usage` for targeted instrumenting of hot sections.

Tips for measuring:

- Run profile on representative prod-like traffic. Cold-path profiles lie.
- Compare p95 over 100 reqs, not average; long-tail reveals GC, reindexing,
  lock contention.
- Cache hit rate at the function level (xhprof inclusive vs self time) shows
  "function is hot" but not "the database took 2 seconds"; combine with
  distributed tracing.

## Avoiding N+1 queries

The N+1 anti-pattern: you issue 1 query for a list, then N queries to lazy
fetch related data per row — usually via an ORM getter.

```php
// BAD: fetch list, then 1 query per row bonus
$users = $pdo->query("SELECT id FROM users")->fetchAll();
foreach ($users as $row) {
    $uid = $row['id'];
    $bonus = $pdo->query("SELECT * FROM bonuses WHERE user_id = $uid")->fetch();
    // 100 users => 101 queries
}
```

Solutions, in order of preference:

1. **JOIN** in SQL:

```sql
SELECT u.id, b.amount
FROM users u
LEFT JOIN bonuses b ON b.user_id = u.id;
```

2. **Two-query with IN-list**:

```php
$userIds = array_column($users, 'id');
$placeholders = implode(',', array_fill(0, count($userIds), '?'));
$stmt = $pdo->prepare("SELECT user_id, amount FROM bonuses WHERE user_id IN ($placeholders)");
$stmt->execute($userIds);
$bonuses = array_column($stmt->fetchAll(), 'amount', 'user_id');   // keyed by user_id
foreach ($users as $uid) { /* use $bonuses[$uid] */ }
```

3. **Batch loading** in the ORM (e.g. Eloquent `with(['bonuses'])`,
   Doctrine `->matching(Criteria::where(...))` with `IN` join).
4. **Eager load relationships** at the model/repository boundary so callers
   never implicitly fetch related rows.

The 100-rows-101-queries profile is the smoking gun; same shape arises for
HTTP subrequests and cache walks. Always question "fetch loop inside the
iteration".

## Optimizing hot paths

- Push work out of the per-request path. Compute once at deploy time
  (warm caches, route tables, templates compiled).
- Cache aggressive: opcode (OPcache), preloaded bytecode, APCu/Redis for
  user data, HTTP Cache-Control/CDN for responses.
- Reduce query count: aggregate via joins, use bulk inserts/updates, batch
  `es` operations.
- Use generators when streaming large result sets out of the database with
  `PDO::FETCH_*` and `yield` rows instead of `fetchAll` for huge rows.
- Avoid work in destructors (latency under load); avoid heavy `__get`/`__set`
  auto-properties.
- Disable `opcache.validate_timestamps` in production; precompute route
  matchers; switch autoloader to `--classmap-authoritative`.
- For pure CPU-bound numerical work, JIT it. For I/O wait, use async
  frameworks (Swoole, FrankenPHP persistent, AMP v3).

## Preloading

`opcache.preload` (PHP 7.4+) compiles a configurable script at FPM startup
inside the master process and **permanently** links its classes/functions
into all workers' OPcache. Once preloaded, classes cannot be unloaded;
the entire process must be restarted to clear them. Use preloading for:

- The Composer autoload classmap.
- Framework bootstrap classes (the kernel, container, routes).
- Often-loaded vendor classes (Doctrine, Symfony).

`opcache.preload=/var/www/preload.php`; said file calls `require_once` on
the classmap or otherwise pulls its target classes into memory.

Preload caveats: preloaded classes/functions are **permanently** cached for
the lifetime of the FPM pool — they cannot be unloaded, so changes require a
full FPM process restart (or `opcache_reset()` run from a CLI/PHP interface
within the same SAPI, which itself triggers a reload). On large deploy fleets
many teams wire an FPM reload (`systemctl reload php-fpm`) as the last step
of deploy to refresh preloaded and opcached bytecode uniformly.

## Worker mode and modern runtimes

FPM's per-request model re-bootstraps the framework every request — OPcache
and preloading remove parse/compile cost, but container wiring, config, and
route compilation still run each time. Worker-mode runtimes keep the booted
application resident and feed it many requests:

- **FrankenPHP** — modern app server built on Caddy with PHP embedded.
  Classic mode is a drop-in FPM replacement; **worker mode** keeps the app
  in memory (supported by Laravel Octane and the Symfony Runtime). Ships as
  a single static binary, speaks HTTP/2 and HTTP/3, supports 103 Early
  Hints. The current default answer to "how would you deploy PHP in 2026
  without FPM?"
- **Swoole / OpenSwoole** — event-driven server with coroutines and
  connection pools; the most async-native option.
- **RoadRunner** — Go application server that keeps a pool of persistent
  PHP workers and talks PSR-7 to them.

Trade-offs to volunteer in an interview: worker mode removes bootstrap cost
(often 2–10x throughput on framework-heavy apps) but **breaks the
shared-nothing assumption** — leaked static state, dirty singletons, and
stale DB connections now persist across requests and become your bug class.
You must reset per-request state, watch memory growth, recycle workers
periodically, and in coroutine runtimes ensure nothing blocks the event
loop. Framework integrations (Octane, Symfony Runtime) exist precisely to
manage that reset.

## Common performance pitfalls

- **Slow ORM magic** — `__get`/`__set`, dynamic property defaults. Profile
  inclusive time; replace with explicit getters.
- **Heavy `array_*`** at scale — pairwise `array_merge` in loops (O(n²)).
  Prealloc and merge once at end.
- **Re-computed closures** — object has `fn()` fields re-created on every
  construction; reuse via `static` factory.
- **Stat storms** — `is_readable` per file in a loop; `realpath` cache
  limited; bump `realpath_cache_size` on big projects.
- **`opcache.validate_timestamps=1` in prod** — per-request stat syscalls.
- **Tiny TTL sessions** — replication thrash if session writes one per hit.
- **Logging in hot path** — Monolog log line per request can saturate I/O;
  batch/async log forwarders.
- **Caching rarely-hit keys** — cache lookup itself becomes cost; classify.
- **JIT enabled, but app is I/O bound** — you paid memory + warmup for nothing.
- **Blackfire/Xdebug in production** — overhead breaks the headline metrics.
- **Fiber/event-loop app that calls blocking PDO** — one blocked query
  stalls all fibers in the worker.

---

# Security

## SQL injection

Vulnerable: string concatenation in the query.

```php
// VULNERABLE
$sql = "SELECT id FROM users WHERE email = '" . $_POST['email'] . "'";

// Also vulnerable (still interpolating into query text)
$q = "SELECT id FROM users WHERE email = '" . addslashes($_POST['email']) . "'";
```

Secure: prepared statements with bound parameters, either via PDO or MySQLi.

```php
// SECURE (PDO)
$pdo = new PDO('mysql:host=db;dbname=app;charset=utf8mb4', $user, $pass, [
    PDO::ATTR_EMULATE_PREPARES => false,           // use server-side prepares
    PDO::ATTR_ERRMODE           => PDO::ERRMODE_EXCEPTION,
]);
$stmt = $pdo->prepare('SELECT id FROM users WHERE email = ?');
$stmt->execute([$_POST['email']]);
$uid = $stmt->fetchColumn();

// Named parameters
$stmt = $pdo->prepare('SELECT id FROM users WHERE email = :email AND active = :active');
$stmt->execute([':email' => $_POST['email'], ':active' => 1]);
```

Key rules:

- Never interpolate user input into SQL text, even via `addslashes` or
  `mysqli_real_escape_string` — bound parameters are the canonical defense.
- Whitelist-then-interpolate is the exception: if you must insert a column
  name or sort direction, validate against an allowlist (`['asc','desc']`)
  then interpolate the known-good token.
- `ATTR_EMULATE_PREPARES=false` forces real server-side prepared statements,
  which avoids a class of edge cases in emulated prepares and enforces
  server-side type handling.
- ORM query builders (Doctrine `QueryBuilder`, Laravel `DB::table`)
  parameterize their WHERE clauses under the hood. Raw helpers that bypass
  parameterization exist (Laravel `DB::unprepared()`, Doctrine
  `NativeQuery` with concatenated SQL) — never feed user-controlled strings
  into them.

## XSS

Code injection into HTML. Defense: **context-aware output encoding**. For
HTML body contexts the canonical escape is `htmlspecialchars` with the
right flags:

```php
function esc(string $s): string {
    return htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML5, 'UTF-8', true);
}

echo '<a title="' . esc($row['title']) . '">' . esc($row['text']) . '</a>';
```

`ENT_QUOTES` escapes single quotes too (needed for single-quoted attributes);
`ENT_HTML5` is the modern flag; the charset MUST be the page charset
(UTF-8 by default in 8.1+; explicit is safe).

Context rules (the heart of XSS, often under-stressed):

- **HTML body**: `htmlspecialchars` (above).
- **Attribute** (single or double quoted): same; ensure quoting is intact.
- **JavaScript string literal inside `<script>`**: escape with a JSON-ish
  transliteration `json_encode($s, JSON_HEX_TAG | JSON_HEX_AMP | ...)`
  AND/OR avoid embedding user data in `<script>` at all; pass via data
  attributes.
- **URL/`href`**: validate scheme (allowlist `http(s)`, `mailto`); reject
  `javascript:`.
- **CSS**: avoid embedding user data; if needed, escape with `\HHHHHH`.
- **Inner SVG/MathML**: extra rules; prefer templating framework that
  knows SVG context.

Template engines (Twig, Blade) auto-escape HTML context by default; you must
opt-in to raw output (`|raw`) and only do so with sanitizer knowledge. The
default-escape property of modern templating is one of the strongest
defenses; do not circumvent it for "convenience".

`Content-Security-Policy` headers limit the damage even when an XSS slips
through: `script-src 'self' 'nonce-...'`.

## CSRF

Cross-Site Request Forgery: an attacker site triggers the user's browser to
submit a logged-in session request.

Defenses:

- **Anti-CSRF tokens** per session/form. The form gets a nonce; the server
  compares submitted to session-stored nonce.
- **`SameSite` cookies** (`SameSite=Lax` for top-level navigations;
  `Strict` for fully prevention of cross-site Sends). Most modern browsers
  require explicit `SameSite` and default to `Lax`.
- **`POST` semantics** for any state-changing action; `GET` only reads.
- **Idempotent tokens / double-submit cookie**.
- **`Origin`/`Referer` header check** as defense-in-depth (proxyable; not
  primary).
- **Verifying using per-form tokens tied to the session** (not just signed
  global tokens).

```php
// Generate
$csrf = bin2hex(random_bytes(32));
$_SESSION['csrf_token'] = $csrf;          // store; rotate per session lifetime
echo '<input type="hidden" name="csrf" value="' . $csrf . '">';

// Verify
if (!hash_equals($_SESSION['csrf_token'] ?? '', $_POST['csrf'] ?? '')) {
    http_response_code(419); exit;
}
```

`hash_equals` is constant-time comparison to avoid string-compare timing
oracles.

## SSRF

Server-Side Request Forgery: the server makes an outbound HTTP request from
user-supplied URL. Ban lists are insufficient — DNS rebinding, IPv6 fallback,
cloud metadata IPs (`169.254.169.254`), 30x redirects to internal addresses
all bypass naive filters.

```php
// VULNERABLE
$ch = curl_init($_POST['url']);
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
$response = curl_exec($ch);
```

Mitigations:

- **Allowlist** of permitted hosts/schemes/ports (`https`, port 443, a fixed
  list of hosts) when known.
- **Resolve and pin the IP**: do `gethostbynamel` and only connect to IPs that
  you've validated as not in private/reserved ranges.
- **Disable redirects** (`CURLOPT_FOLLOWLOCATION=false`) or revalidate each
  redirect URL.
- **Block link-local/private ranges** (RFC 1918, RFC 4193, `127.0.0.0/8`,
  `169.254.0.0/16`, IPv6 `fc00::/7`, `fe80::/10`).
- **Use a forward proxy** that enforces egress rules; better to have a
  network-layer control than code-based ones.

## Insecure deserialization

`unserialize()` on untrusted data is **RCE-equivalent**: PHP instantiates
any class named in the payload, and `__wakeup`/`__unserialize` get to run
arbitrary code. The classic exploit: a magic method chain drives payloads
from a "gadget chain" available in the project's vendor tree.

```php
// VULNERABLE
$data = unserialize($_COOKIE['state']);
// A crafted "O:..." blob runs __wakeup chains in vendor classes.

// SECURE: use a data format that does not invoke arbitrary code paths.
$json = json_decode($_COOKIE['state'], true, flags: JSON_THROW_ON_ERROR);
```

If you must use `serialize`:

- `unserialize($payload, ['allowed_classes' => false])` (7.0+) needs no
  class instantiation. `[AllowedClass::class]` for per-class.
- Sign the payload with HMAC for transport integrity (not secrecy):

```php
$hmac = hash_hmac('sha256', $payload, $secret);
if (!hash_equals($hmac, $sent)) { throw new InvalidToken(); }
$data = unserialize($payload, ['allowed_classes' => [SafeDto::class]]);
```

Even with HMAC, modern code should prefer JSON + DTOs.

## Password hashing

The canonical hashing APIs are `password_hash()` and
`password_verify()`. They auto-select salt and cost; do not roll your own hash.

```php
$hash = password_hash($plain, PASSWORD_DEFAULT);     // bcrypt by default in distros
if (password_verify($input, $hash)) { /* ok */ }

// Argon2 (modern, memory-hard; needs argon2 dlopen)
$hash = password_hash($plain, PASSWORD_ARGON2ID, [
    'memory_cost' => 1 << 17,   // 128 MiB
    'time_cost'   => 4,
    'threads'     => 2,
]);
```

Rules:

- `PASSWORD_DEFAULT` is `bcrypt` in standard builds; `password_needs_rehash`
  lets you upgrade on next successful login as algorithms/cost change.
- Bcrypt cost factor: ~10-12 (measured to take ~100-300 ms on your hardware).
- Bcrypt truncates passwords at 72 bytes — for arbitrary-length passwords
  apply HMAC-SHA-256 first (`hash_hmac('sha256', $plain, $serverPepper)`)
  then pass the binary output to bcrypt; this also lets you *pepper*.
- Never roll your own hash. Never store MD5/SHA1/SHA-256 (these are too fast
  for an offline attacker with a leaked DB).
- Rate-limit failed attempts in addition to slow hashing.
- Have security review any custom password flow; prefer framework-provided
  abstractions (Symfony PasswordHasher, Laravel Hash).

## Session security

`session_start()` bootstraps a cookie-based session ID. Defaults are
*not* sufficient on their own. Secure production baseline:

```php
session_set_cookie_params([
    'lifetime' => 0,            // cookie valid for the browser session length
    'path'     => '/',
    'domain'   => '',            // current host, no implicit broad domain
    'secure'   => true,         // require HTTPS
    'httponly' => true,         // hide from JS -> closes XSS -> session theft class
    'samesite' => 'Lax',        // or 'Strict'; defense against CSRF
]);
ini_set('session.use_strict_mode', '1');   // reject attacker-suggested IDs
session_start();
```

### Session fixation

An attacker sends the victim a URL with a known session ID
(`?PHPSESSID=attacker-supplied`). If the server accepts the ID as-is, the
attacker becomes a co-holder of the session after the victim logs in.

Defense: **regenerate the session ID on privilege change** (login, password
change, role escalation):

```php
session_regenerate_id(true);              // delete old; send new cookie
```

`use_strict_mode=1` requires the server to reject unknown IDs (the server
must already have a record for the ID); combined with regeneration on login
this is the canonical fixation defense.

Other hardening:

- `session.gc_maxlifetime` and `session.cookie_lifetime` align.
- Storage backend protection: in a multi-server deployment, store sessions
  in Redis/Memcached with TLS access limited to the app nodes; files-on-disk
  default fails multi-node.
- Implement idle and absolute timeouts server-side; cookie `Max-Age` alone
  is not a security boundary.
- Logout invalidates the server-side record, not just clears the cookie:
  `session_destroy()` after `session_regenerate_id(true)`.

## Input validation

Validate at the boundary (controller) and *fail loudly*. Use `filter_var`,
typed casts, and value objects.

```php
$email = filter_var($_POST['email'] ?? '', FILTER_VALIDATE_EMAIL)
    ?? throw new InvalidEmail();
$age = filter_var($_POST['age'] ?? '', FILTER_VALIDATE_INT,
    ['options' => ['min_range' => 18, 'max_range' => 120]]) ?? throw new InvalidAge();

// Prefer a small library: https://github.com/Respect/Validation, Valitron, or Laravel validators.
```

Whitelist validation > blacklist. Reject by default; allow explicitly; emit
typed domain objects internally (`Email` value object) so you can't slip
garbage past the boundary.

`$_POST['id']` cast to `int` is validation plus normalization; use
input-format libraries when the input shape is complex.

## File upload security

```php
if ($_FILES['file']['error'] !== UPLOAD_ERR_OK) { /* log + reject */ }

if ($_FILES['file']['size'] > 5_000_000) { reject('too large'); }

$allowedMimes = ['image/jpeg', 'image/png', 'image/webp'];
$finfo = new finfo(FILEINFO_MIME_TYPE);
if (!in_array($finfo->file($_FILES['file']['tmp_name']), $allowedMimes, true)) {
    reject('unsupported type');
}

// Avoid trusting $_FILES['file']['name'] for storage path
$safeName = bin2hex(random_bytes(16)) . '.jpg';
$dest = '/var/www/uploads/' . $safeName;        // outside the docroot
move_uploaded_file($_FILES['file']['tmp_name'], $dest);
```

Rules:

- Check `$_FILES['...']['error']` first; an error code isn't just a "no
  file" case — it can be ATTACK_TRUNCATED or partial upload.
- Validate size server-side.
- Use `finfo` to inspect actual bytes, never trust `$_FILES['...']['type']`
  (set by the client).
- Don't use the user-supplied name for the on-disk filename; use a random
  generated name to avoid path traversal and collisions.
- Store outside the web docroot; serve via a controller that re-checks MIME
  and sets Content-Disposition.
- For images, re-encode them if you need to strip exif/embedded payloads.

## Composer audit / dependency CVEs

```bash
composer audit                # exit ≠ 0 if CVEs found in installed deps
composer audit --format=plain
composer audit --locked       # audit against composer.lock without installing
```

Integrate in CI:

```bash
composer audit --locked --no-dev --no-interaction "&&" composer test
```

Subscribed orgs use additional platforms (Roave/SecurityAdvisories,
GitHub Dependabot, Snyk) for cross-validation and for advisories that
packagist's data hasn't picked up. `roave/security-advisories` is a
`*-dev` package that blocks installing known-vulnerable versions by pinning
constraints through the require graph.

`composer outdated --direct` exposes known-good updates that you may be
missing.

## Secrets handling

- 12-factor: secrets come from environment variables or a secrets manager
  (HashiCorp Vault, AWS Secrets Manager, Doppler), not from the repository.
  Symfony's secrets vault (sodium-encrypted files with the decrypt key
  outside the repo) is a viable framework-native option.
- Never commit a `.env.local` with prod secrets. Add to `.gitignore`.
- Don't `dd()`, log, or trace stack back to error pages with full context
  (Symfony `prod` should set `APP_ENV=prod` and hide framework debug toolbars
  in `prod`).
- Use a tool like `gitleaks` or `trufflehog` to scan history.
- Cache credentials for downstream services should be short-lived and
  refreshable (OIDC tokens, AWS IAM role sessions). Avoid long-lived DB
  passwords. Rotate.
- Rate-limit and lock the secrets endpoint: a leaked secret or stack trace
  should not also dump secrets.