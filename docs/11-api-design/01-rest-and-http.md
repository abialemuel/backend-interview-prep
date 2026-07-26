# REST and HTTP

Most API design interviews are, in practice, REST interviews. "REST" in industry usage means JSON resources over HTTP with sensible use of methods and status codes — not Fielding's full dissertation. This file covers the decisions you will actually be graded on: how to model resources, how to use HTTP semantics correctly, and the recurring contract questions (pagination, errors, versioning) where there are well-established right answers.

## Resource modeling

The single highest-leverage step. Identify the **nouns** in the domain and expose them as resources with stable identifiers; the verbs come for free from HTTP methods. A payments domain has `payments`, `refunds`, `customers`, `payment_methods` — not `/processPayment` or `/doRefund`.

Rules of thumb that hold up:

- **Plural nouns for collections**, identifier for individual items: `/orders`, `/orders/ord_8f3k2`. Consistency matters more than the plural/singular choice itself, but plural is the industry default.
- **Nest only for genuine ownership, and only one level.** `/customers/{id}/payment_methods` is fine because a payment method cannot exist without a customer. `/customers/{id}/orders/{oid}/items/{iid}/discounts` is not — deep nesting bakes the object graph into every URL and breaks the moment an item can be reached another way. If a resource has its own identity and lifecycle, give it a top-level collection and filter instead: `/orders?customer_id=cus_123`.
- **Prefixed, opaque IDs** (`cus_123`, `pi_8f3k2` — the Stripe convention) are worth mentioning: they make IDs self-describing in logs and support tickets, prevent clients from doing arithmetic on them, and let you change the underlying storage (auto-increment to UUID) without a contract change. Never expose raw auto-increment IDs — they leak business volume and invite enumeration attacks.
- **Actions that are not CRUD** are the classic interview trap. Three respectable options: model the action as a sub-resource POST (`POST /orders/{id}/cancellation` — creates a cancellation), as a state transition via the parent (`POST /payments/{id}/capture` — Stripe's approach, pragmatic and common), or as a first-class resource when the action has its own lifecycle (`POST /refunds` with a `payment_id` in the body, because refunds are queried, listed, and have states of their own). Avoid `PATCH {"status": "cancelled"}` for anything with side effects — a status field update should not silently trigger money movement.
- **Long-running operations**: return `202 Accepted` with a pointer to an operation resource the client can poll (`GET /operations/{id}` returning `{status: "running" | "succeeded" | "failed", result: ...}`), or accept a callback/webhook URL. Never hold an HTTP connection open for minutes.

## HTTP methods done right

| Method | Semantics | Safe | Idempotent | Typical use |
| --- | --- | --- | --- | --- |
| GET | Read, no side effects | Yes | Yes | Fetch resource or collection |
| POST | Create / process | No | **No** | Create resource, trigger action |
| PUT | Full replace at a known URI | No | Yes | Client-supplied ID, full update |
| PATCH | Partial update | No | No (by default) | Update a subset of fields |
| DELETE | Remove | No | Yes | Delete or soft-delete |

The safe/idempotent columns are not trivia — they are what proxies, retry middleware, and browsers rely on. A GET with side effects will eventually get replayed by a prefetcher or retried by an intermediary, and you will spend a weekend figuring out why. POST is the one non-idempotent method, which is exactly why idempotency keys exist (covered in `03-api-operations.md`).

PUT vs PATCH: PUT replaces the entire resource (fields you omit are cleared), PATCH modifies only what you send. In practice most APIs implement "PATCH with merge semantics" (RFC 7396 JSON Merge Patch: send the fields to change, `null` to clear a field) and skip PUT entirely. If asked, know that JSON Patch (RFC 6902, an operations array) exists for surgical edits but is rarely worth the complexity.

## Status codes that matter

You do not need all ~60. You need to use about a dozen precisely and never lie with them:

| Code | Meaning | When |
| --- | --- | --- |
| 200 | OK | Successful GET/PATCH/PUT; POST that performs an action |
| 201 | Created | POST created a resource; include `Location` header and the resource in the body |
| 202 | Accepted | Work queued, not done; body points to an operation to poll |
| 204 | No Content | Successful DELETE, or update with nothing to return |
| 304 | Not Modified | Conditional GET, cache still valid (no body) |
| 400 | Bad Request | Malformed syntax, invalid field values |
| 401 | Unauthorized | Missing or invalid credentials (really "unauthenticated") |
| 403 | Forbidden | Authenticated, but not allowed |
| 404 | Not Found | No such resource — also the correct answer for "exists but you may not know it exists" (tenant isolation) |
| 409 | Conflict | Version conflict, duplicate creation, state precludes the operation |
| 422 | Unprocessable | Syntactically valid, semantically wrong (validation errors) — 400 vs 422 split is a convention choice; pick one and be consistent |
| 429 | Too Many Requests | Rate limited; include `Retry-After` |
| 500 | Internal error | Your bug; never for client errors |
| 503 | Unavailable | Overload/maintenance; include `Retry-After` |

Two lies interviewers listen for: returning 200 with `{"error": ...}` in the body (breaks every HTTP-aware retry, cache, and monitoring layer — your error rate dashboard reads zero while everything is on fire), and returning 500 for validation failures (pages the on-call for the client's typo). The 4xx/5xx split is the contract that separates "your fault, do not retry blindly" from "our fault, retry with backoff."

## Caching, ETags, and conditional requests

HTTP has a built-in caching model most APIs ignore, and using it is cheap senior-level signal.

- **`Cache-Control`** on responses: `private, max-age=60` for per-user data cacheable briefly by the client; `public, max-age=300, stale-while-revalidate=60` for shared data behind a CDN; `no-store` for sensitive payloads. An API that sets nothing gets heuristic caching behavior it did not choose.
- **`ETag`** is a version fingerprint of the representation (a hash, or better, a version number you already store). Client sends it back as `If-None-Match`; if unchanged you return `304` with no body. On mobile clients polling a 50 KB resource, this is a real bandwidth and latency win for one header's worth of work.
- **Conditional writes** are the bigger prize: `If-Match` turns the ETag into optimistic concurrency control. Client GETs the resource (ETag `"v7"`), edits it, sends `PATCH ... If-Match: "v7"`. If someone else wrote in between, you return `412 Precondition Failed` and the client re-fetches and re-applies. This is the HTTP-native answer to "two admins edit the same record" — the same `UPDATE ... WHERE version = ?` pattern you would use in MySQL, expressed in the contract.

```http
GET /articles/42
ETag: "v7"

PATCH /articles/42
If-Match: "v7"
{"title": "New title"}

412 Precondition Failed   <- someone else wrote v8 first; re-fetch and retry
```

## Pagination: offset vs cursor

Every collection endpoint needs pagination from day one — retrofitting it is a breaking change. The choice is the most common "there is a right answer" question in the round.

**Offset pagination** (`?page=3&per_page=50`, i.e. `LIMIT 50 OFFSET 100`):

- Simple, supports "jump to page 7" and "page 3 of 41".
- **Skew under concurrent writes**: a row inserted at the top while you are on page 2 shifts everything — page 3 shows a duplicate or silently skips a row. For anything feed-like or being consumed by a syncing client, this is a correctness bug.
- **Performance degrades linearly**: `OFFSET 100000` makes MySQL walk and discard 100k rows before returning 50. Deep pagination is a classic slow-query source.

**Cursor (keyset) pagination**: the response includes an opaque cursor encoding the position of the last item; the next request says `?cursor=eyJpZCI6ODQ3fQ&limit=50` and the server queries `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 50` — an index seek, constant cost at any depth, stable under inserts.

```json
{
  "data": [ ... ],
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNS0xMFQxMjowMDowMFoiLCJpZCI6ODQ3fQ",
  "has_more": true
}
```

Rules: the cursor must be **opaque** (base64 of the keyset, ideally signed) so clients cannot construct or parse it — otherwise its internals become part of your contract. Sort keys must be unique in combination (always append `id` as a tiebreaker to a timestamp sort). The cost: no random access, no total count without a separate (expensive, approximate) query.

**Default answer**: cursor for anything unbounded, user-generated, or consumed by machines; offset acceptable for small, slowly-changing, human-browsed admin lists where "page 7 of 12" is a genuine requirement. Cap `limit` (e.g., max 100) regardless.

## Filtering and sorting

Keep them as query parameters with a documented, bounded grammar — do not accidentally ship a query language.

```http
GET /orders?status=shipped&created_after=2026-01-01T00:00:00Z&sort=-created_at&limit=50
```

- Simple equality filters as plain params (`status=shipped`); ranges as suffixed params (`created_after`, `amount_gte`) or bracket syntax (`amount[gte]=1000`) — either is fine, consistency wins.
- Sorting via a `sort` param, `-` prefix for descending, comma for multi-key: `sort=-created_at,id`. **Whitelist sortable and filterable fields** — every one you allow is a promise there is an index behind it; an open-ended `sort=any_column` is a self-inflicted denial of service.
- If clients genuinely need arbitrary boolean logic, that is a search endpoint (`POST /orders/search` with a structured body) — a different contract with different performance promises. Do not grow it organically out of query params.

### Sparse fieldsets and expansion

Two related response-shaping tools worth naming before someone says "just use GraphQL":

- **Sparse fieldsets**: `GET /orders?fields=id,status,total` returns only those fields. Useful when a resource is heavy (a large description blob) and list views need a fraction of it. Keep the grammar flat — no nested field selection — or you are reimplementing GraphQL badly.
- **Expansion**: `GET /orders/42?expand=customer,items.product` inlines related resources that would otherwise be ID references, saving the client N follow-up calls. Stripe's `expand` is the reference implementation. Bound the depth (one or two levels) and document which fields are expandable — each one is a join you are promising to serve at list-endpoint volume.

Both are additive, opt-in, and cache-friendly (the URL is the cache key), which is why they are the REST-native answer to moderate over-fetching complaints. When the requirements outgrow them — many clients, deep graphs, per-screen shapes — that is the honest threshold for a BFF or GraphQL, covered in the next file.

## Error format: RFC 9457 problem details

RFC 9457 (which obsoleted RFC 7807) standardizes the error body as `application/problem+json`, and it is the answer to give when asked "what does your error response look like":

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 402,
  "detail": "Account acc_9k2 has a balance of 4.50 USD; the charge requires 20.00 USD.",
  "instance": "/payments/pay_8f3k2",
  "balance": "4.50",
  "request_id": "req_5m1xq"
}
```

- `type` is a URI identifying the error **category** — this, not the human-readable strings, is what clients branch on. Machine-readable, stable, documented.
- `title` is a short, stable summary of the category; `detail` is the human-readable specifics of this occurrence. Clients must not parse `detail`.
- Extension members are allowed and encouraged (`balance`, `request_id`). For validation failures, add an `errors` array with per-field entries: `[{"field": "email", "code": "invalid_format", "message": ...}]`.
- Always include a **request/correlation ID** so a support ticket can be joined to your logs.
- Never leak internals: stack traces, SQL, hostnames, or "user with email x@y.com not found" (an account-enumeration oracle) do not belong in error bodies.

The deeper principle: an error response is API surface. Clients will build retry logic and UX on it, so it needs the same stability guarantees as your success responses.

## Versioning strategies

| Strategy | Example | Pros | Cons |
| --- | --- | --- | --- |
| URL path | `/v1/orders` | Explicit, visible, trivially routable/cacheable | "Whole API" versions; v2 implies migrating everything |
| Header | `Accept: application/vnd.example.v2+json` | URLs stay stable, per-resource granularity possible | Invisible in browser/logs, harder to demo and debug |
| Query param | `/orders?version=2` | Easy to try | Mixes versioning into filtering; caching ambiguity |
| Date-based | `Stripe-Version: 2026-03-14` (header, pinned per API key) | Granular, continuous evolution; no big-bang v2 | Requires serious infrastructure: version gates in code, per-key pinning, tooling |

The strategy question is less important than the **policy** question, and strong candidates say so: versioning is your escape hatch of last resort, not your evolution mechanism. Additive changes (new optional fields, new endpoints, new enum values where documented as open) should never require a version bump; clients must be told to ignore unknown fields. You version only for genuinely breaking changes — and each live version is a codebase you maintain, test, and eventually have to sunset (see the deprecation policy in `03-api-operations.md`).

Pragmatic default: URL-path `v1` from day one (so the slot exists), a written definition of what counts as breaking, and the ambition to never need `v2`. Mention Stripe's date-pinning as the gold standard for API-as-product companies, while being honest that it is expensive to operate.

## HATEOAS pragmatism

HATEOAS — responses carry hypermedia links that tell the client what it can do next — is the most-asked "do people actually use this?" question. The honest answer: full HATEOAS (clients discovering all capabilities at runtime, no out-of-band docs) lost. Real clients are written against documentation and OpenAPI, not by following links, and generic hypermedia clients never materialized.

What survived is worth keeping:

```json
{
  "id": "pay_8f3k2",
  "status": "requires_capture",
  "links": {
    "self": "/payments/pay_8f3k2",
    "capture": "/payments/pay_8f3k2/capture",
    "refunds": "/payments/pay_8f3k2/refunds"
  }
}
```

- `next`/`prev` links in pagination (universally adopted).
- Links to related resources save clients from URL-construction — the server can restructure URLs without breaking anyone.
- **Affordance links as state signals**: including `capture` only when the payment is actually capturable moves business rules ("when can I capture?") from client code into the response. GitHub's API does this well.

Interview framing: "Level 2 of the Richardson maturity model — resources, methods, status codes — plus pagination links and selective related-resource links. Full HATEOAS is not worth the cost for APIs consumed by known clients."

## OpenAPI

OpenAPI (3.1, which aligned its schema dialect with JSON Schema) is the de facto contract format for REST, and the interview question is really about workflow:

- **Design-first** (write the spec, review it, then implement) vs **code-first** (generate the spec from annotations/code). Design-first front-loads the contract review — the cheapest moment to catch a bad design is before it is implemented — and enables parallel client/server work against a mock. Code-first guarantees spec/implementation agreement but tends to ship whatever the code happened to do. Platform and public-API teams generally go design-first; either way, the spec must live in CI, not in a wiki.
- What the spec buys you: generated clients and server stubs, request/response validation middleware (the spec enforced at runtime, not just documented), interactive docs, mock servers, **linting for consistency** (Spectral rules encoding your style guide: naming, required error shapes, pagination params — this is how a fifty-team org keeps APIs uniform), and **breaking-change detection in CI** by diffing the spec on every PR (`oasdiff` and similar).

```yaml
paths:
  /orders:
    get:
      operationId: listOrders
      parameters:
        - name: status
          in: query
          schema: { type: string, enum: [pending, shipped, delivered] }
        - name: cursor
          in: query
          schema: { type: string }
      responses:
        "200":
          content:
            application/json:
              schema: { $ref: "#/components/schemas/OrderList" }
```

!!! tip "Contract hygiene checklist for the interview"
    When you present an API in the round, sweat these details unprompted — they are what "has actually shipped an API" sounds like: timestamps in ISO 8601 UTC (`2026-05-10T12:00:00Z`); money as integer minor units or string decimals, **never** floats, always with an explicit `currency`; enums documented as open (clients tolerate unknown values) or closed; consistent `snake_case` (or `camelCase` — pick one); IDs opaque and prefixed; every collection paginated and capped; every error in problem-details shape with a request ID.
