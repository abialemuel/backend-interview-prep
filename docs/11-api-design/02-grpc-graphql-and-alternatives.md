# gRPC, GraphQL, and Alternatives

REST is the default, not the only answer. Interviewers increasingly expect you to compare it honestly against gRPC and GraphQL, and to know the "reverse direction" mechanisms — webhooks, WebSockets, SSE — because almost every design exercise eventually asks "and how does the server tell the client something happened?" The skill being graded is matching the tool to the consumer, not advocacy.

## gRPC

gRPC is an RPC framework over HTTP/2 using Protocol Buffers as the interface definition and wire format. Its home turf is service-to-service communication inside an organization.

### Protobuf and the contract

```protobuf
syntax = "proto3";

service PaymentService {
  rpc CreatePayment(CreatePaymentRequest) returns (Payment);
  rpc StreamPaymentEvents(StreamRequest) returns (stream PaymentEvent);
}

message Payment {
  string id = 1;
  int64 amount_minor = 2;
  string currency = 3;
  PaymentStatus status = 4;
  reserved 5;                // deleted field; number can never be reused
  google.protobuf.Timestamp created_at = 6;
}
```

The `.proto` file is a **compiler-enforced contract**: both sides generate typed code from it, so an entire class of "client sent `amount` as a string" bugs disappears. The binary encoding is 3-10x smaller than JSON and much cheaper to serialize — at high internal QPS this is real CPU and bandwidth.

Evolution rules are strict and worth reciting: field **numbers** are the wire identity, never reuse or renumber them; deleting a field means marking its number `reserved`; all fields are optional in proto3 and unknown fields are ignored, so adding fields is always safe. This gives protobuf better default evolvability than undisciplined JSON — the discipline is built into the toolchain rather than into a style guide.

### Streaming modes

HTTP/2 multiplexing gives gRPC four call shapes:

| Mode | Shape | Use case |
| --- | --- | --- |
| Unary | one request → one response | Standard RPC; the vast majority of calls |
| Server streaming | one request → stream of responses | Watch/subscribe (etcd watches), large result sets delivered incrementally |
| Client streaming | stream of requests → one response | Uploads, client-side batching of telemetry |
| Bidirectional | both stream independently | Chat, real-time sync, long-lived agent connections |

### Deadlines and cancellation

gRPC's most underrated feature. The client attaches a deadline to every call and it **propagates**: if the edge request has 800 ms left, the downstream call inherits that budget, and when it expires every server in the chain gets a cancellation and stops working on a response nobody will read. Contrast REST, where timeout policy is ad hoc per client and a timed-out caller leaves the server happily computing garbage. Deadline propagation prevents wasted work and cascading pile-ups in deep call graphs.

```go
ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
defer cancel()
resp, err := client.CreatePayment(ctx, req) // deadline travels in gRPC metadata
if status.Code(err) == codes.DeadlineExceeded { /* budget spent anywhere downstream */ }
```

### Errors and transcoding

gRPC has its own status model — a small fixed set of codes (`INVALID_ARGUMENT`, `NOT_FOUND`, `ALREADY_EXISTS`, `FAILED_PRECONDITION`, `RESOURCE_EXHAUSTED`, `UNAVAILABLE`, `DEADLINE_EXCEEDED`, ...) that maps roughly onto HTTP status classes. Two interview-relevant points:

- The codes carry **retry semantics** the same way 4xx/5xx do: `UNAVAILABLE` and `DEADLINE_EXCEEDED` are retryable with backoff, `INVALID_ARGUMENT` and `FAILED_PRECONDITION` are not. Structured detail rides along via `google.rpc.Status` with typed detail messages (`BadRequest` with field violations, `RetryInfo` with a backoff hint) — the protobuf equivalent of RFC 9457 problem details.
- **Transcoding** (grpc-gateway, Envoy's gRPC-JSON filter, Google's `google.api.http` annotations) lets you define the API once in protobuf and serve both gRPC and a derived REST/JSON surface. This is the standard answer to "we want gRPC internally but partners want REST": annotate the RPCs with HTTP bindings, generate the gateway, maintain one contract.

```protobuf
rpc GetPayment(GetPaymentRequest) returns (Payment) {
  option (google.api.http) = { get: "/v1/payments/{id}" };
}
```

### When to choose gRPC

- **Internal service-to-service** at any real scale: typed contracts, low overhead, deadlines, streaming — this is its design center, and it is the default east-west protocol at most large shops.
- Polyglot organizations: codegen for a dozen languages beats hand-maintaining a dozen HTTP clients.
- Operational conveniences that come standard: server reflection (so `grpcurl` and debug UIs can discover services), the standard health-checking protocol (`grpc.health.v1`, which load balancers and Kubernetes probes understand), and interceptors as the middleware model for auth, logging, and metrics.
- **Weaknesses**: browsers cannot speak native gRPC (you need gRPC-Web or a gateway/transcoding layer, at which point public-facing you may as well serve REST); binary payloads are not `curl`-able without tooling (`grpcurl` exists, but the debugging ergonomics are worse); load balancing needs L7 awareness (a naive L4 balancer pins all traffic on one long-lived HTTP/2 connection to one backend — you need Envoy/linkerd-style per-request balancing or client-side lookaside).
- Worth knowing in 2026: **Connect RPC** (from Buf) serves gRPC, gRPC-Web, and a plain JSON-over-HTTP protocol from the same protobuf service definition, and works from browsers and `curl` without a translation layer — it has become a popular way to keep protobuf contracts while shedding gRPC's browser and debuggability pain. `buf` itself (lint, format, **breaking-change detection against the previous schema in CI**, schema registry) is now the standard protobuf toolchain; mentioning schema-diff CI for protos is the same maturity signal as `oasdiff` for OpenAPI.

