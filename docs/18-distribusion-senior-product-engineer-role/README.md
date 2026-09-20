# Distribusion Technologies — Senior Product Engineer, IMS

Targeted prep for the **Senior Product Engineer, IMS** role (Remote/Berlin, Hybrid) at Distribusion Technologies. This pack covers the **HR screen** first — that is the immediate next stage — plus company research you can use if asked.

> **Evidence note:** Distribusion does not publish its interview process. The stage map and salary figures below are estimates from public data (company site, news page, job post, Berlin market norms). Salary numbers are market estimates, not offer data. Verify with the recruiter.

> **Salary-data note:** Glassdoor, kununu, and levels.fyi have no accessible Distribusion data (pages blocked or empty — small company, few reviews). Compensation expectations must come from Berlin market benchmarks and what Aulia confirms in the screen.

> **Source CV:** `~/Downloads/Abia New CV.pdf` (July version). All candidate facts below come from it.

> **Contact:** Talent Partner Aulia — `talent@distribusion.com`. Use this email for scheduling questions, thank-you follow-ups, and logistics.

## 1. Company research cheat sheet

Memorize these. HR loves candidates who did homework, and one factual slip ("isn't that a bus company?") is hard to recover from.

### What they are

- **Ground transportation technology company**, HQ Berlin. B2B marketplace/platform, not a consumer travel site.
- One-line pitch: *"Booking ground transportation should be as easy as booking a flight."* They connect carriers (bus, rail, ferry, airport transfer) with online travel retailers.
- Claims the **first global B2B booking API** for ground transport and the **largest carrier/retailer network** in the industry.
- Flow: **search → booking → settlement**. One contract for the carrier, one standardized integration, unified settlement. The platform handles commercial terms, distribution, and invoicing.
- **2,000+ retailers**, **250+ carriers**, **70+ countries** (site numbers; the job post says "hundreds of operators").
- Retail partners: **Google Maps, Booking.com, Trainline, Expedia, Kayak, Trip.com, GetYourGuide, Moovit, Rome2Rio, Busbud**.
- Carrier partners include **Deutsche Bahn, Amtrak, Renfe, SNCF, Brightline, Megabus, WESTbahn, European Sleeper, LTG Link, KAI (Indonesia rail)**.
- Growth: **10x in the past year** (job post claim), one of the fastest-growing startups in travel.

### Funding and leadership

- **$80M Series C, September 2024**, led by **TQ Ventures**. Existing investors: **Creandum, Northzone, Lightrock**.
- Leadership: **Thomas Doering (CEO)**, **Johannes Thunert (Founder & CBO)**.
- Recent strategic moves worth name-dropping:
  - **Won a Deutsche Bahn tender** (Sep 2024) for a new multi-carrier sales solution — a big credibility signal with national rail operators.
  - Rail expansion: KAI (Indonesia), Railways of Slovakia, WESTbahn (Austria), LTG Link (Lithuania), Arenaways (Italy), European Sleeper.
  - Corporate travel / TMC push: Arrive Agencies, Lanes & Planes partnerships (2025).
  - Brazil market: closing the offline-to-online bus sales gap.
  - Positioning includes **sustainability**: shifting travellers to ground transport = lower emissions vs. short-haul flights.
- Cybersecurity certified (Cyber Essentials badge on their site).

### The role: what "IMS" means

IMS = the internal/operator-facing side of the platform (schedule management, seat allocation, pricing for carriers). The job post lists four product areas:

1. **Transit schedule system** — hundreds of thousands of bus departures daily; used by operations teams, revenue managers, planners.
2. **Ticket booking backend** — seat allocation across stops, reallocation on vehicle swaps, bus merges, reinforcements. This is inventory management with shared resources — classic concurrency and consistency problem.
3. **Revenue management** — dynamic pricing on time-to-departure, demand, remaining capacity; carrier-controlled pricing, A/B testing, automated algorithms (airline/yield-management style).
4. **Data transformation layer** — models, incremental pipelines, semantic definitions, data quality (testing, contracts, freshness checks, monitoring).

Tech signals from the post: **Python** backend, distributed systems, event-driven architecture, outbox/idempotency/retries/eventual consistency, deep SQL (ACID, indexing, locking, partitioning, sharding), cloud infra, messaging systems, distributed databases. Plus DX: CI, PR reviews, AI-assisted development.

