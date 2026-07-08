# Rendering Strategies

Understanding where and when HTML is produced is the foundation of reasoning about Next.js on the server. This file walks through the classic rendering spectrum, then the App Router model (React Server Components), the server/client boundary, and the Next.js 15 caching-default changes that interviewers love to probe.

---

## The Rendering Spectrum

These terms predate the App Router but remain the vocabulary interviewers use. Know what each produces and when it's computed.

### Client-Side Rendering (CSR)

The server sends a near-empty HTML shell plus a JS bundle. The browser downloads JS, runs it, and the components render in the browser, typically fetching data after mount.

```tsx
'use client';
import { useEffect, useState } from 'react';

export default function Products() {
  const [items, setItems] = useState([]);
  useEffect(() => {
    fetch('/api/products').then((r) => r.json()).then(setItems);
  }, []);
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

- **Pros:** cheap server, rich interactivity, client-side routing after load.
- **Cons:** poor First Contentful Paint (FCP), weak SEO (crawlers see a shell unless they execute JS), no content until JS loads.
- **Use when:** authenticated dashboards, highly interactive tools where SEO/initial paint don't matter.

### Server-Side Rendering (SSR)

HTML is generated **on every request** on the server and sent ready to paint. In the Pages Router this was `getServerSideProps`. In the App Router, a route is dynamically rendered when it uses dynamic APIs (see *Static vs Dynamic* below) — there is no separate function, it's inferred.

- **Pros:** always-fresh content, good SEO, fast FCP.
- **Cons:** per-request server compute; the slowest data source blocks the whole page (unless streaming is used).
- **Use when:** personalized content, dashboards, anything that must reflect the latest request (auth, A/B variants).

### Static Site Generation (SSG)

HTML is generated **once at build time**. The output is plain files served from a CDN. In the Pages Router this was `getStaticProps`. In the App Router, a route is statically rendered by default if it doesn't touch dynamic APIs.

- **Pros:** fastest possible response, cheapest to serve (CDN), great SEO.
- **Cons:** content is frozen at build time; to refresh you rebuild or use ISR.
- **Use when:** marketing pages, docs, blogs — anything that changes rarely and is identical for all users.

### Incremental Static Regeneration (ISR)

Statically generated, but regenerated in the background after a revalidation period or on-demand. Combines SSG speed with freshness.

```tsx
// App Router: time-based ISR via the extended fetch
export const revalidate = 60; // seconds

export default async function Page() {
  const res = await fetch('https://api.example.com/articles', {
    next: { revalidate: 60 },
  });
  const data = await res.json();
  return <ArticleList items={data} />;
}
```

- A visitor gets the stale (cached) HTML instantly; Next.js regenerates in the background and the **next** visitor gets the fresh version (stale-while-revalidate semantics).
- On-demand ISR uses `revalidateTag` / `revalidatePath` (covered in `02-data-fetching-and-api-routes.md`).
- **Use when:** content changes occasionally (e.g., every few minutes) but you want CDN-speed delivery.

### Streaming SSR

Instead of buffering the entire HTML response, the server flushes chunks as they become ready. The browser can start painting before all data has resolved. React's Suspense component marks the boundaries of these streamed chunks.

```tsx
export default function Page() {
  return (
    <main>
      <Header />
      <Suspense fallback={<Skeleton />}>
        <SlowChart /> {/* streams in when ready */}
      </Suspense>
    </main>
  );
}
```

- Eliminates the "all-or-nothing" waterfall of classic SSR — a slow query no longer blocks the whole page.
- In the App Router, streaming is the default rendering mode for dynamic routes via `loading.tsx` and Suspense.

---

## The App Router Model: Server Components vs Client Components

The App Router (the `app/` directory) is built on **React Server Components (RSC)**. This is the most important conceptual shift from the Pages Router.

### Server Components are the default

Every component in `app/` is a **Server Component** unless you explicitly opt out. Server Components:

- Run **only on the server** (never shipped to the browser, never in the client bundle).
- Can be **async** and `await` data directly — no `useEffect`, no loading states needed for the data itself.
- Can read the filesystem, talk to databases, use secrets — safely, since the code never reaches the client.
- **Cannot** use React state (`useState`), effects (`useEffect`), browser APIs, or event handlers (`onClick`).

```tsx
// app/products/page.tsx — a Server Component (default)
import { db } from '@/lib/db';

