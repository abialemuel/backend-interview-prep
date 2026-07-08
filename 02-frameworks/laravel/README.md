# Laravel Interview Prep

This section covers Laravel **12.x**, with callouts to the structural changes introduced in Laravel 11 that remain relevant in 12: the slim application skeleton, the consolidated `bootstrap/app.php` bootstrap file, `bootstrap/providers.php` for provider registration, and the removal of `app/Http/Kernel.php` and `app/Console/Kernel.php` in favor of fluent configuration in `bootstrap/app.php`.

The goal is to take a working backend engineer from "I have built a few Laravel apps" to "I can answer deep architectural and framework-internals questions in an interview." Topics span the request lifecycle, the service container, Eloquent internals, routing, queues, testing, and security.

## Files in this section

| # | File | What it covers |
|---|------|----------------|
| 1 | `01-architecture-and-lifecycle.md` | Request lifecycle, the service container, service providers, facades, contracts, middleware pipeline, slim skeleton, configuration, lifecycle hooks. |
| 2 | `02-eloquent-and-database.md` | Eloquent models, relationships (incl. polymorphic and `hasOneOfMany`), eager loading and the N+1 problem, collections, accessors/mutators via `Attribute`, scopes, soft deletes, factories, migrations, transactions. |
| 3 | `03-routing-middleware-and-queues.md` | Routing and route model binding, middleware (terminable, parameters, priority), form requests, validation, the queue subsystem, events/listeners, broadcasting, scheduling. |
| 4 | `04-testing-and-security.md` | PHPUnit/Pest, HTTP and database tests, fakes (`Bus`, `Queue`, `Event`, `Mail`, `Notification`), Dusk, snapshots; authentication, gates/policies, CSRF, encryption, hashing, XSS, Sanctum vs Passport. |
| 5 | `05-interview-questions.md` | 28+ questions split into Easy / Medium / Hard with model answers. |

## Recommended reading order

1. `01-architecture-and-lifecycle.md` — Establishes the mental model (container, providers, facades) the rest of the docs assume.
2. `02-eloquent-and-database.md` — Builds on DI for scopes/repositories and is the most commonly tested domain.
3. `03-routing-middleware-and-queues.md` — Routing + queues rely on the container and middleware pipeline from file 1.
4. `04-testing-and-security.md` — Testing relies on the container (fakes, facades); security relies on middleware and routing knowledge.
5. `05-interview-questions.md` — Use as a self-test after the first four; cross-references back to the reference files.

## Version notes

- **Laravel 12.x** ships with the same slim skeleton introduced in Laravel 11; first-party WebSocket server **Reverb**, starter kits for Livewire and React/Inertia, and **Laravel Concurrency** are notable 11.x/12.x additions.
- **Laravel 11** introduced the slim skeleton, `bootstrap/app.php`-driven configuration (middleware, exceptions, routing, commands), `bootstrap/providers.php`, `routes/console.php` for commands, and a health route.
- Code examples target PHP 8.2+ syntax (constructor property promotion, readonly props, enums, `#[...]` attributes where relevant).
