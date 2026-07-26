# Laravel Interview Prep

This section covers Laravel **13.x** (released March 2026, PHP 8.3+), with callouts to the structural changes introduced in Laravel 11 that remain the foundation of 12 and 13: the slim application skeleton, the consolidated `bootstrap/app.php` bootstrap file, `bootstrap/providers.php` for provider registration, and the removal of `app/Http/Kernel.php` and `app/Console/Kernel.php` in favor of fluent configuration in `bootstrap/app.php`.

The goal is to take a working backend engineer from "I have built a few Laravel apps" to "I can answer deep architectural and framework-internals questions in an interview." Topics span the request lifecycle, the service container, Eloquent internals, routing, queues, testing, and security.

## Files in this section

| # | File | What it covers |
|---|------|----------------|
| 1 | `01-architecture-and-lifecycle.md` | Request lifecycle, the service container, service providers, facades, contracts, middleware pipeline, slim skeleton, configuration, lifecycle hooks. |
| 2 | `02-eloquent-and-database.md` | Eloquent models, relationships (incl. polymorphic and `hasOneOfMany`), eager loading and the N+1 problem, collections, accessors/mutators via `Attribute`, scopes, soft deletes, factories, migrations, transactions. |
| 3 | `03-routing-middleware-and-queues.md` | Routing and route model binding, middleware (terminable, parameters, priority), form requests, validation, the queue subsystem, events/listeners, broadcasting, scheduling. |
| 4 | `04-testing-and-security.md` | PHPUnit/Pest, HTTP and database tests, fakes (`Bus`, `Queue`, `Event`, `Mail`, `Notification`), Dusk, snapshots; authentication, gates/policies, CSRF, encryption, hashing, XSS, Sanctum vs Passport. |
| 5 | `05-interview-questions.md` | 35 questions split into Easy (junior) / Medium (senior) / Hard (staff) with model answers and grading notes. |

## Recommended reading order

1. `01-architecture-and-lifecycle.md` — Establishes the mental model (container, providers, facades) the rest of the docs assume.
2. `02-eloquent-and-database.md` — Builds on DI for scopes/repositories and is the most commonly tested domain.
3. `03-routing-middleware-and-queues.md` — Routing + queues rely on the container and middleware pipeline from file 1.
4. `04-testing-and-security.md` — Testing relies on the container (fakes, facades); security relies on middleware and routing knowledge.
5. `05-interview-questions.md` — Use as a self-test after the first four; cross-references back to the reference files.

## Version notes

- **Laravel 13.x** (March 2026, PHP 8.3–8.5) added the first-party **AI SDK** (agents, embeddings, `Str::toEmbeddings()`), **JSON:API resources**, **vector/semantic search** (`whereVectorSimilarTo` with pgvector), **`Queue::route()`** for central job routing, the **`PreventRequestForgery`** origin-aware CSRF middleware, `Cache::touch()`, and expanded first-party **PHP attributes** (`#[Middleware]`, `#[Authorize]` on controllers; `#[Tries]`, `#[Backoff]`, `#[Timeout]`, `#[FailOnTimeout]` on jobs). It was deliberately a low-breakage upgrade.
- **Laravel 12.x** (Feb 2025) kept the slim skeleton and shipped new starter kits (Livewire, React/Inertia, Vue); first-party WebSocket server **Reverb** and **Concurrency** landed in the 11.x cycle. Laravel 12 receives bug fixes until August 2026; Laravel 11 is end-of-life as of March 2026.
- **Laravel 11** introduced the slim skeleton, `bootstrap/app.php`-driven configuration (middleware, exceptions, routing, commands), `bootstrap/providers.php`, `routes/console.php` for commands, and a health route.
- First-party packages worth knowing by name for interviews: **Octane** (long-lived app server), **Horizon** (Redis queue dashboard), **Reverb** (WebSockets), **Pennant** (feature flags), **Precognition** (live validation), **Pulse** (app metrics), **Folio** (page-based routing).
- Code examples target PHP 8.3+ syntax (constructor property promotion, readonly props, enums, `#[...]` attributes where relevant).
