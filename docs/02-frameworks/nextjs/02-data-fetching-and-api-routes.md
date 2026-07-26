# Data Fetching, Route Handlers, Server Actions, and Routing

This file covers how data gets into your server components, how Next.js caches it, how you expose endpoints (Route Handlers) and mutations (Server Actions), and the routing/layout primitives of the App Router. Everything here is App Router / Next.js 16.

---

## The Extended `fetch` API

Next.js extends the Web `fetch` API with a `next` option that controls caching and revalidation. This is the primary data-fetching primitive for Server Components.

```tsx
// Time-based revalidation (ISR): revalidate at most every 60s
const res = await fetch('https://api.example.com/articles', {
  next: { revalidate: 60 },
});

// On-demand revalidation: tag the request, invalidate later
const res = await fetch('https://api.example.com/articles', {
  next: { tags: ['articles'] },
});

// Explicitly skip the cache (the default since Next 15, stated explicitly)
const res = await fetch('https://api.example.com/live', {
  cache: 'no-store',
});

// Cache indefinitely (build-time only)
const res = await fetch('https://api.example.com/config', {
  cache: 'force-cache',
});
```

| Option | Meaning |
|--------|---------|
| `cache: 'no-store'` | Never cache; always fetch fresh. |
| `cache: 'force-cache'` | Cache indefinitely (until revalidated). |
| `next: { revalidate: seconds }` | Cache and revalidate after N seconds (time-based ISR). |
| `next: { tags: [...] }` | Cache and tag for **on-demand** invalidation via `revalidateTag`. |

> **Default since Next 15 (unchanged in 16):** `fetch` is **not cached** unless you opt in. In Next 14 it was cached by default. This is the most-tested behavioral change.

### Why is `fetch` extended at all?

Because Next.js needs cache controls to be **per-request** and to integrate with its on-demand invalidation system (`revalidateTag`). The standard `fetch` has no notion of "tag this entry so I can bust it later," and no concept of time-based revalidation across renders. Extending `fetch` (rather than inventing a separate API) means you can use the same familiar API in Server Components, Route Handlers, and (via Server Actions) after mutations.

### Data fetching mental model

In the App Router you fetch data **in the Server Component that needs it**, directly. There is no `getServerSideProps` / `getStaticProps` — async Server Components replace them. Fetch as close to where data is used as possible; Next's request memoization (below) deduplicates identical fetches within a render pass, so you don't need to hoist all fetching to the top.

```tsx
// app/posts/page.tsx
export default async function Page() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] },
  }).then((r) => r.json());
  return <PostList posts={posts} />;
}
```

For non-`fetch` data sources (ORM queries, third-party SDKs), the Next 16 answer is the **`"use cache"` directive** (with Cache Components enabled) — it caches any async function, and the compiler derives the cache key from the function's closure and arguments:

```tsx
import { cacheLife, cacheTag } from 'next/cache';
import { db } from '@/lib/db';

export async function getProducts() {
  'use cache';
  cacheLife('hours');        // built-in profile: seconds/minutes/hours/days/max, or custom
  cacheTag('products');      // enables on-demand invalidation via revalidateTag/updateTag
  return db.product.findMany();
}
```

`"use cache"` works at three granularities: at the top of a file (all exports cached), on a component (caches its rendered output), or on a function. The legacy `unstable_cache` API still exists for apps that haven't enabled `cacheComponents`, but it's superseded — mentioning `"use cache"` is the current-knowledge signal in interviews:

```tsx
// Legacy (pre-Cache Components) equivalent
import { unstable_cache } from 'next/cache';
const getProducts = unstable_cache(
  async () => db.product.findMany(),
  ['products'],
  { revalidate: 60, tags: ['products'] }
);
```

---

## The Four Caching Layers

Next.js has four distinct caches. Knowing them, what invalidates each, and how they interact is a core interview topic.

### 1. Request Memoization (per-request)

Within a **single server render pass** (one request), identical `fetch` calls with the same URL + options are deduplicated. This is in-memory and lasts only for the request. It means a Server Component and a child can both `fetch` the same URL without hitting the origin twice.

