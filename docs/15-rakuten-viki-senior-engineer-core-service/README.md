# Rakuten Viki — Senior Engineer, Core Service

Targeted preparation pack for the **Senior Engineer, Core Service** role at Rakuten Viki (Singapore, relocation role). The Codility online assessment is already done — this pack covers every remaining stage: HR screen, technical coding rounds, system design, and the hiring-manager/behavioral loop.

> **Evidence note:** Rakuten Viki does not publish its engineering interview process, but **Glassdoor carries real candidate reports for company id E694051** (53 questions / 51 reviews as of 2026), mined via search-indexed question pages since the site itself is login/Cloudflare-walled. The stage map and question bank combine that verified data with NodeFlair's 39 candidate reports. The full loop count for this specific role is still an estimate — verify with the recruiter. Salary figures are listing/community data, not a guaranteed offer.

---

## Company snapshot

| Fact | Detail |
|------|--------|
| What it is | OTT streaming service specializing in Asian TV — Korean, Chinese, Taiwanese, Japanese dramas and films, distributed globally |
| Founded | 2007 (as Viki — "video" + "wiki"); launched publicly Dec 2010 |
| Singapore link | Moved HQ operations to Singapore in 2008; Singapore remains a major engineering and business office (alongside San Mateo HQ, Tokyo, Seoul) |
| Ownership | Acquired by Rakuten Group in 2013 (~US$200M); operates inside Rakuten's global digital content ecosystem (Rakuten TV, Kobo, etc.) |
| Scale | 70+ million users (2024) |
| Business model | SVOD (**Viki Pass** subscription) + free ad-supported tier (AVOD); content syndication deals with partners such as Hulu and Netflix |
| Signature differentiator | Community-powered subtitles — volunteer-contributed subtitles in **200+ languages** (~50 of them endangered languages), built on its own collaborative subtitling platform |
| Originals | *Dramaworld* (2016), *Where Your Eyes Linger* (2020), *Light on Me* (2021), 100+ originals |

Talking points that land: it is one of the few streaming products where the **community is part of the product** (crowdsourced translation at scale), it runs **both SVOD and AVOD** (entitlement + ad-serving complexity), and the Singapore office is a genuine engineering hub, not a satellite office.

## What "Core Service" likely means

No public team map exists. Based on the title and Viki's product shape, Core Service most plausibly owns the central backend platform that everything else rides on — labeled inference, confirm with the recruiter:

- **Catalog / metadata** — titles, episodes, series, availability windows, geo-rights
- **Playback** — stream manifests, signed CDN URLs, device/session handling
- **Entitlement & subscription** — Viki Pass checks, trials, purchases, ad-tier gating
- **User / identity** — accounts, profiles, auth
- **Subtitles pipeline** — the community translation workflow and subtitle delivery
- **Player telemetry** — QoS/rebuffering beacons feeding ops and recommendations

The interview-relevant common denominator: **high-read-volume APIs, caching, low-latency hot paths, correctness of entitlement/billing decisions, and event pipelines** — all standard senior backend territory and all covered by this repo.

## Stage map

| # | Stage | Status / what happens |
|---|-------|----------------------|
| 1 | Codility online assessment | ✅ Done |
| 2 | HR / recruiter screen (~30 min) | **Next.** Motivation, background, salary, relocation logistics. See HR screen below |
| 3 | Technical coding round(s) (~60 min each) | Live coding — **Glassdoor-verified topics**: round-robin/weighted load balancer implementations, sorting, "practical API usage". Some loops add a **CS-fundamentals round** (OS: heap/stack; DB: ACID; networking: TCP/IP vs UDP) — see [interview questions](03-interview-questions.md) |
| 4 | System design round (~60 min) | Senior-level backend design; verified senior questions include the browser-search walkthrough — see [system design](02-system-design.md) |
| 5 | Hiring manager / behavioral | **Verified HM question: "Have you used Rakuten Viki before, and how would you improve the product?"** — use the product before this round. Plus team fit and past-project deep dives |
| 6 | Offer + relocation logistics | Salary, EP sponsorship, relocation package, start date |

Some loops compress stages 3–4 into a single 90-minute session, and some add a take-home or a second design round. Ask the recruiter for the exact loop map — candidates are entitled to know how many rounds and who sits on each.

