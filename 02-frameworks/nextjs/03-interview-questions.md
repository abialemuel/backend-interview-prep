# Next.js Interview Questions

Attempt each question out loud before reading the model answer. Answers reflect **Next.js 15** (App Router, React 19). Where Next 14 differed, it's called out.

---

## Easy

### Q1: What is the difference between CSR, SSR, SSG, and ISR?

**Answer:** These describe where and when HTML is produced. In **CSR** (Client-Side Rendering) the server ships a bare HTML shell plus a JS bundle, and the browser renders after executing JS — fast for the server but poor initial paint and weak SEO. **SSR** (Server-Side Rendering) generates full HTML on the server on every request, giving good SEO and fresh content but at per-request compute cost. **SSG** (Static Site Generation) renders HTML once at build time and serves it from a CDN — the fastest and cheapest, but content is frozen until the next build. **ISR** (Incremental Static Regeneration) is SSG that regenerates in the background on a time interval or on-demand, giving CDN speed with eventual freshness. In the App Router, SSR vs SSG vs ISR is largely inferred from whether you use dynamic APIs and how you configure `fetch`, rather than chosen via separate functions.

### Q2: What is a React Server Component and how does it differ from a Client Component?

**Answer:** A Server Component runs **only on the server** — its code is never shipped to the browser, it can be `async` and `await` data directly, and it can safely touch the database or filesystem. A Client Component is opted into with the `'use client'` directive; it renders on the server for the initial HTML but then **hydrates** in the browser and can use hooks, effects, event handlers, and browser APIs. In the App Router, **Server Components are the default** — you opt into client only where you need interactivity. The practical rule: keep pages/layouts as Server Components for data, and drop into Client Components only for interactive leaf nodes.

### Q3: Can you import a Server Component into a Client Component?

**Answer:** No — not directly. The moment a module has `'use client'`, everything it imports becomes part of the client bundle, so importing a Server Component would defeat its server-only nature (it would lose the ability to stay out of the bundle and use server-only APIs). The idiomatic workaround is to **pass the Server Component as `children` (or another prop) from a Server Component into a Client Component**. The server renders the Server Component to the RSC payload on the server, and the Client Component receives the already-rendered result as a slot — so interactivity lives in the client wrapper while data-heavy rendering stays on the server.

### Q4: When does a route become dynamic in the App Router?

**Answer:** A route is dynamic (rendered per-request) when it uses any **dynamic function**: `cookies()` or `headers()` from `next/headers`, or when it reads `searchParams`/`params` in a way that opts in. Otherwise, with static data, the route is **static** (prerendered at build). You can force the choice with route segment config: `export const dynamic = 'force-dynamic' | 'force-static' | 'error'`. Note that `cookies()`/`headers()`/`searchParams`/`params` are **async** in Next 15, and merely awaiting them makes the route dynamic.

### Q5: What are the SEO implications of choosing CSR versus SSR/SSG?

**Answer:** With pure CSR, crawlers that don't execute JavaScript see an empty shell, so public content is poorly indexed and social-card previews are blank. SSR and Streaming produce fully-formed HTML on the first request, so crawlers and link-preview fetchers see real content — good SEO. SSG and ISR are even better: fully-formed HTML served from a CDN edge, with the fastest TTFB. As a rule, any page that must be indexed or share-previewed should render its contentful HTML on the server; pure CSR is fine for authenticated app surfaces where SEO is irrelevant.

### Q6: What is `loading.tsx` and how does it relate to Suspense?

**Answer:** `loading.tsx` is a special file whose content becomes the `Suspense` fallback for the route segment while it (and its async children) load. Under the hood Next wraps the segment in a `<Suspense>` boundary, so the static shell streams immediately and the async content streams in when ready. You can also place explicit `<Suspense>` boundaries around individual async subtrees to stream them independently, letting the rest of the page paint without waiting. This is the App Router's streaming-SSR mechanism — a slow query no longer blocks the whole page.

### Q7: Why would you use a Server Action instead of a Route Handler?

**Answer:** Use a **Server Action** when a mutation is triggered by your own UI (a form submission or button click) and you want to "call a server function, then revalidate affected routes" without hand-writing an HTTP endpoint. Use a **Route Handler** when you need a real public HTTP endpoint — webhooks, third-party API integrations, or an API consumed by non-UI clients. Server Actions give you progressive enhancement (a plain `<form action={fn}>` works even without JS) and integrate tightly with `revalidatePath`/`revalidateTag`, but they're not a replacement for a documented REST/RPC API surface.

---

## Medium

### Q8: How did Next.js 15 change the default caching behavior compared to Next 14?

