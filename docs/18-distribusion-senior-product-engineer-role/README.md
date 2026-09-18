# Distribusion Technologies — Senior Product Engineer, IMS

Targeted prep for the **Senior Product Engineer, IMS** role (Remote/Berlin, Hybrid) at Distribusion Technologies. This pack covers the **HR screen** first — that is the immediate next stage — plus company research you can use if asked.

> **Evidence note:** Distribusion does not publish its interview process. The stage map and salary figures below are estimates from public data (company site, news page, job post, Berlin market norms). Salary numbers are market estimates, not offer data. Verify with the recruiter.

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

Adapt to your actual résumé, but hit this arc: backend engineer → distributed systems and data-intensive services → ownership across full lifecycle → product impact. Keep it under 2 minutes.

> I'm a backend engineer with [N] years building and operating services in distributed systems — design, deployment, monitoring, and on-call, not just implementation. Most recently at [company] I owned [system], which handled [scale number], where I worked on [1–2 concrete things: e.g., seat/inventory-style allocation, event-driven pipelines, database performance]. I've also done a lot of [DX work: CI, testing strategy, tooling]. I'm looking for a role where engineering decisions directly move the product — which is exactly what this role is: schedules, seat allocation, and pricing are the product, and the engineering quality is the business.

**Rule:** end with "why this role", not with your current job. That tees up the next question.

### 3.2 "Why Distribusion?"

Key: connect **their domain** to **systems problems you enjoy**, and show you read the news page. Pick 2–3 of these hooks:

- **Hard, real systems problems**: seat allocation across stops with vehicle swaps and merges is genuinely difficult distributed-state management — shared mutable inventory, concurrency, compensation logic. That's more interesting than CRUD.
- **Scale**: hundreds of thousands of departures daily, millions of travellers, external partners (Google, Booking) depending on the API. High stakes, real failure modes.
- **Growth moment**: 10x growth and a fresh $80M Series C means the systems built for the earlier scale are being rebuilt — a senior engineer gets to shape that rather than maintain a finished machine.
- **End-to-end ownership**: the post explicitly says "own and evolve core systems end-to-end" and values people who challenge requirements. That matches how I like to work.
- Optional warmth: ground transport is the sustainable travel choice; DB tender and rail expansion show the company winning trust of national operators — that momentum is attractive.

Model answer (short):

> Three things. First, the problems: seat allocation and dynamic pricing under real-world disruption — vehicle swaps, merges, delays — are some of the hardest distributed-state problems in travel, and I want to work on exactly that. Second, the timing: 10x growth in a year and a Series C means the platform is being rebuilt for the next order of magnitude; a senior engineer can shape that instead of babysitting it. Third, the team's position: the Deutsche Bahn tender and partnerships with Google Maps and Booking.com show the network effect is real. I want to build infrastructure that a whole industry runs on.

### 3.3 "Why are you leaving your current role?" / "Why now?"

Rules: no negativity about current employer; frame as pull, not push. Match to this role's gaps.

> I've grown a lot at [company] — [one real achievement]. But [the scale/impact ceiling: e.g., our traffic plateaued / my scope narrowed to one service / there's no room to own a product end-to-end]. I want the next challenge to be owning a system where my decisions directly move business metrics. Distribusion's stage — post-Series C, 10x growth, core systems being rebuilt — is exactly where a senior engineer has maximum leverage.

### 3.4 "The role is Python; your background is [Go/PHP/etc.]. How do you feel about that?"

**Expect this. It is the #1 HR-screen risk for this job.** Prepare a crisp, honest, confident answer:

> My primary language is [X], but the transferable core is what this role actually tests: transaction management, locking, indexing, idempotency, outbox patterns, event-driven design — none of that lives in a language. I've [real evidence: worked in multiple languages / shipped a production service in a second language / picked up X in weeks at Y]. I'd ramp on Python's ecosystem before day one — I'd start now with FastAPI/async patterns and the testing stack — and I'm comfortable being judged on that ramp.

Do **not** say "Python is easy, I'll learn it in a weekend." Do **not** over-apologize. If you already have any production Python, name the specific project. If not, show you've started: mention reading through their engineering content or building a small service.

### 3.5 Logistics questions — prepare exact answers

HR screens fail on vague logistics. Have one-line answers ready:

| Question | How to answer |
|---|---|
| Work authorization | State it plainly: where you can work, current visa status, whether you need sponsorship. If the role is Remote and you need them to sponsor a German visa, say so now — surprises later kill offers. |
| Notice period | Exact date you could start. German market norm is 3 months; if you have less, say it — it's a competitive advantage. |
| Remote vs Berlin | They offer both. Know your preference and reason. If remote: confirm your time zone overlap with CET (their teams are global, so this is normal, but state your hours). |
| Salary expectations | See §4. Have a number and a range. Never "I'm flexible" as a whole answer. |
| Competing processes | It's fine to say you're in other processes; adds urgency if framed as demand. Optional. |
| Other offers/deadlines | If real, mention — it speeds things up. |