## HR screen preparation

### Talking points

- 8+ years building production backend systems in **Go and Ruby** across e-commerce, telecom, and ride-hailing; currently at Careem (Uber).
- **Careem**: large restaurant-network integration in Saudi Arabia; rebuilt a government event relay — error classification of transient/terminal/ordering failures, SQS FIFO retries, ordering gate, exponential backoff, elimination of silent event loss.
- **Telkom Indonesia**: built an AI Proxy and MCP Orchestrator used across multiple business units; redesigned a monitoring workload into a stateless autoscaled Go architecture cutting memory use ~80%.
- Earlier: Bukalapak, Tanihub — caching (Redis), database performance, high-traffic e-commerce.
- Mentoring and backend standards work — senior-level ownership signal.

### "Why Rakuten Viki?" (draft, rehearse aloud)

> Viki is a product where backend engineering is directly visible in the user experience: a playback request has to resolve catalog, rights, entitlement, and subtitles in milliseconds, and the community-subtitling pipeline is a genuinely unusual distributed-workflow problem. I have spent my career on integration-heavy, reliability-critical backend systems — at Careem I rebuilt a regulated event relay where silent event loss was the cardinal sin, and at Telkom I built shared platform services used across business units. Core Service at Viki is that same job: the platform every other team depends on. Rakuten ownership adds stability and a global ecosystem, and the Singapore engineering office is where the product is actually built.

### "Why Singapore?" (draft)

> I have already worked with Singapore-based and international teams, and I am intentionally open to relocation. Singapore is Viki's engineering hub, it is a regional base for the streaming industry, and I am ready to commit long-term. This is a relocation conversation I am prepared to move quickly on.

### Likely HR questions

- Timeline: notice period, earliest start, willingness to relocate (have a concrete date — verified SG reports show recruiters push back on anything over ~2 weeks and ask about commuting/hybrid logistics).
- Current compensation and expectations — see salary section below; give a **monthly SGD base range**, ask about package structure (bonus, benefits, relocation).
- Any other offers in flight (be honest, no games).
- Visa status / EP sponsorship requirement — say it plainly: will need Employment Pass sponsorship and relocation support.
- Hybrid/onsite expectations in Singapore — one verified SG report cites ~20 remote working days in the benefits package; confirm current policy and ask what happens before permanent housing is secured.

### Questions to ask the recruiter

1. How many interview rounds remain, and what does each assess?
2. Which team is "Core Service" — catalog, playback, entitlement, platform, or a mix?
3. What is the core stack (languages, cloud, data stores) and the on-call model?
4. Does the offer include Employment Pass sponsorship and the relocation package (housing, flights, temporary accommodation)? What is the typical EP timeline?
5. What does success look like at 6 and 12 months for this role?

## Salary research (Singapore, monthly base)

| Source | Figure |
|--------|--------|
| NodeFlair user submissions — Software Engineer, Senior | avg **S$9,450/mo**, range S$8,500–S$10,400 |
| NodeFlair — MyCareersFuture historical listing data | S$6,545–S$11,909/mo |
| Glassdoor SG — Software Engineer (28 submissions, all levels, Jan 2026) | avg S$7,000/mo |
| NodeFlair — Software Engineer, Mid (reference) | avg S$7,500/mo (S$7,000–S$8,000) |
| NodeFlair — Software Engineer, Junior (reference) | avg S$6,043/mo (S$5,000–S$6,500) |

Positioning: with 8+ years, a current senior role at Careem (Uber), and a relocation package in play, quote **S$10,000–S$11,000/mo base** as the target, settle anchored at S$9,500+. NodeFlair company reviews rate Viki compensation at 3.6/5 (the weakest category vs 4.1 overall), and Glassdoor's all-level SWE average of S$7,000/mo reinforces the picture of a cost-managed comp stack — be ready to negotiate on total package (bonus, Viki Pass perks, relocation value, AWS/cert budget) rather than base alone. Employment Pass approvals for senior roles typically require qualifying salaries well above the S$5,600 minimum (Jan 2025 baseline, higher with age via COMPASS) — a sub-S$9,000 offer for a relocated senior is below market; treat that as a data point in negotiation.

## Relocation and Employment Pass quick facts