**Answer:** In Next 14, `fetch()` requests were **cached by default** (behaving like `cache: 'force-cache'`) and `GET` Route Handlers were cached by default too — which frequently surprised developers with stale data. In Next 15, **both are not cached by default**: `fetch` behaves like the Web standard (fresh unless you opt in), and `GET` Route Handlers are dynamic unless configured otherwise. To opt back into caching you pass `cache: 'force-cache'`, `next: { revalidate }`, or `next: { tags }` to `fetch`, or set `export const dynamic = 'force-static'` plus `revalidate` on a Route Handler. This is one of the most commonly tested behavioral changes.

```tsx
// Next 15: opt into caching explicitly
const res = await fetch(url, { next: { revalidate: 60 } });
```

### Q9: What are the four caching layers in Next.js, and how do they differ?

**Answer:** (1) **Request Memoization** — within a single server render pass, identical `fetch` calls are deduplicated in-memory; it's per-request and not shared. (2) **Data Cache** — stores `fetch` results **across requests** keyed by URL+options, persists across deploys (on Vercel), and is governed by `next.revalidate`/`next.tags`/`cache`. (3) **Full Route Cache** — caches the fully-rendered HTML + RSC payload of **static** routes; dynamic routes are never cached here. (4) **Router Cache** — a client-side, in-memory cache of RSC payloads for navigated routes, powering instant client-side navigation and back/forward. They cascade: invalidating a Data Cache entry (e.g., `revalidateTag`) can invalidate dependent Full Route Cache entries, and a Server Action's response tells the client to drop affected Router Cache entries.

### Q10: Compare `revalidateTag`, `revalidatePath`, and time-based revalidation.

**Answer:** **Time-based** (`next: { revalidate: 60 }` or `export const revalidate = 60`) revalidates after N seconds with stale-while-revalidate semantics — the current value is served and a background regeneration happens; the *next* request gets the fresh value. **`revalidateTag('articles')`** busts every `fetch` (Data Cache) tagged `'articles'`, plus the static routes that depend on them — content-aware and coarse. **`revalidatePath('/articles')`** busts the Full Route Cache for a specific path (and, with options, nested paths and their Data Cache entries) — useful when you know a specific URL's output changed. Use time-based when freshness has a natural TTL; use on-demand (`revalidateTag`/`revalidatePath`) when you know exactly when content changed (e.g., after a CMS publish or a mutation).

### Q11: How do App Router Route Handlers differ from Pages Router API routes?

**Answer:** Pages Router API routes (`pages/api/*.ts`) use an Express-like `(req, res)` signature and were never automatically cached. App Router Route Handlers (`app/api/*/route.ts`) export per-method functions (`GET`, `POST`, etc.) that take a `NextRequest` and return a Web `Response`. In Next 14 GET handlers were cached by default; in Next 15 they are **not cached by default** (opt in with segment config). Route Handlers support native `ReadableStream` streaming and can run on the Edge runtime. They replace Pages Router API routes for new code, but for mutations triggered by your own UI you'd usually prefer Server Actions over either.

```tsx
// app/api/items/route.ts
export async function GET() {
  return Response.json(await getItems());
}
```

### Q12: What can and can't middleware do, and what runtime does it run on?

**Answer:** `middleware.ts` runs **before** route matching and is intended for cheap request-time gating: auth checks, redirects, i18n locale negotiation, header rewriting. It runs on the **Edge Runtime** by default (a restricted Node subset — no native modules, limited APIs), so it can't do heavy work or query the database directly, and it shouldn't, because it runs on every matched request and adds latency to all of them. It can't render response bodies — it returns `NextResponse.next()`, `redirect()`, or `rewrite()`. Use the `matcher` config to limit which paths invoke it (avoid running it on static assets). For auth, check a JWT/cookie in middleware but do the full DB-backed authorization in the Route Handler/Server Component/Action.

### Q13: How do nested layouts work, and how do they differ from templates?

**Answer:** Each segment in `app/` can have its own `layout.tsx`, and layouts **nest** — a child route is wrapped by the layouts of all its parent segments up to the root layout (`app/layout.tsx`, which must render `<html>`/`<body>`). Crucially, layouts **persist across navigations** between sibling routes: their state is preserved and they don't re-render, which makes them ideal for global providers and shared chrome. A `template.tsx` is structurally similar but **re-mounts** on every navigation, giving a fresh component instance and fresh state — useful only when you explicitly want per-route reset. Route groups (`(name)`) let you group routes sharing a layout without affecting the URL.

### Q14: How do you pass data from a Server Component to a Client Component?

**Answer:** Pass it as a **prop**, and the value must be **JSON-serializable** — plain objects, arrays, strings, numbers, booleans, null. You cannot pass functions, class instances, `Date` objects (serialize to an ISO string first), or arbitrary React elements. If you need to pass a Server Component's *rendered output* into a Client Component, pass it as the `children` prop (the slot pattern) rather than trying to import the server component. This serialization constraint is enforced because the value crosses the server→client boundary via the RSC payload.

