# PHP — Backend Interview Prep

This section of the repository covers PHP as a backend language, from core
concepts through modern object-oriented PHP, performance, security, and a
curated set of interview questions. It targets **PHP 8.5 (Nov 2025)** as the
current stable release, with explicit notes where a feature was introduced in
PHP 8.3 or 8.4 and is still relevant. Code idioms assume PHP 8.3+ syntax by
default (constructor property promotion, readonly, enums, `match`, named
arguments, fibers, intersection/union/DNF types, property hooks, asymmetric
visibility).

## Files in this section

| File | Description |
| ---- | ----------- |
| `01-core-concepts.md` | Types, variables, arrays, functions, closures, arrow functions, generators, fibers, enums, readonly, property hooks, asymmetric visibility, request lifecycle, OPcache. |
| `02-oop-and-modern-php.md` | Classes, inheritance, interfaces, traits, enums, static/self/parent, late static binding, type declarations (union/intersection/DNF), SOLID, DI, service containers, PSR standards, Composer, attributes, fibers. |
| `03-performance-and-security.md` | OPcache, JIT, GC, profiling, N+1, preloading, SQLi, XSS, CSRF, SSRF, deserialization, password hashing, sessions, uploads, Composer audit. |
| `04-interview-questions.md` | 25+ interview questions grouped by difficulty, with model answers. |

## What this covers

- **PHP 8.5** (Nov 2025) — asymmetric visibility on properties (incl. promoted), further JIT improvements, internal API hardening/deprecations; the baseline syntax unless noted.
- **PHP 8.4** (Nov 2024) — property hooks, new array functions (`array_find`, `array_find_key`, `array_any`, `array_all`), `new` expressions in initializers, typed class constants enforcement. (Asymmetric visibility was designed in parallel but landed in 8.5.)
- **PHP 8.3** (Nov 2023) — typed class constants, `json_validate`, dynamic class constant fetch, `#[\Override]` attribute, `Randomizer` additions, granular `E_*` levels.

Older features (match, enums, readonly, fibers, intersection types, named
arguments, constructor property promotion) are treated as baseline and used
freely without calling out their introduction version.

## Recommended reading order

1. `01-core-concepts.md` — establish the mental model of how a PHP request
   runs and the core language primitives.
2. `02-oop-and-modern-php.md` — the idioms you will be expected to produce in
   an interview coding round.
3. `03-performance-and-security.md` — where most senior backend questions
   actually land.
4. `04-interview-questions.md` — self-test after the above, or warm up before
   the loop.

Read top to bottom the first time; afterwards you can treat each file as an
independent reference. Cross-references between files use the form
"`02-oop-and-modern-php.md` § Enums".