### Candidate snapshot (from CV)

- **Profile:** Senior Software Engineer — Backend & AI Systems. 8+ years. Based South Tangerang, Indonesia (UTC+7), open to relocation.
- **Stack:** Go (primary), Ruby, Python, PHP. FastAPI + FastMCP on the Python side. PostgreSQL, MySQL, MongoDB, Redis, DynamoDB, Elasticsearch. Kafka, RabbitMQ. AWS/GCP/Azure, Kubernetes, Terraform. Datadog, OTel, Jaeger.
- **Current:** Careem (Uber), Dubai — Nov 2025–present. KSA restaurant-network integration end-to-end (catalog sync, order lifecycle, promotions); reliability layer for UAE government (CERT) event relay — 3-way error classifier + SQS FIFO retry with ordering gate.
- **Before:** Telkom Indonesia, Lead Backend (Feb 2023–Nov 2025) — AI Proxy & MCP Orchestrator powering TelkomGPT, RBAC multi-tenant isolation, AI-assisted GitLab MR reviewer, ~80% memory reduction via stateless migration + K8s autoscaling, Board of Experts, p99 query optimization, full observability stack, mentoring.
- **Earlier:** RRQ Guild (first engineer, 0→1 platform + Auth0 + AWS), Tanihub (FIFO stock reservation, payment microservice), Hubbedin (Singapore, part-time), Bukalapak (O2O tribe, 100M+ users, on-prem→GCP migration).
- **Education:** BSc MIS, Ciputra University, GPA 3.8, Best Student. 2× hackathon winner.

**Direct mappings to the job post — use these in every round:**

| Job post asks | Your evidence |
|---|---|
| B2B marketplace integration layer (their core product) | **Careem Enabler** — KSA partner onboarding, catalog sync, order lifecycle, promotions sync. Same system shape as carrier↔retailer integration: external partner APIs, eventual consistency, dropped event = money lost. Domain differs, architecture doesn't |
| Seat allocation across stops, reallocation (swaps, merges) | **Tanihub FIFO stock reservation** — real-time shared inventory reservation at scale; nearest analog to seat inventory |
| Event-driven reliability, retries, eventual consistency | **Careem CERT relay** — error classification, FIFO ordering gate, exponential backoff; fixed silent drop of regulated events |
| AI-assisted development, tooling, self-service | **Telkom AI Proxy + MCP Orchestrator + AI MR reviewer** — you built exactly what the post wants "driven" |
| Deep database expertise | p99 query optimization via indexing + execution plans (Telkom); Redis caching at Bukalapak scale |
| Ownership, full lifecycle, standards | Board of Experts at Telkom; first engineer at RRQ Guild (0→1); led on-prem→GCP migration |
| Ambiguous requirements → pragmatic solutions | Partner integration at Careem (KSA network onboarding automation) |

## 2. Likely interview process (estimate)

Typical for a Series C European startup hiring a senior engineer:

1. **HR screen** (~30 min) — this stage. Motivation, logistics, comp, communication check.
2. **Hiring manager or technical intro** (~45 min) — background deep-dive, system ownership stories.
3. **Technical interviews** (1–2 rounds) — likely system design around their domain (seat allocation, schedules, pricing) + coding in Python or language-agnostic.
4. **Data/systems deep-dive** — databases, pipelines, consistency.
5. **Stakeholder/product interview** — translating ambiguous requirements; the post weights this heavily.
6. **Founder/final + references.**

Confirm the exact map with Aulia in the HR call (see §6).

## 3. HR screen: questions and answers

### 3.1 "Tell me about yourself" (the 90-second version)

> I'm a backend engineer with 8+ years building and operating distributed systems — design, deployment, monitoring, and on-call, not just implementation. Right now I'm at Careem, Uber's super-app, where I own partner-facing integrations for KSA restaurant networks and built the reliability layer for a government event relay — error classification, FIFO-ordered retries, no silent drops. Before that, as a Lead Backend Engineer at Telkom Indonesia, I built an AI platform from scratch powering multiple business units, drove an 80% memory reduction through a stateless rearchitecture, and sat on the company-wide Board of Experts setting backend standards. Earlier I did high-scale consumer work at Bukalapak, 100M+ users, and — most relevant to this role — at Tanihub I built a FIFO stock reservation system for real-time shared inventory at scale. I'm looking for a role where engineering decisions directly move the product, and owning systems like schedules, seat allocation, and pricing end-to-end is exactly that.