```tsx
// Server Component
import ClientChart from './ClientChart';
export default async function Page() {
  const points = await fetch(url).then((r) => r.json()); // JSON-safe
  return <ClientChart points={points} />;
}
```

### Q15: How did `params` and `searchParams` change in Next.js 15?

**Answer:** In Next 15, both `params` (in pages, layouts, Route Handlers, and `generateMetadata`) and `searchParams` (in pages and `generateMetadata`) became **async — they're Promises you must `await`**. In Next 14 you accessed them synchronously (`params.id`); code written that way will warn or break on Next 15. Reading `searchParams` opts the route into dynamic rendering. The async-ness reflects that these come from the request, which in the streaming/RSC model is resolved asynchronously.

```tsx
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  // ...
}
```

### Q16: Why does Next.js extend the standard `fetch` API instead of using a separate data function?

**Answer:** Next extends `fetch` with a `next` option (`revalidate`, `tags`) and per-request `cache` controls so that caching/revalidation can be expressed **per call site** and integrated with the on-demand invalidation system (`revalidateTag`). A separate data function (like the old `getStaticProps`) couldn't express "tag this entry so I can bust it later" at the call site, and would force all data fetching to a single file. By extending the familiar Web `fetch`, the same API works in Server Components, Route Handlers, and Server Actions, and the cache is controlled inline. Notably, in Next 15 the *default* is closer to the browser's (no caching), so the extension is mostly about the `next` options for opt-in caching/invalidation.

### Q17: How do you stream a response from a Route Handler?

**Answer:** Return a `Response` whose body is a `ReadableStream`. You construct the stream with a `start(controller)` function that `enqueue`s encoded chunks and `close`s when done, set the appropriate `Content-Type` (e.g., `text/event-stream` for SSE), and return it. The browser receives chunks as they're produced — useful for large datasets, server-sent events, or AI token streams. This is the App Router's native streaming primitive, replacing manual piped streams from the Pages Router.

```tsx
export async function POST() {
  const enc = new TextEncoder();
  const stream = new ReadableStream({
    async start(c) {
      for (const chunk of await generate()) c.enqueue(enc.encode(chunk));
      c.close();
    },
  });
  return new Response(stream, { headers: { 'Content-Type': 'text/event-stream' } });
}
```

---

## Hard

### Q18: Explain the "children as slot" pattern for composing Server and Client Components. Why can't you just import the server component?

**Answer:** You can't import a Server Component into a Client Component because `'use client'` marks the **top of a client subtree** — everything imported from there is bundled for the browser, so a server component pulled in would lose its server-only guarantees (it couldn't stay out of the bundle or use server-only APIs like the DB). The pattern is to define the Client Component to accept `children: React.ReactNode`, and have a **Server Component** parent render the Server Component and pass it as `children` into the Client Component. The server renders the Server Component to the RSC payload on the server, and the Client Component receives the already-rendered result as a slot — so the interactive toggle lives in the client wrapper while the data-heavy rendering stays server-side. This is the canonical bridge between interactivity and server-rendered data.

```tsx
// ClientWrapper.tsx
'use client';
export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return <div>{open && children}</div>;
}
// page.tsx (Server)
<ClientWrapper><ServerChart /></ClientWrapper>;
```

### Q19: Walk through how the four caching layers interact during a request to a static, ISR-cached route, and after an on-demand revalidation.

**Answer:** At build time, the Server Component renders and its `fetch` calls populate the **Data Cache** (tagged/`revalidate`d per their options); the fully-rendered HTML + RSC payload is stored in the **Full Route Cache**. On a request, the CDN serves the prerendered HTML instantly; on a client-side navigation, the **Router Cache** serves the RSC payload without a round-trip. Within that original render, any duplicate `fetch` calls were deduplicated by **Request Memoization** (per-request only). Now suppose a Server Action runs `revalidateTag('articles')`: it invalidates the **Data Cache** entries tagged `'articles'`, which cascades to invalidate the **Full Route Cache** for routes that depended on them; the Server Action's response also tells the browser to drop the affected **Router Cache** entries, so the next navigation refetches a fresh RSC payload. Request Memoization is unaffected (it's per-request and ephemeral).

### Q20: What is Partial Prerendering (PPR), and how does it differ from choosing static or dynamic for a whole route?

**Answer:** PPR (experimental in Next 15) lets a **single route** be partly static and partly dynamic. The static portions — anything outside a Suspense boundary that doesn't use dynamic APIs — are prerendered at build time and served from the CDN edge instantly; the dynamic portions (wrapped in `<Suspense>`, using `cookies()`/`searchParams`, etc.) are streamed in at request time. Without PPR, a route is either fully static or fully dynamic — using `cookies()` anywhere forces the entire route dynamic, even the parts that don't depend on the request. PPR aims to give "static speed for the shell, dynamic data for the personalized bits" without compromising freshness. Enable via `experimental.ppr` in `next.config.js` and `export const experimental_ppr = true` per route. Call it experimental/directional in an interview, not stable.

