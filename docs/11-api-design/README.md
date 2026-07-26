# API Design

API design has quietly become its own interview round at a growing number of companies — Stripe, Shopify, Twilio, and most API-first or platform teams run a dedicated "design the API" session separate from the system design round. The reason is simple: an API is the one part of your system you cannot refactor freely. Internal code can be rewritten on a whim; a published API is a contract with people you do not control, on release schedules you do not control, and every mistake you ship becomes something you support for years. Companies that sell APIs (or whose internal platform teams serve dozens of consuming teams) have learned that the ability to design a clean, evolvable contract is a distinct skill from the ability to design a scalable system — and they interview for it separately.

The API design round also has a different texture from system design. There is less capacity math and more judgment: naming, resource modeling, error shapes, versioning policy, pagination mechanics, what to do when a client retries a payment. The interviewer is usually probing for scar tissue — has this candidate actually lived with an API in production, watched clients depend on undocumented behavior, and had to evolve a contract without breaking anyone? The good news is that this material is highly learnable, and unlike system design there are broadly agreed-upon right answers to many of the questions (use cursor pagination for anything unbounded; use RFC 9457 for errors; never delete a field from a response).

As with the system design section, the goal here is not memorization but **trade-offs**: REST vs gRPC vs GraphQL is not a religious question, it is a table of costs and benefits that flips depending on who your consumers are. Offset vs cursor pagination is not a style choice, it is a correctness and performance decision. Concentrate on being able to say *when* each choice is right and what you pay for it.

## What this section covers

| File | Description |
| --- | --- |
| README.md | This overview: why API design is its own round, scope, and recommended reading order. |
| 01-rest-and-http.md | Resource modeling, HTTP semantics (methods, status codes, headers, caching, conditional requests), pagination, filtering and sorting, RFC 9457 error format, versioning strategies, HATEOAS pragmatism, OpenAPI. |
| 02-grpc-graphql-and-alternatives.md | gRPC (protobuf, streaming, deadlines), GraphQL (schema, N+1, persisted queries), webhooks, WebSockets and SSE, and the decision table for choosing between them. |
| 03-api-operations.md | The operational side: idempotency keys, rate limiting algorithms and headers, authentication options, API gateways, backward compatibility and deprecation, contract testing, observability. |
| 04-interview-questions.md | Graded practice questions from junior to staff level, including full design exercises with model answers. |

## What the API design interview actually tests

A typical round hands you a prompt like "design the API for a ride-hailing service" or "design Stripe's payments API" and expects you to:

1. **Model the resources.** Identify the nouns, their relationships, and their lifecycles. This is the API equivalent of schema design, and getting it wrong poisons everything downstream.
2. **Define the operations.** Map actions onto HTTP methods (or RPCs), handle the awkward cases (actions that are not CRUD, long-running operations, bulk operations), and choose sensible URLs.
3. **Specify the contract precisely.** Status codes, error bodies, pagination, field naming, timestamps, money representation. Vague hand-waving here is the most common failure mode — interviewers want to see actual request/response examples.
4. **Handle failure.** What happens on retry? On partial failure? On concurrent modification? Idempotency and optimistic concurrency are near-guaranteed follow-ups.
5. **Plan for evolution.** How do you add a field, rename a field, deprecate an endpoint, and version the API — without breaking a client you have never met?

Senior and staff variants add: rate limiting policy, authentication model, webhook design for the reverse direction, SLA/deprecation policy, and the organizational question of how you keep fifty teams' APIs consistent (style guides, linting, review boards).

## Recommended reading order

1. `01-rest-and-http.md` — the foundation. Most API interviews are REST interviews, and most of the judgment calls (pagination, errors, versioning) live here. Read this first.
2. `02-grpc-graphql-and-alternatives.md` — the alternatives and when they win. You need enough depth on gRPC and GraphQL to compare honestly, plus webhooks because "how does the server call the client back" appears in almost every design exercise.
3. `03-api-operations.md` — the production concerns that distinguish a senior answer from a junior one. Idempotency and rate limiting in particular are asked constantly.
4. `04-interview-questions.md` — self-test after the first three. Answer out loud before reading the model answer; in this round more than most, precise wording is the skill being graded.

A note on framing: examples here use JSON-over-HTTP with occasional Go and PHP snippets, because that is the lingua franca of the interview. But the underlying discipline — model resources carefully, make the contract explicit, design for the retry, never break a client — is identical whether the transport is HTTP, gRPC, or a message queue.