**Structure:** present → recent proof of seniority → most relevant domain story (Tanihub inventory ≈ seat allocation) → why this role. Keep under 2 minutes, do not recite the whole CV.

### 3.2 "Why Distribusion?"

Key: connect **their domain** to **systems problems you enjoy**, and show you read the news page. Pick 2–3 of these hooks:

- **Hard, real systems problems**: seat allocation across stops with vehicle swaps and merges is genuinely difficult distributed-state management — shared mutable inventory, concurrency, compensation logic. That's more interesting than CRUD.
- **Scale**: hundreds of thousands of departures daily, millions of travellers, external partners (Google, Booking) depending on the API. High stakes, real failure modes.
- **Growth moment**: 10x growth and a fresh $80M Series C means the systems built for the earlier scale are being rebuilt — a senior engineer gets to shape that rather than maintain a finished machine.
- **End-to-end ownership**: the post explicitly says "own and evolve core systems end-to-end" and values people who challenge requirements. That matches how I like to work.
- Optional warmth: ground transport is the sustainable travel choice; DB tender and rail expansion show the company winning trust of national operators — that momentum is attractive.

Model answer (short):

> Three things. First, the problems: seat allocation and dynamic pricing under real-world disruption — vehicle swaps, merges, delays — are some of the hardest distributed-state problems in travel, and they map directly onto work I've done. At Tanihub I built FIFO stock reservation for shared inventory at scale; at Careem I build reliability layers for event flows where a dropped event is a real business incident. Second, the timing: 10x growth in a year and a Series C means the platform is being rebuilt for the next order of magnitude — a senior engineer can shape that instead of babysitting it. Third, the team's position: the Deutsche Bahn tender and partnerships with Google Maps and Booking.com show the network effect is real. I want to build infrastructure that a whole industry runs on.

### 3.3 "Why are you leaving your current role?" / "Why now?"

Rules: no negativity about Careem; frame as pull, not push.

> Careem has been a strong chapter — I'm working on partner integrations and event reliability at real scale, and Uber's engineering bar is high. But my scope is one integration domain within a giant org. I want to own a core system end-to-end again, the way I did at Telkom and RRQ Guild, where my decisions move the product's business metrics directly. Distribusion's stage — post-Series C, 10x growth, core operator systems being rebuilt — is exactly where a senior engineer has maximum leverage. And honestly, the problem is a good fit: allocation under real-world disruption is the kind of system I enjoy most.

### 3.4 "The role is Python; your background is Go. How do you feel about that?"

**Expect this. It is the #1 HR-screen risk for this job.** Your CV already lists Python (FastAPI, FastMCP), so the answer is strong — never present yourself as a Python beginner:

> My primary production language is Go, but Python is part of my daily toolkit — I've used it with FastAPI for services and FastMCP for the MCP orchestrator I built at Telkom, which is production software serving multiple business units. The core of this role is transferable regardless of language: transaction management, locking, idempotency, outbox patterns, event-driven design — none of that lives in a language. I'd ramp fully on the Python ecosystem's conventions — testing stack, async patterns, typing — and I'm comfortable being judged on that ramp.

Do **not** say "Python is easy, I'll learn it in a weekend." Do **not** over-apologize. Lead with FastAPI/FastMCP production usage, then the patterns argument. If asked for detail in a later round, be ready to name what you'd re-learn: pytest, pydantic, async/await idioms, SQLAlchemy vs Django ORM conventions, ruff/mypy toolchain.

### 3.5 Logistics questions — prepare exact answers

HR screens fail on vague logistics. Have one-line answers ready:

