# Next.js

This section covers **Next.js 15** (App Router, React 19, Server Components) from the perspective of a **backend engineer** preparing for interviews. Next.js is often thought of as a frontend framework, but its server-side capabilities — rendering strategies, data fetching, Route Handlers, Server Actions, caching, and middleware — are exactly the areas probed in backend and full-stack interviews. The notes here emphasize those server-side concerns while staying accurate for a full-stack context.

> Last reviewed against: **Next.js 15** (App Router default), **React 19** (stable Server Components, Actions), Turbopack stable for dev. The Pages Router is covered only where it clarifies App Router migration/contrast.

## Why this matters for backend interviews

Backend candidates are frequently asked to build or reason about a BFF (backend-for-frontend), an isomorphic data layer, or an SSR-heavy app. Next.js sits at that boundary. Interviewers use it to probe:

- **Rendering strategy trade-offs** — CSR vs SSR vs SSG vs ISR, and when streaming wins.
- **The server/client component boundary** — what runs where, what can be imported where, serialization rules.
- **Caching** — the four Next.js caching layers, and the **Next.js 15 default change** (fetch/Route Handlers no longer cached by default), a very common question.
- **Data fetching & mutations** — extended `fetch`, Route Handlers, and Server Actions as an alternative to hand-written REST endpoints.
- **Edge vs Node runtimes** — middleware constraints, where Route Handlers can run.

## Files in this section

| # | File | Description |
|---|------|-------------|
| 1 | [`01-rendering-strategies.md`](./01-rendering-strategies.md) | CSR/SSR/SSG/ISR/streaming, RSC vs Client Components, the server/client boundary, Next 15 caching defaults, static vs dynamic rendering, PPR. |
| 2 | [`02-data-fetching-and-api-routes.md`](./02-data-fetching-and-api-routes.md) | Extended `fetch`, the four caching layers, revalidation, Route Handlers, Server Actions, middleware, layouts/routing, metadata, async `params`/`searchParams`. |
| 3 | [`03-interview-questions.md`](./03-interview-questions.md) | 20+ Q&A grouped by difficulty, with model answers. |

## Recommended reading order

1. **`01-rendering-strategies.md`** — Establishes the mental model (where code runs, how HTML is produced) that everything else depends on. Read this first; the App Router's server/client split is the single most-tested concept.
2. **`02-data-fetching-and-api-routes.md`** — Builds on rendering by showing how data flows to the server, how it's cached/invalidated, and how you expose endpoints/mutations (Route Handlers, Server Actions).
3. **`03-interview-questions.md`** — Self-test. Attempt each answer out loud before reading the model answer, since articulation matters as much as correctness.

## Conventions used

- Code examples are idiomatic **App Router TypeScript/TSX** for Next.js 15.
- Where behavior changed between Next.js 14 and 15, the difference is called out explicitly (caching defaults and async `params`/`searchParams` are the two biggest changes).
- Experimental features (e.g., Partial Prerendering, the `use cache` directive) are clearly marked as such.

## Sources

Content is synthesized from the official Next.js documentation (https://nextjs.org/docs) and the React 19 reference (https://react.dev). Always cross-check against the latest docs before relying on specifics — Next.js evolves quickly and some defaults have shifted between major versions.