```tsx
export const experimental_ppr = true;
export default async function Page() {
  return (
    <>
      <StaticHero />
      <Suspense fallback={<Spinner />}><PersonalizedGreeting /></Suspense>
    </>
  );
}
```

### Q21: Design a mutation flow using Server Actions that creates a record and refreshes the UI. What are the security concerns?

**Answer:** Write a `'use server'` function that authenticates the user, validates the input with a schema (e.g., Zod), performs the DB write, calls `revalidatePath`/`revalidateTag` to bust the affected caches, and optionally `redirect`s. Wire it to a `<form action={fn}>` for progressive enhancement, or call it from a Client Component inside `useTransition` for a pending state. Security concerns: Server Actions are **publicly callable endpoints** (Next generates a URL for each), so you must **authenticate and authorize every call** and **never trust `formData`** — validate strictly. Don't expose secrets or pass them through the client, and be mindful that a malicious client can craft arbitrary `FormData`. Treat a Server Action with the same scrutiny as a Route Handler.

```tsx
'use server';
export async function createPost(fd: FormData) {
  const user = await requireUser();
  const title = z.string().min(1).parse(fd.get('title'));
  await db.post.create({ data: { title, authorId: user.id } });
  revalidatePath('/posts');
  redirect('/posts');
}
```

### Q22: A page mixes a fast header, a personalized greeting, and a slow analytics chart. How would you render it for the best perceived performance?

**Answer:** Use streaming SSR with explicit `<Suspense>` boundaries. Render the header (static, or fast server data) immediately so the shell paints; wrap the personalized greeting (dynamic, uses `cookies()`) in its own `<Suspense>` so it streams in after the request-specific read; wrap the slow analytics chart (async Server Component fetching a slow endpoint) in another `<Suspense>` with a skeleton fallback. Because each boundary streams independently, the slow chart doesn't block the greeting or the shell. If the static parts are truly static and the personalized part is small, PPR could even prerender the shell at build and stream only the dynamic bits. Avoid making the whole page a single dynamic fetch — that recreates the classic SSR "block on the slowest query" problem.

### Q23: Compare the Edge and Node runtimes in Next.js. When does the choice matter?

**Answer:** **Edge** runtime (V8-based, no full Node) is cold-start-free, globally distributed, and ideal for lightweight, low-latency code — middleware runs here by default, and you can opt Route Handlers onto it. It's great for auth gating, redirects, geo-based logic, and simple responses, but it lacks many Node APIs and can't use native modules or heavy libraries. **Node** runtime (the default for Route Handlers and Server Components/Actions) has the full Node API surface, can use any npm package and the filesystem/database directly, but has cold starts and isn't as globally distributed. The choice matters most for Route Handlers and middleware: keep middleware on Edge and cheap; put heavy/database work on Node. Don't try to run an ORM query in Edge middleware.

### Q24: How does the Router Cache affect the user experience, and how do you invalidate it after a mutation?

**Answer:** The Router Cache is a client-side, in-memory store of RSC payloads for navigated routes, which makes client-side `next/link` navigation and back/forward **instant** (no server round-trip for already-visited routes) and powers stale-while-revalidate navigation. It has time limits (default ~5 min for static, ~30s for dynamic). After a mutation, the cleanest invalidation is to run `revalidatePath`/`revalidateTag` in a Server Action — the action's response tells the client to drop the affected Router Cache entries, so the next navigation refetches fresh RSC payload. You can also call `router.refresh()` from a Client Component to force a refetch of the current route's RSC payload. Without invalidation, a user who mutated data might still see the stale cached view on subsequent navigations within the cache TTL.

### Q25: When would you reach for `unstable_cache` versus the extended `fetch`?

**Answer:** Use the extended `fetch` whenever your data source is an HTTP endpoint — it integrates directly with the Data Cache, `next.revalidate`/`next.tags`, and on-demand revalidation, and request memoization dedupes it within a render. Reach for `unstable_cache` when your data comes from a **non-`fetch` source** — a direct ORM/database query, a third-party SDK, or an in-process computation — because those don't flow through the extended `fetch` and so wouldn't be cached otherwise. `unstable_cache` wraps such a function with an explicit cache key and the same `revalidate`/`tags` options, opting it into the Data Cache. The name signals it's not yet a frozen API; for many backend workloads (Prisma, Drizzle, etc.) it's the right tool to get ISR-style caching on DB queries.

```tsx
const getProducts = unstable_cache(
  async () => db.product.findMany(),
  ['products'],
  { revalidate: 60, tags: ['products'] }
);
```
