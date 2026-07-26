# Next.js

This section covers **Next.js 16** (App Router, React 19.2, Server Components) from the perspective of a **backend engineer** preparing for interviews. Next.js is often thought of as a frontend framework, but its server-side capabilities — rendering strategies, data fetching, Route Handlers, Server Actions, caching, and the proxy layer — are exactly the areas probed in backend and full-stack interviews. The notes here emphasize those server-side concerns while staying accurate for a full-stack context.

> Last reviewed against: **Next.js 16.x** (mid-2026; 16 shipped October 2025, 16.3 in preview) — App Router default, **Turbopack** the default bundler for dev and build, **Cache Components / `"use cache"`** as the opt-in caching model, and `proxy.ts` replacing `middleware.ts`. The Pages Router is legacy and covered only where it clarifies migration/contrast.

## Why this matters for backend interviews

Backend candidates are frequently asked to build or reason about a BFF (backend-for-frontend), an isomorphic data layer, or an SSR-heavy app. Next.js sits at that boundary. Interviewers use it to probe:

- **Rendering strategy trade-offs** — CSR vs SSR vs SSG vs ISR, and when streaming wins.
- **The server/client component boundary** — what runs where, what can be imported where, serialization rules.
- **Caching** — the four Next.js caching layers, the **Next.js 15 default change** (fetch/Route Handlers no longer cached by default), and the **Next.js 16 Cache Components model** (`"use cache"`, `cacheLife`, `updateTag`) — the most common question cluster in 2026 interviews.
- **Data fetching & mutations** — extended `fetch`, Route Handlers, and Server Actions as an alternative to hand-written REST endpoints.
- **The network boundary** — `proxy.ts` (formerly `middleware.ts`), Edge vs Node runtimes, where Route Handlers can run.

## Files in this section

| # | File | Description |
|---|------|-------------|
| 1 | [`01-rendering-strategies.md`](./01-rendering-strategies.md) | CSR/SSR/SSG/ISR/streaming, RSC vs Client Components, the server/client boundary, Next 15/16 caching defaults, static vs dynamic rendering, PPR via Cache Components. |
| 2 | [`02-data-fetching-and-api-routes.md`](./02-data-fetching-and-api-routes.md) | Extended `fetch`, the four caching layers, `"use cache"` and Cache Components, revalidation (`revalidateTag`/`updateTag`/`refresh`), Route Handlers, Server Actions, `proxy.ts`, layouts/routing, metadata, async `params`/`searchParams`. |
| 3 | [`03-interview-questions.md`](./03-interview-questions.md) | 30 Q&A graded junior/senior/staff, with model answers. |

## Recommended reading order

1. **`01-rendering-strategies.md`** — Establishes the mental model (where code runs, how HTML is produced) that everything else depends on. Read this first; the App Router's server/client split is the single most-tested concept.
2. **`02-data-fetching-and-api-routes.md`** — Builds on rendering by showing how data flows to the server, how it's cached/invalidated, and how you expose endpoints/mutations (Route Handlers, Server Actions).
3. **`03-interview-questions.md`** — Self-test. Attempt each answer out loud before reading the model answer, since articulation matters as much as correctness.

## Conventions used

- Code examples are idiomatic **App Router TypeScript/TSX** for Next.js 16.
- Where behavior changed across Next.js 14 → 15 → 16, the difference is called out explicitly (caching defaults, async `params`/`searchParams` becoming mandatory, `middleware.ts` → `proxy.ts`, and the `experimental.ppr` flag being folded into Cache Components are the big ones).
- Opt-in features (Cache Components / `"use cache"`) and beta features (Turbopack filesystem caching) are marked with their actual status — don't present opt-in as default in an interview.

## Sources

Content is synthesized from the official Next.js documentation (https://nextjs.org/docs) and the React 19 reference (https://react.dev). Always cross-check against the latest docs before relying on specifics — Next.js evolves quickly and some defaults have shifted between major versions.
