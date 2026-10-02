# Model Answers — Verified Glassdoor Questions

Technical practice answers for the historical question bank in [03-interview-questions.md](03-interview-questions.md). These reports do not establish your interview sequence. Your next confirmed interviews assess fit; prioritize the [recruiter answers](01-recruiter-exploratory-call.md) and [manager answers](05-hiring-manager-fit.md), which are tailored to your CV. Describe technical proposals as proposals and claim personal implementation only when accurate.

## 1. "What happens when I do a search in a browser?" (Senior SWE — verified)

This is a systems-walkthrough probe. The trap is reciting the blog-post meme and stopping; the win is walking layers with **one senior-level observation per layer** and offering to go deeper anywhere. Structure: client → network → server → back to client.

**Answer:**

> I'll walk it end to end, saying where things can go wrong at each layer.
>
> **URL parsing.** The browser splits the query into scheme, host, path, and query params. One senior detail: the search input goes into the query string, so anything typed is visible in logs and referrer headers — which is why it must be transmitted over TLS, and why search terms are PII from a compliance view.
>
> **DNS.** The browser checks its own DNS cache, then the OS cache (and `/etc/hosts`), then asks the configured resolver. If uncached, the resolver does the recursion: root → `.com` TLD → authoritative nameserver. Senior observations: TTLs bound how fast you can fix mistakes (a low TTL before a migration is a deliberate tactic), and CDN providers use DNS geo-routing, so *where the name resolves to already shapes the whole request path*. For a streaming service, DNS is the first traffic-steering decision.
>
> **TCP + TLS.** The browser opens a TCP connection (SYN/SYN-ACK/ACK, one RTT) and then TLS 1.3, which completes in one additional RTT with 0-RTT resumption for revisits. HTTP/3 replaces this with QUIC over UDP, combining transport and crypto handshakes — fewer round trips, and no head-of-line blocking across streams. If the interviewer asks "why did HTTP/3 move to UDP?", the honest answer is: not because UDP is faster, but because TCP evolution was trapped in OS kernels and middleboxes, while QUIC lives in user space and can iterate.
>
> **The request hits the edge.** For a search request, the CDN usually passes through (search responses are personalized, so they're not cacheable — you state the *cacheability decision*, not just "there's a CDN"). The edge forwards to a load balancer, which picks an instance — and here is where the round-robin vs weighted discussion from this company's own interview reports plugs in.
>
> **Server side of a search.** The request hits a stateless service layer. Auth/session comes from a cookie or token. The query goes to a search backend — typically an inverted index (Elasticsearch/OpenSearch family) or a database with a full-text index, ranked by relevance scoring (BM25-style plus business boosts: availability in the user's region, trending signals). The hot path is protected by caching (popular queries) and rate limiting (abuse). Senior observation: for a content platform the ranking input includes rights metadata, because returning a result the user cannot watch is a wasted click — relevance is not just text matching.
>
> **Response and rendering.** JSON over HTTP/2 or 3 comes back; the browser parses HTML into a DOM, CSS into a CSSOM, builds the render tree, computes layout, paints, and composites. JavaScript can block parsing — hence `defer`/`async` and code splitting. Caching headers (`Cache-Control`, ETags) decide what the next identical visit costs. First contentful paint is the client-side mirror of the server-side p99 latency.
>
> If asked "what would you measure?" — server: DNS resolution time, TLS handshake time, TTFB, p99 of the search service; client: FCP and interaction latency. If asked "what breaks first under load?" — the search backend's index and the DB connection pool, before the stateless service tier, because services scale horizontally but stateful tiers need capacity planning.

For a personal connection, use an incident you actually investigated. The CV documents Careem's error classification and retry work, but does not document DNS failover or TLS storms.

## 2. "Implement a round-robin load balancer" (verified)

Verbatim report: *given servers and requests as string arrays*. Clarify: "each request goes to the next healthy server in cycle?"

```go
type Balancer struct {
    mu  sync.Mutex
    servers []string
    next uint64
}

func (b *Balancer) Pick() string {
    b.mu.Lock()
    defer b.mu.Unlock()
    s := b.servers[b.next%uint64(len(b.servers))]
    b.next++
    return s
}
```

Say out loud: O(1) dispatch; the counter is shared state so it needs atomicity (`atomic.AddUint64` works too if the server list is fixed); modulo jumps when the list changes mid-flight — an instance added at index 0 reshuffles assignments, which is why production balancers graduate to consistent hashing ([deep dive](../13-distributed-systems/02-replication-and-partitioning.md)). If they add failures: "health checks remove instances from the slice; the counter mod adjusts — acceptable skew, or use a ring."

## 3. "Implement a weighted round-robin load balancer" (verified follow-up)

Two answers, present both:

**Naive:** expand the list — server with weight 3 appears 3 times, plain round-robin over the expanded list. Correct, O(1), memory O(total weight); bursty assignment (weight-3 server gets 3 consecutive requests).

**Smooth weighted round-robin (nginx algorithm) — the answer that lands:**

```go
type Server struct {
    name          string
    weight        int
    currentWeight int
}

func (s *Server) Pick(servers []*Server) *Server {
    total := 0
    var best *Server
    for _, sv := range servers {
        sv.currentWeight += sv.weight
        total += sv.weight
        if best == nil || sv.currentWeight > best.currentWeight {
            best = sv
        }
    }
    best.currentWeight -= total
    return best
}
```

Why it's better: with weights A:5, B:1, C:1, naive gives AAAAABC; smooth interleaves ABACAA-ish patterns, so per-second fairness is better. Mention nginx and LVS use this — one sentence of "I know where this runs in production" is worth more than the code.

## 4. OS fundamentals: heap vs stack (verified — "OS questions tested on heap stack etc")

> The stack is per-goroutine in Go, LIFO, holds frames: locals whose size the compiler knows, return addresses, registers. Allocation is a pointer bump — effectively free; freed on return. The heap is shared, managed by the GC, holds anything that escapes the frame: values returned upward, captured by goroutines, or too large. Go's **escape analysis** decides at compile time, and you can see it with `-gcflags=-m`.
>
> Allocation patterns and garbage-collection pressure can affect memory and latency. I would profile the workload before deciding whether pre-sizing collections, reusing buffers, or changing data ownership would help.

Your Telkom monitoring improvement came from the documented stateless architecture, Kubernetes autoscaling, and Go worker-pool concurrency. The CV does not attribute its ~80% memory reduction to value semantics or stack allocation.

If probed on "what's in the heap vs stack for a closure?" — a closure capturing a local forces that local to escape.

## 5. DB fundamentals: ACID (verified — "DB on ACID")

> **Atomicity**: all-or-nothing — a transfer debits A and credits B in one transaction; a crash rolls back. InnoDB keeps the old values in **undo logs** to make this possible.
>
> **Consistency**: the DB moves between valid states — constraints, FKs, CHECKs hold. Senior nuance: the engine enforces declared constraints, but *domain* consistency (money sums staying zero across services) is the application's job.
>
> **Isolation**: concurrent transactions don't see each other's intermediate states; in practice it's a dial, not a switch — READ COMMITTED vs REPEATABLE READ vs SERIALIZABLE in MySQL InnoDB (MVCC + locking under the hood). Anomalies to name: dirty read, non-repeatable read, phantom read.
>
> **Durability**: committed means survives crash — **redo log** with write-ahead and fsync policy (`innodb_flush_log_at_trx_commit=1`), which is the exact knob where throughput vs safety lives. That's the senior answer: each letter maps to a concrete mechanism and a tunable trade-off.

## 6. Networking: TCP/IP vs UDP (verified)

> TCP gives you a byte stream: connection setup, ordering, retransmission, flow and congestion control — reliability at the cost of latency and head-of-line blocking. UDP gives you datagrams: no setup, no ordering, no retry — you own reliability if you need it.
>
> Why it matters for streaming: video delivery does *not* use raw UDP for the media itself — HLS/DASH segments ride TCP or QUIC. The reason: the player needs every byte of a segment in order, and the ecosystem (CDNs, middleboxes) is TCP/QUIC-native. Raw UDP loss handling would mean rebuilding FEC/ARQ yourself. Where UDP genuinely wins and we use it: DNS (single round trip), QUIC (reliability re-implemented in user space, no TCP head-of-line blocking, better loss recovery on mobile — directly relevant to a streaming player on flaky networks), and WebRTC (interactive latency where stale frames are useless).
>
> The senior synthesis: "reliability is a spectrum you engineer per use case — TCP is the default, UDP is the tool when latency beats completeness or when you need user-space innovation."

## 7. "Algorithm questions on sorting and practical API usage" (verified, Senior SWE)

**Sorting internals, the 3-minute version:**

> Quicksort: in-place, cache-friendly, O(n log n) average, O(n²) worst unless randomized/pivot-median; not stable. Mergesort: O(n log n) always, stable, O(n) extra memory. Heapsort: O(n log n) in-place, not stable, poor cache locality. Timsort (Python/Java) exploits real-world partial ordering — runs + merge. In Go: `sort.Slice` is quicksort-family (unstable, introsort — falls back to heapsort to bound the worst case), `sort.SliceStable` is mergesort-family; Go 1.19+ uses pdqsort — pattern-defeating quicksort, detects sorted/rev-sorted input, worst case still bounded. Saying "pdqsort" and *why* (adaptive + bounded worst case) signals currency.
>
> When I'd not sort at all: top-K with a heap (O(n log k) beats full sort for k << n), counting/radix sort for bounded integer keys, and for "sort then binary search" asks — consider a map or a sorted structure maintained incrementally.

**"Practical API usage"** — they're testing whether you know the *contracts* you'd build and consume daily. Cover in 60 seconds:

> I would consider pagination behavior under concurrent writes, safe retries for state-changing requests, rate-limit behavior, compatibility, error contracts, and efficient database access. The right design depends on the client and service requirements.

For experience, describe only mechanisms you actually implemented. A study guide topic is not evidence that you shipped that API feature.

## 8. "Have you used Rakuten Viki before? How would you improve it?" (verified HM question)

Explore the available product flows first, without needing to purchase a subscription. Then answer with an observation, user impact, and a hypothesis to validate:

> I've been exploring Viki as part of my preparation. The Asian-content focus and subtitle community stand out. I'd like to discuss **[one real observation from your exploration]**, understand its user impact, and check whether it is an area the team is already addressing.

Use this only after actually exploring the product. Do not claim weeks of usage or an observed defect without evidence. The [manager guide](05-hiring-manager-fit.md) includes a product-review approach. Ask about existing metrics and priorities before asserting a bottleneck or prescribing a solution.

## 9. Relocation pushback (verified — "why would you need more than 2 weeks?")

> I would plan the start date around my actual notice period, approved work authorization, and the relocation arrangements we agree on. Could you explain your preferred timeline and the support available?

This historical question came from another role. Do not promise a two-week start, remote work approval, or office attendance arrangements that have not been agreed. See the [recruiter logistics preparation](01-recruiter-exploratory-call.md).

Frame: cooperative, concrete, no defensiveness. The question is testing logistics realism, not loyalty.

## 10. Take-home (verified pattern — build-an-app / scrape-for-duplicates)

Deliverable bar: small clean repo, not a clever script. README with run instructions; tests covering the core logic and at least one edge case; pagination/HTTP politeness if scraping (robots.txt, rate limit); a short "design notes" section saying what you'd harden for production. Time-box visibly — a note like "spending 2 more hours I would add X" reads as senior judgment.

## Drill plan

1. Answer 1 and 4–6 out loud, timed to 3 minutes each — these are the verified fundamentals probes.
2. Code 2 and 3 from memory in Go, twice.
3. Use the Viki app before every round — answer 8 must reference *current* product behavior.