```tsx
// Parent and child both fetch /user/1 — the origin is hit once for the request
async function Parent() {
  const u = await fetch('/api/user/1').then((r) => r.json());
  return <><Header />{u.name}<Child /></>;
}
async function Child() {
  const u = await fetch('/api/user/1').then((r) => r.json()); // memoized, no new call
  return <span>{u.email}</span>;
}
```

Not shared across requests or users. Not persisted to disk.

### 2. Data Cache (across requests, across deployments)

Stores the **result of `fetch`** keyed by URL + options, **across requests** (and survives deploys on Vercel). This is where `next.revalidate`, `next.tags`, `cache: 'force-cache'` / `'no-store'` apply. Since Next 15 this is **opt-in** — without an opt-in, `fetch` behaves like the browser (no Data Cache entry). With Cache Components (Next 16), `"use cache"` output lives in this same layer, governed by `cacheLife`/`cacheTag`.

Invalidated by:
- Time-based expiry (`next.revalidate`).
- On-demand: `revalidateTag(tag)` or `revalidatePath(path)`.
- Manual: `revalidatePath` / `revalidateTag` called from Server Actions, Route Handlers, or `revalidate`-after hooks.

### 3. Full Route Cache (static route HTML)

When a route is **statically rendered**, Next.js caches the full rendered HTML + RSC payload at build (or first request). Subsequent requests get the prerendered result without re-executing the component tree. This is why static routes are so fast.

Invalidated when:
- The underlying Data Cache entries are invalidated (because the route depends on that data).
- A new deploy.
- `revalidatePath` / `revalidateTag` for the route.

Dynamic routes (using `cookies()`/`headers()`/`searchParams`) are **never** Full-Route-Cached — they re-render per request.

### 4. Router Cache (client-side, in-memory)

A client-side, in-memory cache of the RSC payload for **navigated routes**, stored in the browser for the duration of the session. It makes client-side navigation (Link clicks) instant and powers back/forward. Since Next 15 the default `staleTimes` are **0 for dynamic routes** (always refetched on navigation — the Next 14 30s default caused notorious staleness bugs) and 5 minutes for static/prefetched routes; both are configurable. Next 16 rewrote the prefetch layer on top of this: shared layouts are downloaded once across many links (deduplication) and only the missing parts of a page are prefetched (incremental prefetching).

Invalidated by:
- `router.refresh()` — refetches current route's RSC payload.
- Navigation to the route after it expired.
- Server Action returning a `revalidate`/`revalidateTag`/`revalidatePath` — which causes the client to refetch affected routes.

### How they interact

A typical static + ISR flow: build time → component renders → `fetch` populates the **Data Cache** → full HTML stored in **Full Route Cache** → on request, CDN serves the prerendered HTML; on client navigation, the **Router Cache** serves the RSC payload. When `revalidateTag('articles')` runs, it busts the **Data Cache** entry, which cascades to invalidate the **Full Route Cache** for dependent routes, and the Server Action response tells the client to drop the affected **Router Cache** entries.

---

## Revalidation

### Time-based

```tsx
await fetch(url, { next: { revalidate: 60 } });
// or route-level
export const revalidate = 60;
```

Stale-while-revalidate: the current cached value is served immediately and regenerated in the background; the *next* request gets the fresh value.

### On-demand

Next 16 refined the on-demand APIs into three tools with different consistency semantics:

```tsx
import { revalidateTag, revalidatePath, updateTag, refresh } from 'next/cache';

// Route Handler or Server Action: stale-while-revalidate invalidation
export async function POST(req: Request) {
  await updateArticle();
  revalidateTag('articles', 'max');  // Next 16: cacheLife profile required as 2nd arg
  revalidatePath('/articles');       // bust the Full Route Cache for that path
  return Response.json({ ok: true });
}
```