- **Employment Pass (EP)**: minimum qualifying salary S$5,600/month from 1 Jan 2025, higher thresholds by age (40s and above); COMPASS points system applies (salary, qualifications, local-vs-foreign mix of the firm). A senior hire is normally EP-feasible, but the company must sponsor it — ask directly.
- Standard relocation asks: flights, temporary housing (4–8 weeks), shipping allowance, and a relocation-agency contact. Get what is promised in the offer letter, not verbally.
- Useful to ask: who handles the EP application (Viki HR or an agency — they usually do this), and expected processing time (typically 2–6 weeks after submission).

## CV-to-JD mapping

| Core Service theme | Your anchor story |
|--------------------|-------------------|
| High-scale read APIs and caching | Bukalapak/Tanihub production caching and Redis work; CDN/edge caching concepts in [06-system-design](../06-system-design/README.md) |
| Integration reliability, event pipelines | Careem restaurant-network integration (KSA); CERT relay: transient vs terminal failure classification, SQS FIFO retries, ordering gate |
| Platform services used by many teams | Telkom AI Proxy + MCP Orchestrator across business units |
| Efficiency and stateless design | Monitoring workload rebuilt as stateless autoscaled Go, ~80% memory reduction |
| Correctness under failure (billing/entitlement analog) | Regulated trip-event relay with zero silent loss; idempotency and dedup practice in [13-distributed-systems](../13-distributed-systems/README.md) |
| Mentoring and standards | Backend standards definition, mentoring engineers at Careem |

## Study plan for this week

1. Re-read [API Design](../11-api-design/README.md) — pagination, versioning, idempotency keys, rate limiting. Core Service is API work first.
2. Work through [System Design](../06-system-design/README.md) — caching and microservices especially; then [02-system-design.md](02-system-design.md) in this pack.
3. Code daily in Go: scheduling/LB problems, then the graded list in [03-interview-questions.md](03-interview-questions.md) with the spoken model answers in [04-model-answers.md](04-model-answers.md).
4. Skim [13-distributed-systems](../13-distributed-systems/README.md) — exactly-once, idempotency, locks. Senior follow-ups live here.
5. Rehearse the two drafted answers and three anchor stories aloud ([Behavioral](../08-behavioral/README.md)).

## Final checklist

- [ ] Confirm with recruiter: rounds remaining, loop map, stack
- [ ] Rehearse "Why Viki?" and "Why Singapore?" aloud
- [ ] Salary range decided: S$10,000–S$11,000 base, package negotiation plan ready
- [ ] Relocation questions prepared (EP, housing, flights, timeline)
- [ ] Notice period / start date concrete
- [ ] Daily Go coding practice scheduled
- [ ] System design walkthrough of this pack done at whiteboard-level fluency

## References

- [Rakuten Viki site](https://www.viki.com/)
- [Rakuten Viki — Wikipedia](https://en.wikipedia.org/wiki/Rakuten_Viki) (history, acquisition, scale figures)
- [NodeFlair — Rakuten Viki salaries](https://nodeflair.com/companies/rakuten-viki/salaries) and [interview reports](https://nodeflair.com/companies/rakuten-viki/interviews)
- [Glassdoor — Rakuten Viki interview questions](https://www.glassdoor.com/Interview/Rakuten-Viki-Interview-Questions-E694051.htm) (login-walled; indexed question pages: [verified senior SWE questions](https://www.glassdoor.com/Interview/I-got-some-algorithm-questions-regarding-sorting-and-practical-API-usage-QTN_8639661.htm), [coding + fundamentals report](https://www.glassdoor.com/Interview/Coding-Round-Given-servers-and-requests-represented-by-string-arrays-implement-round-robin-load-balancer-Follow-up-quest-QTN_7315639.htm))
- [Glassdoor SG — Rakuten Viki Software Engineer salaries](https://www.glassdoor.sg/Salary/Rakuten-Viki-Software-Engineer-Salaries-E694051_D_KO13,30.htm)
- [MOM Employment Pass — qualifying salary](https://www.mom.gov.sg/passes-and-permits/employment-pass/eligibility)
- [Viki careers presence on LinkedIn](https://www.linkedin.com/company/rakuten-viki/) — check posters and recent posts before each call