## GraphQL

GraphQL exposes a typed schema and lets the client specify exactly the shape of data it wants in a single request. It solves a real problem — mobile clients over-fetching and making waterfall round trips against resource-shaped REST — and creates several new ones.

### Schema and execution

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  customer: Customer!
  items: [OrderItem!]!
}

type Query {
  order(id: ID!): Order
  orders(status: OrderStatus, first: Int, after: String): OrderConnection!
}
```

The client asks for `order(id: "42") { status customer { name } items { sku } }` and gets exactly that JSON shape. Each field is backed by a **resolver**; the server executes the query by walking the tree. Pagination is conventionally Relay-style connections (`first`/`after` with cursors — note GraphQL standardized on cursor pagination for the same reasons REST should).

Writes are **mutations** — explicitly named operations (`createOrder(input: ...)`) that behave like RPCs and execute serially within a request; the schema convention is one focused mutation per business action with a typed input object and a payload that includes the changed entity plus user-facing errors as data (`userErrors: [{field, message}]` for validation, reserving transport-level GraphQL errors for system failures). **Subscriptions** cover server push, transported over WebSockets or SSE — all the operational costs of those transports (next section) apply; GraphQL only standardizes the envelope.

Schema evolution mirrors the REST rules with better tooling: additive changes are safe, removals go through `@deprecated(reason: "use newField")` first, and because every client query names exactly the fields it reads, field-level usage analytics are built in — you know precisely who still queries a deprecated field before you remove it. This is one of GraphQL's genuine operational advantages over REST, where field-level usage requires guesswork or response logging.

### N+1 and DataLoader

The canonical GraphQL production incident. A query for 50 orders with their customers naively executes 1 query for the orders, then the `customer` resolver fires **once per order** — 50 additional queries. The fix is **DataLoader**: resolvers do not query directly but enqueue keys with a loader that coalesces everything requested in the same execution tick into one batched query and caches per-request:

```javascript
const customerLoader = new DataLoader(async (ids) => {
  // called once with all customer IDs collected this tick
  const rows = await db.query("SELECT * FROM customers WHERE id IN (?)", [ids]);
  const byId = new Map(rows.map((r) => [r.id, r]));
  return ids.map((id) => byId.get(id));   // must return in input order
});