### 3.6 "What do you know about us?"

Use §1 compressed to 30 seconds:

> You're the B2B ground transportation marketplace — the technology layer connecting 250+ bus, rail, and ferry carriers in 70+ countries to retailers like Google Maps and Booking.com. You run the full chain from search to settlement, grew 10x last year, and raised an $80M Series C led by TQ Ventures in 2024. Recently you've been expanding hard in rail — the Deutsche Bahn tender, KAI in Indonesia, WESTbahn — which tells me the network effect is compounding.

### 3.7 Behavioral softballs HR asks

- **Strengths**: ownership + ambiguity → pick one story where you turned a vague problem into a shipped system (mirrors their "turn ambiguous problems into clear requirements" line).
- **Weakness**: real but non-fatal, with a mitigation. E.g., "I go deep on technical elegance before checking business value — I now force an early written trade-off doc with stakeholders." Avoid fake weaknesses.
- **Conflict/misalignment story**: have one STAR story ready where you challenged a requirement, proposed an alternative, and committed to the outcome — the post explicitly values "challenging requirements."
- **Why senior, not staff**: scope story — you lead projects and influence direction, and you want maximum hands-on ownership of core systems, which is what "Senior Product Engineer" here means.

## 4. Salary: what to say

No public levels.fyi/Glassdoor data specific to Distribusion was found. Use Berlin senior-engineer market rates as the anchor and label them as market data:

- **Berlin senior backend (Series B–D, well-funded):** roughly **€80k–€110k base**, with equity. Top-of-band for strong senior/profile-match candidates reaches ~€115–120k. Remote contracts outside Germany are often structured differently (contractor or local-entity employment).
- **If they ask first** (they usually do): give a range whose **bottom is a number you'd actually accept**:

  > "Based on the Berlin market for senior backend engineers at this stage, I'm looking at roughly €[X]–€[Y] base, plus equity. I'm flexible on structure for the right role — what range is budgeted for this position?"

- **If they push for one number**: give the bottom of your real range, not your dream number — the range gets negotiated down, the single number gets negotiated up.
- **Ask in return**: "Is there a comp band for this level? Equity? Any location-based adjustment for remote?" — these are normal HR-stage questions.
- Fill in X/Y from your current comp + target uplift (typical: +15–25% to switch, more if you're leaving equity or seniority on the table).

## 5. Risks to manage in the screen

1. **Python gap** — prepare §3.4 cold. If HR flags it to the hiring manager, your answer determines whether you get a technical round.
2. **"Product engineer" is not "product manager"** — if the title confuses the conversation, clarify: engineering role with product accountability, building the operator-facing systems (IMS).
3. **Time-zone / remote fit** — they're remote-first with global teams; preempt the "will you overlap with CET" worry with your working-hours commitment.
4. **Know the difference between their two audiences**: they have a B2B API for retailers *and* internal/operator tools (IMS). This role is the operator side. Confusing these signals weak domain reading.

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

- [ ] 90-second intro rehearsed, ends with why-this-role
- [ ] Why-Distribusion answer: 3 hooks (allocation problem / 10x + Series C / DB tender & rail momentum)
- [ ] Python answer ready with concrete ramp evidence
- [ ] Exact notice period and earliest start date
- [ ] Work authorization and remote/time-zone answer, one line each
- [ ] Salary range with real bottom number
- [ ] Company numbers memorized: 250+ carriers, 2,000+ retailers, 70+ countries, $80M Series C (TQ Ventures, Sep 2024), 10x growth
- [ ] 5 questions from §6 written down
- [ ] Recruiter's name (Aulia) and quiet room, camera on, good audio

## 8. Useful links

- [Company site](https://www.distribusion.com/) · [About](https://www.distribusion.com/about-us) · [News](https://www.distribusion.com/news)
- [Series C announcement](https://www.distribusion.com/news/distribusion-announces-%2480m-series-c-led-by-tq-ventures-to-drive-global-expansion-and-to-double-down-on-advanced-retail-technology-for-its-partners)
- [DB tender news](https://www.distribusion.com/news/distribusion-wins-tender-to-support-the-development-of-a-new-multi-carrier-sales-solution-for-deutsche-bahn)
- [Careers portal](https://careers.distribusion.com/)
- [LinkedIn](https://www.linkedin.com/company/distribusion/) — check who you're talking to and recent posts before the call

After the technical screen is scheduled, build system-design practice for their domain here: seat allocation across stop-pairs, schedule ingestion at scale, dynamic pricing engine. That material belongs in a separate file once the stage is confirmed.