| Question | How to answer |
|---|---|
| Work authorization | You're an Indonesian citizen, currently employed by Careem (Dubai entity) remotely from Indonesia. For **Remote**: no sponsorship needed — ask how they contract (German entity + EOR, contractor, or local entity). For **Berlin**: you'd need a German work visa / EU Blue Card (salary threshold ~€48k+; a senior offer clears it) — say you're open to relocation, CV says so, and ask if they sponsor. State this plainly, no surprises later. |
| Notice period | **Fill in exact number before the call.** Careem notice period + any leave accrued. Give an exact earliest-start date, e.g. "I can start [date]." An exact date reads as organized; "a couple of months" reads as vague. |
| Remote vs Berlin | Decide your preference before the call and give one line + reason. Remote: "Remote from Indonesia works well — I already work with a Dubai-headquartered org across time zones; I can anchor [e.g., 14:00–22:00 WIB] for solid CET overlap." Relocation: "Open to relocating to Berlin — my CV says so." |
| Time-zone overlap | WIB (UTC+7) is 5–6 hours behind CET. Jakarta afternoon 14:00–21:00 = Berlin morning 09:00–16:00 (summer). That's a full-day overlap. Preempt the worry; don't wait for them to raise it. |
| Salary expectations | See §4. Have a number and a range. Never "I'm flexible" as a whole answer. |
| Competing processes | Fine to say you're in other processes; adds urgency if framed as demand. Optional. |
| Other offers/deadlines | If real, mention — it speeds things up. |

### 3.6 "What do you know about us?"

Use §1 compressed to 30 seconds:

> You're the B2B ground transportation marketplace — the technology layer connecting 250+ bus, rail, and ferry carriers in 70+ countries to retailers like Google Maps and Booking.com. You run the full chain from search to settlement, grew 10x last year, and raised an $80M Series C led by TQ Ventures in 2024. Recently you've been expanding hard in rail — the Deutsche Bahn tender, KAI in Indonesia, WESTbahn — which tells me the network effect is compounding.

### 3.7 Behavioral softballs HR asks