const resolvers = {
  Order: { customer: (order) => customerLoader.load(order.customer_id) },
};
```

The loader must be **per-request** (a global one caches stale data across users) and return results in key order. Every serious GraphQL server uses this pattern; being able to explain *why the execution model creates the problem* — resolvers are independent and composable, so no single resolver can see the whole query — is the senior-level answer.

### Persisted queries and operational hardening

An open GraphQL endpoint lets any client send arbitrarily expensive queries — deeply nested, wide, or maliciously recursive. Defenses, in increasing strictness:

- **Depth and complexity limits**: reject queries beyond a nesting depth or a computed cost score before executing.
- **Persisted queries**: clients register queries at build time (or the server learns them), then send only a query hash at runtime. Cuts payload size, enables CDN caching of GET requests, and — in allowlist mode — turns the API from "execute arbitrary queries" into "execute these N known queries," which is effectively compiling GraphQL's flexibility away in production while keeping it in development. This is the standard posture for public-facing GraphQL.
- Field-level auth in resolvers (every field is an entry point; REST-style "guard the endpoint" thinking leaves holes).

### When GraphQL is wrong

- **Server-to-server APIs**: the consumers are known, the shapes are fixed — you are paying schema, resolver, and caching complexity for flexibility nobody uses. Use REST or gRPC.
- **Simple CRUD with one or two clients you control**: same reason.
- When **HTTP caching matters**: everything is a POST to `/graphql` with a unique body, so CDN/proxy caching is lost by default (persisted queries over GET claw some back, but it is a fight rather than a default).
- File uploads, and anything where per-request cost must be predictable for capacity planning.
- Where it genuinely earns its keep: many diverse clients (web, iOS, Android, partners) with different data needs over a **rich object graph**, or as a federation layer stitching many services into one graph for frontend teams (Apollo Federation) — essentially a smarter BFF. If you have one client, you probably wanted a BFF, not GraphQL.

## Webhooks

Webhooks invert the direction: the provider POSTs events to a consumer-registered URL. They appear in every payments/platform design exercise, and the grading is on the operational details:

- **Signing.** The consumer must verify the payload came from you: HMAC-SHA256 over the raw body with a shared per-endpoint secret, sent as a header (Stripe's `Stripe-Signature`), with a **timestamp included in the signed content** to prevent replay (reject signatures older than ~5 minutes). The consumer must verify against the raw bytes before parsing — verify-after-parse is a classic bug.

```
signed_payload = "{timestamp}.{raw_body}"
signature = HMAC_SHA256(endpoint_secret, signed_payload)
```

- **Retries.** The consumer's endpoint will be down. Treat delivery as an at-least-once queue problem: retry with exponential backoff and jitter over hours or days (Stripe retries for up to 72 hours), count only 2xx as success, disable endpoints that fail persistently and notify the owner, and expose a redelivery UI/API. Deliver from a queue with per-endpoint concurrency limits so one slow consumer cannot back up everyone's events.
- **Idempotency and ordering.** Because delivery is at-least-once and retries reorder, every event carries a unique `event_id`; consumers dedupe on it and must not assume order. Best practice for both sides: send **thin events** (type + IDs) and have the consumer fetch current state from the API — this makes out-of-order delivery mostly harmless and keeps sensitive data out of third-party request logs.

The consumer-side checklist, since interviewers ask both directions:

1. Verify the signature against the **raw body** before parsing; reject stale timestamps.
2. Persist the event and return 2xx immediately; process asynchronously — a handler doing 30 seconds of work inline gets timed out and re-delivered forever.
3. Dedupe on `event_id` (unique constraint in a processed-events table).
4. Treat the event as a hint: fetch current resource state from the API rather than trusting a possibly-stale payload.
5. Reconcile: because webhooks can be missed entirely (endpoint down past the retry horizon), poll a list endpoint periodically as a backstop. Providers should expose `GET /events` precisely to make this possible.

## WebSockets and SSE

For server-to-client push where the client is a browser or app:

- **SSE (Server-Sent Events)** is a long-lived HTTP response streaming `text/event-stream`. One direction (server → client), automatic browser reconnection with `Last-Event-ID` for resume, works through ordinary HTTP infrastructure. Ideal for feeds, notifications, progress updates, and LLM token streaming — which made SSE mainstream again. If the client only ever listens, SSE is the answer.

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

id: 42
event: order.updated
data: {"order_id": "ord_8f3k2", "status": "shipped"}

id: 43
event: order.updated
data: {"order_id": "ord_9m1x", "status": "delivered"}
```
- **WebSockets** upgrade the connection to a full-duplex message channel. Needed when the client genuinely sends frequently too: chat, collaborative editing, multiplayer, trading. Costs: stateful long-lived connections (sticky sessions or a connection-aware tier; a pub/sub backplane like Redis to route messages across nodes), heartbeats, and hand-rolled reconnect/resume logic — you own everything HTTP gave you for free.
- Either way, note the fleet-wide consequence: 500k connected clients is a memory/file-descriptor capacity problem and a thundering-herd reconnect problem on deploys — this is where the interviewer wants to hear connection draining and jittered reconnects. Plain polling remains a legitimate answer for low-frequency updates; it is stateless and boring, and boring scales.

## Decision table

| Dimension | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Primary consumers | Public/partner clients, browsers | Internal services | Multiple diverse frontends |
| Contract | OpenAPI (optional, bolt-on) | Protobuf (mandatory, compiled) | Schema (mandatory, runtime) |
| Wire format | JSON (readable, verbose) | Binary protobuf (compact, fast) | JSON |
| Fetch shape | Server decides (over/under-fetch) | Server decides | Client decides |
| HTTP caching / CDN | Native and excellent | Not applicable | Poor by default; persisted queries help |
| Streaming | SSE/WebSockets bolted on | First-class, four modes | Subscriptions (via WebSockets/SSE) |
| Browser support | Native | Needs gRPC-Web/gateway | Native |
| Debugging | `curl` and eyeballs | Needs tooling (`grpcurl`) | Good tooling (GraphiQL), harder tracing |
| Failure/perf predictability | Per-endpoint, easy to reason about | Per-RPC, deadlines built in | Per-query, variable cost |
| Evolution tooling | Spectral lint + oasdiff in CI | `buf breaking` in CI; reserved field numbers | `@deprecated` + built-in field usage analytics |
| Typical role | Public API, simple internal APIs | East-west service mesh traffic | BFF/aggregation layer for frontends |

The composite answer that lands well: **REST for the public API** (universal clients, HTTP caching, curl-ability), **gRPC for internal service-to-service** (contracts, performance, deadlines), **GraphQL only when a diverse-client aggregation problem actually exists**, webhooks for provider→consumer events, SSE for one-way push, WebSockets only for genuine bidirectional traffic. Choosing one religion for everything is the junior tell; matching protocol to consumer is the senior one.

!!! tip "Answering 'REST or gRPC or GraphQL?' in the room"
    Do not answer in the abstract — interrogate the prompt: who are the consumers (browsers? partners? your own services?), how many client shapes exist, does the org already run a mesh or a federation layer, and what is the caching story. Then commit: "public surface REST, east-west gRPC, and I would not introduce GraphQL here because there is one client" is a complete, defensible answer. Hedging across all three without choosing reads worse than a wrong-but-reasoned choice.