```tsx
'use server';
// Server Action: read-your-writes — user must see their change immediately
export async function updateProfile(userId: string, data: Profile) {
  await db.users.update(userId, data);
  updateTag(`user-${userId}`);   // expires AND refreshes within the same request
}

// Server Action: refresh uncached data only (e.g., a notification count)
export async function markRead(id: string) {
  await db.notifications.markAsRead(id);
  refresh();                     // server-side sibling of router.refresh()
}
```

| Function | Semantics | Where |
|----------|-----------|-------|
| `revalidateTag(tag, profile)` | Stale-while-revalidate: serve cached, regenerate in background. In Next 16 the second `cacheLife` profile argument (e.g., `'max'`, `'hours'`, `{ expire: 3600 }`) is required; the single-argument form is deprecated. | Route Handlers + Server Actions |
| `updateTag(tag)` | Read-your-writes: expire and re-read fresh data in the same request — for forms/settings where the user expects to see their change. | Server Actions only |
| `refresh()` | Refresh *uncached* dynamic data without touching any cache. | Server Actions only |
| `revalidatePath('/p')` | Invalidates the Full Route Cache for `/p` (and, with options, nested paths and their Data Cache entries). | Route Handlers + Server Actions |

- Use **`revalidateTag`** when a logical entity changed and eventual consistency is fine (CMS publish, background sync).
- Use **`updateTag`** in the mutation path of interactive features — the classic "I saved but still see old data" bug is exactly what it exists to fix.
- Use **`revalidatePath`** when you know a specific URL's output changed.
- Use **time-based** when freshness has a natural TTL and on-demand plumbing isn't worth it.

---

## Route Handlers (the App Router API layer)

Route Handlers live in `app/api/.../route.ts` and export HTTP-method-named functions. They replace Pages Router `pages/api/*.ts`. They run on the server (Node.js runtime by default; Edge runtime opt-in) and can stream responses.

```tsx
// app/api/users/route.ts
import { NextRequest } from 'next/server';
import { db } from '@/lib/db';

export async function GET(req: NextRequest) {
  const users = await db.user.findMany();
  return Response.json(users);
}

export async function POST(req: NextRequest) {
  const body = await req.json();
  const user = await db.user.create({ data: body });
  return Response.json(user, { status: 201 });
}
```

### Dynamic segments

```tsx
// app/api/users/[id]/route.ts
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> } // params is async (mandatory since Next 16)
) {
  const { id } = await params;
  const user = await db.user.findUnique({ where: { id } });
  if (!user) return new Response('Not found', { status: 404 });
  return Response.json(user);
}
```

> `params` is a `Promise` — you must `await` it (async since Next 15; sync access removed in Next 16). Same for `searchParams` in pages.

### Streaming responses

Return a `ReadableStream` to stream data to the client (e.g., large datasets, SSE, AI token streams):

```tsx
export async function POST(req: Request) {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      for (const chunk of await generateChunks()) {
        controller.enqueue(encoder.encode(chunk));
      }
      controller.close();
    },
  });
  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' },
  });
}
```

### Caching Route Handlers (Next 15+)

GET Route Handlers are **not cached by default** since Next 15 (they were in 14; unchanged in 16). To cache a GET handler, opt in:

```tsx
export const dynamic = 'force-static';
export const revalidate = 3600;

export async function GET() {
  return Response.json(await getConfig());
}
```

Non-GET methods are never cached. Any use of `cookies()`/`headers()` makes the handler dynamic.

### Route Handlers vs Pages Router API routes

| | Pages Router `pages/api/*` | App Router `app/api/*/route.ts` |
|---|---|---|
| Signature | `(req, res)` (Express-like) | `(req, ctx)` returning a `Response` (Web API) |
| Caching | Manual | Opt-in via segment config |
| Streaming | Manual piped streams | Native `ReadableStream` |
| Default runtime | Node | Node (Edge opt-in) |

Use Route Handlers when you need an explicit HTTP endpoint (webhooks, third-party integrations, public APIs). For mutations triggered by your own UI, prefer Server Actions.

---

## Server Actions

Server Actions are async functions that run **only on the server** but can be called from Client Components (or from `<form action>`). They let you mutate data and revalidate caches **without writing an API endpoint**. Mark them with `'use server'` at the top of a file (making all exports actions) or above a function.