export default async function Page() {
  const products = await db.product.findMany(); // direct DB access, no API needed
  return (
    <ul>
      {products.map((p) => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

### Client Components ('use client')

Add the `'use client'` directive at the very top of a file to make it (and its imported subtree) a Client Component. Client Components:

- Run on the server **for the initial HTML render** (they're still SSR'd), then **hydrate** in the browser.
- Can use hooks, state, effects, event handlers, browser APIs.
- Are bundled and shipped to the client.

```tsx
// app/components/Counter.tsx
'use client';
import { useState } from 'react';

export default function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>Count: {n}</button>;
}
```

### The `'use client'` directive is a boundary, not a declaration

`'use client'` marks the **top of a client subtree**. Once you're inside a client module, everything it imports is also part of the client bundle — you cannot "pop back" into server mode by importing a server component from a client component.

```tsx
// Client component — this is fine, importing other client components
'use client';
import Counter from './Counter';
```

### When to use each

| Need | Use |
|------|-----|
| Fetch data, read DB/FS, use secrets | Server Component |
| Static markup, no interactivity | Server Component |
| `useState`, `useEffect`, browser APIs | Client Component (`'use client'`) |
| `onClick`, `onChange`, form event handlers | Client Component |
| A heavy, non-interactive dependency (e.g., a markdown parser) | Server Component (keeps it out of client bundle) |
| An interactive chart library | Client Component |

A common, effective pattern: keep pages/layouts as Server Components for data, and drop into Client Components only for the interactive leaf nodes.

### Composability: passing server components as children

You **cannot import a Server Component directly into a Client Component** — the moment a module is client, its imports are client-bound, and a server component pulled in would lose its server-only guarantees. But you **can** pass a Server Component **as a child/prop** from a Server Component into a Client Component. This works because the server component is rendered to RSC payload on the server, and the client component receives the already-rendered result as a slot.

```tsx
// app/components/ClientWrapper.tsx
'use client';
export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return <div>{open && children}</div>; // children rendered on the server, toggled here
}

// app/page.tsx (Server Component)
import ClientWrapper from './components/ClientWrapper';
import ServerChart from './components/ServerChart';

export default function Page() {
  return (
    <ClientWrapper>
      <ServerChart /> {/* server component passed as children — allowed */}
    </ClientWrapper>
  );
}
```

This "children as slot" pattern is the canonical way to bridge interactivity (client) with server-rendered data (server) without breaking the model.

### The `'use server'` directive

Distinct from `'use client'`. `'use server'` marks **server-only functions** (Server Actions) callable from client code. Next.js generates a secure RPC endpoint for each such function. Covered in detail in `02-data-fetching-and-api-routes.md`.

```tsx
'use server';
import { revalidatePath } from 'next/cache';
import { db } from '@/lib/db';

export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: String(formData.get('title')) } });
  revalidatePath('/posts');
}
```

---

## Static vs Dynamic Rendering (App Router)

In the App Router you don't pick SSR vs SSG by calling a function — Next.js **infers** it per-route from whether you use *dynamic functions*.

A route is **static** (prerendered at build time) when it:
- Does not call `cookies()`, `headers()`, or read `searchParams`/`params` (in a way that opts into dynamic).
- Doesn't use `request`-based APIs.

A route becomes **dynamic** (rendered per-request) when it calls any of:

```tsx
import { cookies, headers } from 'next/headers';

export default async function Page() {
  const token = (await cookies()).get('session')?.value;   // -> dynamic
  const ua = (await headers()).get('user-agent');          // -> dynamic
  const sp = await searchParams;                            // -> dynamic (Next 15)
}
```

You can force the choice explicitly with route segment config:

```tsx
export const dynamic = 'force-static';   // or 'force-dynamic', 'error'
export const revalidate = 60;            // time-based ISR
```

> **Next 15 note:** `params` and `searchParams` are now **async** (Promises) in pages and layouts. You must `await` them. Reading them opts the route into dynamic rendering.

---

## Next.js 15 Caching Default Changes

This is one of the most common interview questions. Know the exact diff between 14 and 15.

| Behavior | Next.js 14 (App Router) | Next.js 15 |
|----------|-------------------------|------------|
| `fetch()` requests | Cached by default (`cache: 'force-cache'`) | **Not cached by default** (`cache: 'no-store'` semantics unless opt-in) |
| `GET` Route Handlers | Cached by default | **Not cached by default** |
| Non-`GET` Route Handlers | Never cached | Never cached (unchanged) |
| `cookies()`, `headers()`, `searchParams` | Opt route into dynamic | Opt route into dynamic (also now **async**) |
| Pages/layouts | Cached (Full Route Cache) when static | Static routes still cached; but you must opt into `fetch` caching explicitly |

**Why the change?** The Next 14 defaults surprised many developers — `fetch` silently returned stale data because caching was on by default. Next 15 makes the platform behave like the platform (the Web `fetch` default), so `fetch` is fresh unless you ask to cache. To opt back into caching in Next 15:

```tsx
// Opt into caching (Next 15)
const res = await fetch('https://api.example.com/data', {
  cache: 'force-cache',            // cache indefinitely
  next: { revalidate: 60 },        // or time-based ISR
  // next: { tags: ['collection'] } // or on-demand via revalidateTag
});
```

```tsx
// Route Handler caching opt-in (Next 15) — only GET works with this
export const dynamic = 'force-static';
export const revalidate = 60;

export async function GET() {
  return Response.json(await getData());
}
```

> Implication for backend interviews: when asked "how is Next's fetch different from the browser's," the honest Next 15 answer is "less different than it used to be — Next extends it with `next.revalidate` and `next.tags` for on-demand revalidation, but caching is now opt-in."

---

## Suspense and Streaming

In the App Router, streaming is built in. Two mechanisms:

1. **`loading.tsx`** — a special file whose content becomes the `Suspense` fallback for the route segment while it loads.

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return <Skeleton />;
}
```

2. **Explicit `<Suspense>`** — wrap an async child so only that subtree streams, letting the rest of the page paint immediately.

```tsx
import { Suspense } from 'react';

export default function Page() {
  return (
    <>
      <Nav /> {/* paints instantly */}
      <Suspense fallback={<p>Loading…</p>}>
        <Comments /> {/* async server component, streams in later */}
      </Suspense>
    </>
  );
}
```

The server emits the static shell first, then a streaming chunk for each Suspense boundary as its data resolves. This is the modern replacement for "the whole page blocks on the slowest query."

---

## Partial Prerendering (PPR) — Experimental

PPR combines static and dynamic rendering **within a single route**. The static parts are prerendered at build time and served from the CDN edge instantly; the dynamic parts (wrapped in Suspense) are streamed in at request time. This aims to give you "static speed for the shell, dynamic data for the personalized bits" without choosing one rendering mode for the whole route.

```tsx
import { Suspense } from 'react';
import { cookies } from 'next/headers';

export const experimental_ppr = true;

export default async function Page() {
  return (
    <>
      <StaticHero />                {/* prerendered at build */}
      <Suspense fallback={<Spinner />}>
        <PersonalizedGreeting />    {/* dynamic, streamed per request */}
      </Suspense>
    </>
  );
}

async function PersonalizedGreeting() {
  const c = await cookies();
  return <p>Hi, {c.get('user')?.value}</p>;
}
```

> **Status:** PPR is **experimental** in Next.js 15 (enable via `experimental.ppr` in `next.config.js` and the `experimental_ppr` segment option). Do not present it as stable in an interview — call it the direction Next.js is moving.

---

## SEO Implications of Rendering Choices

A practical point interviewers probe:

- **CSR:** crawlers that don't execute JS see an empty shell — poor SEO for public content.
- **SSR / Streaming:** fully-formed HTML on first request — good SEO, good social-card previews.
- **SSG / ISR:** fully-formed HTML, served from a CDN — best SEO and fastest TTFB.
- **Server Components:** render to HTML on the server, so they are SEO-friendly even though the component code never ships to the client.

Rule of thumb: any page that must be indexed or share-previewed should render its contentful HTML on the server (SSR/SSG/ISR). Pure CSR is fine for authenticated app surfaces.

---

## Putting it together: choosing a strategy

| Scenario | Strategy |
|----------|----------|
| Marketing/docs page, rarely changes | SSG (static by default in App Router) |
| Blog/news, updates every few minutes | ISR (`next: { revalidate }` or on-demand `revalidateTag`) |
| Per-user dashboard, always fresh | Dynamic (uses `cookies()`/`searchParams`), stream with Suspense |
| Heavy slow query + fast shell | Streaming SSR with `<Suspense>` boundaries |
| Pure interactive tool behind auth | Client Component subtree |

Next, see [`02-data-fetching-and-api-routes.md`](./02-data-fetching-and-api-routes.md) for how data reaches these components and how to expose your own endpoints.