- **Strengths**: ownership + turning ambiguity into shipped systems. Story: RRQ Guild — first engineer, no specs, no infra, shaped backend strategy with Head of Product and launched the platform 0→1. Or Telkom AI platform: vague org need ("we need internal AI") turned into a production multi-tenant platform. Both mirror the post's "turn ambiguous problems into clear requirements."
- **Weakness**: real but non-fatal, with mitigation. E.g., "I go deep on technical elegance before checking business value — I now force an early written trade-off doc with stakeholders." Avoid fake weaknesses.
- **Conflict / challenging requirements story**: pick one of — Board of Experts standardization where teams initially resisted new practices; or a Careem integration where a partner's proposed flow had a flaw you pushed back on. Structure: challenge → alternative proposed → committed to the outcome. The post explicitly values "challenging requirements" and "proposing alternative approaches."
- **Mentoring story**: Telkom — mentored juniors on Go, clean architecture, observability. Outcome-framed.
- **Why senior, not staff**: scope story — Board of Experts and platform ownership show org-level influence, and you want maximum hands-on ownership of core systems, which is what "Senior Product Engineer" here means.
- **AI tooling question (likely, it's in the post)**: strongest card. You didn't just *use* AI tools — you built the platform others use: AI Proxy, MCP Orchestrator, AI-assisted GitLab MR reviewer, token-usage cost tracking. Frame: "I've been on both sides — building AI-assisted developer infrastructure and using it daily."

## 4. Salary: what to say

No public levels.fyi/Glassdoor data specific to Distribusion was found. Use Berlin senior-engineer market rates as the anchor and label them as market data:

- **Berlin senior backend (Series B–D, well-funded), relocating to Berlin:** roughly **€80k–€110k base**, plus equity. Top-of-band for strong senior/profile-match candidates reaches ~€115–120k. EU Blue Card eligibility clears easily at this level.
- **Remote from Indonesia:** different structure entirely. European companies typically contract via **EOR (Deel/Remote.com) with location-adjusted pay** or as a **B2B contractor**. Location-adjusted remote comp for senior roles in Indonesia commonly lands at **30–60% of the Berlin equivalent**, but strong remote-first companies pay closer to global bands. **Do not quote a Berlin number if staying remote** — anchor instead to your current Careem comp + uplift, and make contract structure an explicit question.
- **If they ask first** (they usually do): give a range whose **bottom is a number you'd actually accept**, and if remote, make structure part of the answer:

  > "It depends on the structure — remote contract versus Berlin employment. For Berlin, market for senior backend at this stage is roughly €[X]–€[Y] base plus equity. For remote, I'd benchmark against my current total comp and the location structure you use. What's budgeted for this position?"

- **If they push for one number**: give the bottom of your real range, not your dream number — the range gets negotiated down, the single number gets negotiated up.
- **Ask in return**: "Is there a comp band for this level? How does equity work — option pool, vesting? Do you adjust by location for remote, and via what mechanism — EOR or contractor?" — normal HR-stage questions.
- Fill in X/Y from your current Careem total comp + target uplift (typical: +15–25% to switch; more if relocation to Berlin, where cost of living justifies the full band).

## 5. Risks to manage in the screen

1. **Python depth** — softened: your CV shows FastAPI + FastMCP production work, but your large-scale production record is Go. Answer per §3.4: lead with production Python, then transferable-patterns argument, then commit to the ramp. Expect the hiring manager to probe deeper than HR does.
2. **"Product engineer" is not "product manager"** — if the title confuses the conversation, clarify: engineering role with product accountability, building the operator-facing systems (IMS).
3. **Remote structure and comp mismatch** — biggest practical risk if you stay in Indonesia. Berlin-band expectations against location-adjusted remote pay leads to stalled offers. Resolve contract mechanism (EOR vs contractor) and band early, in the very first screen.
4. **Time-zone overlap** — preempts itself if you volunteer your CET-overlap hours (§3.5). Their teams are global and remote-first, so this is low-risk, but never make them raise it.
5. **Know the difference between their two audiences**: they have a B2B API for retailers *and* internal/operator tools (IMS). This role is the operator side. Confusing these signals weak domain reading.

## 6. Questions to ask HR (pick 4–6)

Good HR-stage questions are about process, team, and culture — technical depth comes later.

**Process and role:**
1. "What does the full interview process look like, and what are the next steps and timeline after this call?"
2. "Who would I report to, and how is the engineering team structured — how many engineers, and how are product teams organized?"
3. "What does 'IMS' cover as a team — is this the schedule/booking/pricing product area for operators?"
4. "What's the strongest technical challenge the team is facing right now — is the priority more on scaling what exists or building new product areas?"
5. "Is this role replacing someone, backfilling after growth, or a new position?" (Reveals whether it's a fire-hire or expansion.)

**Culture and work:**
6. "How do hybrid/remote work — how often does the team meet in Berlin, and how async is the day-to-day?"
7. "How does the company use AI-assisted development — the post mentions it as an area to drive. Is there existing tooling?" (Shows you read the post carefully.)
8. "What does success look like at 6 and 12 months for this hire?"

**Logistics:**
9. "What's the comp band for this role, and does equity form part of the package?"
10. "If remote, is the contract through a German entity or a local/EOR arrangement? How are benefits handled?"

Avoid asking anything answerable from the website.

## 7. HR-screen checklist (night before)

- [ ] 90-second intro rehearsed (§3.1), ends with Tanihub-inventory → why-this-role
- [ ] Why-Distribusion answer: 3 hooks (allocation problem mapped to Tanihub/Careem work / 10x + Series C / DB tender & rail momentum)
- [ ] Python answer ready: FastAPI/FastMCP production evidence first (§3.4)
- [ ] **Exact Careem notice period + earliest start date** — fill in before call
- [ ] Work-auth one-liner: remote = no sponsorship, ask EOR/contractor structure; Berlin = relocation + Blue Card (§3.5)
- [ ] Salary: two anchors ready — Berlin band vs remote-adjusted structure (§4)
- [ ] Company numbers memorized: 250+ carriers, 2,000+ retailers, 70+ countries, $80M Series C (TQ Ventures, Sep 2024), 10x growth
- [ ] 5 questions from §6 written down
- [ ] Recruiter's name (Aulia) and quiet room, camera on, good audio
- [ ] Check Aulia's and the hiring manager's LinkedIn before the call

## 8. Useful links

- [Company site](https://www.distribusion.com/) · [About](https://www.distribusion.com/about-us) · [News](https://www.distribusion.com/news)
- [Series C announcement](https://www.distribusion.com/news/distribusion-announces-%2480m-series-c-led-by-tq-ventures-to-drive-global-expansion-and-to-double-down-on-advanced-retail-technology-for-its-partners)
- [DB tender news](https://www.distribusion.com/news/distribusion-wins-tender-to-support-the-development-of-a-new-multi-carrier-sales-solution-for-deutsche-bahn)
- [Careers portal](https://careers.distribusion.com/)
- [LinkedIn](https://www.linkedin.com/company/distribusion/) — check who you're talking to and recent posts before the call

After the technical screen is scheduled, build system-design practice for their domain here: seat allocation across stop-pairs, schedule ingestion at scale, dynamic pricing engine. That material belongs in a separate file once the stage is confirmed.