```tsx
// app/actions.ts
'use server';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { db } from '@/lib/db';
import { requireUser } from '@/lib/auth';

export async function createPost(formData: FormData) {
  const user = await requireUser();
  await db.post.create({
    data: { title: String(formData.get('title')), authorId: user.id },
  });
  revalidatePath('/posts');     // refresh the list
  redirect('/posts');           // navigate after mutation
}
```

### Used in a form (progressive enhancement)

Works **without JS enabled** — the form posts to the action endpoint Next generates:

```tsx
// app/posts/new/page.tsx
import { createPost } from '@/app/actions';

export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

### Called from a Client Component

```tsx
'use client';
import { createPost } from '@/app/actions';
import { useTransition } from 'react';

export function NewPostButton() {
  const [pending, start] = useTransition();
  return (
    <button
      disabled={pending}
      onClick={() => start(() => createPost(new FormData()))}
    >
      {pending ? 'Creating…' : 'Create'}
    </button>
  );
}
```

### When to use Server Actions

- Mutations triggered by your own UI (forms, button clicks) — no need to hand-roll a REST endpoint.
- Any time you want "call a server function, then revalidate the affected routes."
- Avoid for public/third-facing APIs (use Route Handlers) or for very large payloads.

### Validation and security

Server Actions are publicly callable endpoints (Next generates a URL). **Always validate inputs** and **authorize** — never trust `formData`. Treat them like any other API endpoint: authenticate, validate with a schema (e.g., Zod), and never expose secrets.

---

## Proxy (formerly Middleware)

**Next.js 16 renamed `middleware.ts` to `proxy.ts`** to make the network boundary explicit — `middleware.ts` still works for Edge use cases but is deprecated. `proxy.ts` at the project root (or `src/`) runs **before** a request is matched to a route, and — a real behavioral change — it runs on the **Node.js runtime**, not Edge. Use it for auth checks, redirects, i18n locale negotiation, header rewriting.

```tsx
// proxy.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export default function proxy(req: NextRequest) {
  if (!req.cookies.get('session') && req.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', req.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/private/:path*'],
};
```

Migration is mechanical: rename the file, rename the exported function to `proxy` (default export); the logic is unchanged.

### Constraints (know these for interviews)

- **Runtime:** `proxy.ts` runs on **Node.js** — a single predictable runtime, and Node APIs are available. The old `middleware.ts` ran on the Edge runtime (restricted Node subset, no native modules); if an interviewer asks "what runtime does middleware run on," the version-aware answer is "Edge through Next 15; Node via `proxy.ts` in 16."
- **Still keep it cheap.** Even though Node makes DB access *possible*, the proxy runs on every matched request and adds latency to all of them. Read a cookie/header, branch, redirect; do full DB-backed authorization in the Route Handler/Server Component/Action.
- **Cannot** set arbitrary response bodies (it's for request-time gating/rewriting, not rendering).
- The `matcher` config limits which paths run the proxy — use it to avoid running on static assets.
- Optimistic auth checks (verify a JWT signature, check cookie presence) belong here; treat it as a gate, not an auth system.

---

## Layouts, Templates, and Routing Primitives

### Root layout (`app/layout.tsx`)

The top-level layout wraps every route. Required to contain `<html>` and `<body>`. Persists across navigation (does not re-render) — making it ideal for global providers.

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body><Providers>{children}</Providers></body>
    </html>
  );
}
```

### Nested layouts

Each segment can have its own `layout.tsx`, which nests. Layouts **persist** across navigations between sibling routes (state is preserved).

### Templates (`template.tsx`)

Like a layout, but **re-mounts** on every navigation (a fresh state). Use when you explicitly want a fresh component instance per route (rarely needed; layouts are the norm).

### Route groups `(name)`

Folders wrapped in parentheses are **organizational only** — they don't appear in the URL. Use to group routes sharing a layout, or to give different parts of the site different root layouts.

```
app/
  (marketing)/layout.tsx   -> /
  (marketing)/about/page.tsx -> /about
  (app)/dashboard/page.tsx   -> /dashboard
```

