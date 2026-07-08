# System Design

The system design interview is one of the highest-signal rounds in a backend engineering loop. It is not a test of how many buzzwords you can deploy; it is a test of whether you can take an ambiguous problem, structure it, make defensible trade-offs, and communicate clearly while doing so. This section is a condensed reference for the concepts you are most likely to be asked about, with explicit tie-ins to stacks a working backend engineer already knows (AWS, MySQL, Redis, PHP/Go, Laravel).

The goal is not memorization. Two candidates can arrive at very different architectures for the same prompt and both pass, provided each can justify their choices. Concentrate on **trade-offs**: what you gain, what you pay, and under what conditions the trade flips.

## What this section covers

| File | Description |
| --- | --- |
| README.md | This overview: framework, scope, and recommended reading order. |
| 01-scalability-and-load-balancing.md | Scaling theory, load balancing, capacity estimation, CAP/PACELC, consistency models, the database scaling path, async processing, latency numbers, proxies. |
| 02-caching-and-microservices.md | Deep dive on caching (patterns, failure modes, invalidation, consistency) and on microservices (boundaries, communication, sagas, outbox, resilience patterns, observability, migration). |
| 03-interview-questions.md | Graded practice questions covering both conceptual prompts and mini design exercises with model answers. |

## The standard system design interview framework

A typical 45-minute round breaks down roughly as follows. Spend the time intentionally; the worst thing you can do is jump straight into boxes and arrows.

1. **Clarify requirements (3-7 min).** Never assume the prompt. Ask about functional requirements (what the system does), non-functional requirements (scale, latency, consistency, availability), and constraints/users. Explicitly clarify the Read/Write ratio, who the users are, and which quality is the highest priority (latency vs consistency vs cost). Restate the requirements back to the interviewer and agree before moving on.
2. **Back-of-the-envelope estimation (3-5 min).** Compute QPS, storage, bandwidth, and connections. The number itself matters less than the order of magnitude — it tells you whether you need sharding, a cache, or a CDN. Call out assumptions. Example: "1M DAU, each does 10 reads/day -> 10M reads/day -> ~115 RPS average, ~1k RPS peak at 10x."
3. **High-level design (10-15 min).** Draw the components and the flow of a single request end-to-end: client -> DNS/CDN -> load balancer -> web/app tier -> service tier -> database/cache. Name concrete technologies only when you can justify them. Keep it simple first; you cannot scale a design you do not yet have.
4. **Deep dive (10-15 min).** Pick one or two areas the interviewer cares about and go deep: schema, API, caching strategy, queue topology, sharding key, failure modes. This is where most of the signal is. Let the interviewer steer, but do not wait passively.
5. **Bottlenecks and trade-offs (5 min).** Walk the design and name single points of failure, consistency boundaries, capacity ceilings, and what breaks first at 10x and 100x current scale. Discuss what you would sacrifice first (consistency? cost? durability?) under degradation.
6. **Scaling and follow-ups (remaining time).** How does the design change at 10x? What caching do you add, how do you shard, do you split read and write paths, do you move to an event-driven architecture? Have an opinion and a path.

A few meta-habits matter as much as the content:
- **Think out loud.** The interviewer is evaluating your reasoning, not the final diagram.
- **Drive.** Do not wait to be asked "what about caching?" — propose it, justify it, then move on.
- **Use the whiteboard.** Diagrams clarify your own thinking more than the interviewer's.
- **Know when to stop.** A 45-minute round cannot fit everything. Recognizing what to defer is itself a signal.

## Recommended reading order

1. `01-scalability-and-load-balancing.md` — the foundations. Scaling models, the load balancer layer, CAP/PACELC, capacity math, consistency models. Read this first; everything else assumes it.
2. `02-caching-and-microservices.md` — the two biggest practical levers for backend systems. Caching is where most production outages originate; microservices are where most architecture questions end up.
3. `03-interview-questions.md` — use this for self-testing after reading the first two. Try to answer out loud before reading the model answer; the wording matters more than the list of facts.

A note on stack-specific framing: concepts here are presented generically because they translate across employers and stacks. Where useful, examples map onto an AWS + MySQL + Redis + Laravel/Go stack — but the patterns themselves (cache-aside, the outbox, saga choreography, sharding by hash) are exactly the same in any ecosystem.