### Parallel Routes `@slot`

Render multiple route segments in a single layout, each independently. Useful for dashboards with independent panels and for **intercepting routes** (modals).

```tsx
// app/layout.tsx
export default function Layout({
  children,
  analytics,
}: { children: React.ReactNode; analytics: React.ReactNode }) {
  return <main>{children}<aside>{analytics}</aside></main>;
}
```

### Intercepting Routes `(.)` `(..)` `(...)`

Intercept a navigation to show it in-context (e.g., a modal over the current page) while keeping a direct-visit full page. Powers the "click a photo → modal, but visit the URL directly → full photo page" pattern.

### Dynamic and catch-all segments

- `[id]` — dynamic segment.
- `[...slug]` — catch-all.
- `[[...slug]]` — optional catch-all (matches the root too).

---

## Metadata API

Export a static `metadata` object or an async `generateMetadata` function from any layout/page. Next.js merges metadata up the tree and injects `<head>` tags — good for SEO/OpenGraph.

```tsx
// app/products/[id]/page.tsx
export async function generateMetadata({
  params,
}: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await fetch(`https://api.example.com/products/${id}`).then((r) => r.json());
  return { title: product.name, description: product.summary };
}
```

---

## Passing Data from Server to Client Components

Server Components can fetch data and pass it to Client Components **as props**, but the data must be **serializable** (JSON-safe — plain objects, arrays, strings, numbers, booleans, null). You cannot pass functions, class instances, Dates (serialize to ISO string), or React elements (except via `children` slot).

```tsx
// app/page.tsx (Server Component)
import ClientChart from './ClientChart';

export default async function Page() {
  const data = await fetch('https://api.example.com/stats').then((r) => r.json());
  return <ClientChart points={data} />; // data must be JSON-serializable
}
```

```tsx
// ClientChart.tsx
'use client';
export default function ClientChart({ points }: { points: { x: number; y: number }[] }) {
  // ...use points with a charting lib
  return <canvas />;
}
```

If you need to pass a Server Component's rendered output into a Client Component, pass it as `children` (see the composability section in `01-rendering-strategies.md`).

---

## `params` and `searchParams` are async

Since Next 15, both are **async** (Promises) in pages and layouts, and Next 16 **removed synchronous access entirely** (along with sync `cookies()`/`headers()`/`draftMode()`). You must `await` them.

```tsx
// app/search/page.tsx
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ q?: string }>;
}) {
  const { q } = await searchParams;
  const results = await fetch(`https://api.example.com/search?q=${q}`).then((r) => r.json());
  return <Results items={results} />;
}
```

```tsx
// app/posts/[id]/page.tsx
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const post = await fetch(`https://api.example.com/posts/${id}`).then((r) => r.json());
  return <article>{post.body}</article>;
}
```

- Reading `searchParams` opts the route into **dynamic** rendering.
- In `generateMetadata`, `params`/`searchParams` are likewise async Promises.
- Route Handler `params` are also Promises (shown above).

This is a frequent interview gotcha: code written for Next 14 that accessed `params.id` synchronously warned on Next 15 and **breaks outright on Next 16**. The codemod (`npx @next/codemod@canary upgrade latest`) handles most call sites.

---

## Summary of when to use what

| Need | Use |
|------|-----|
| Read data for a Server Component | `fetch` in the component (or ORM + `"use cache"`) |
| Public HTTP endpoint / webhook | Route Handler (`route.ts`) |
| UI-triggered mutation + revalidate | Server Action (`'use server'`) |
| Per-request auth gate / redirect | `proxy.ts` |
| Invalidate cached data after a write (eventual) | `revalidateTag(tag, profile)` / `revalidatePath` |
| User must see their write immediately | `updateTag` in a Server Action |
| Refresh uncached dynamic data after an action | `refresh()` |
| Personalized route (per-user) | Dynamic route (via `cookies()`/`searchParams`) + streaming |
| Shared chrome across routes | Nested layouts |

Next, test yourself with [`03-interview-questions.md`](./03-interview-questions.md).
