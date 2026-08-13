# Mango Voice — Senior Backend Engineer

This section is a targeted prep pack for one specific posting: a **Senior Backend Engineer** role at Mango Voice, a cloud VoIP/SMS/collaboration company, on a **3-5 person team**, Golang-primary, AWS-hosted, with on-call and support-escalation duties, and an explicit potential focus on **AI-driven features and agentic systems** in the **Customer Experience** or **Integrations** domains. The distinctive shape of this JD: it's a *telephony platform* backend role wearing an *AI-platform* hat — the interviewer will test both, and the overlap (real-time media plus real-time AI) is where the hardest questions live. Read this file first — it decodes the JD, maps every bullet to material already in this repo, fills the gaps specific to *this* posting (VoIP protocols, Shape Up, HIPAA, agentic frameworks), and ends with a graded question bank and behavioral prep tuned to the exact phrasing used.

## 1. Decoding the JD

| JD phrase | What it really means | Where interviewers probe it |
| --- | --- | --- |
| "small team of 3-5 engineers... handle on-call duties and support escalations" | Full-stack ownership per engineer — you design, build, deploy, and get paged for the same features. No separate SRE team to hand off to. | Behavioral questions about owning something end-to-end; expect an incident/troubleshooting scenario, not just a coding round. |
| "cloud native applications for viewing, configuring, and using the Mango Voice Business VoIP software system" | The product is an admin/config layer *on top of* a telephony core — you're building the control plane for a VoIP system, not necessarily the media-plane switch itself, but you need to understand what that control plane is controlling. | Section 3.2 below — VoIP concepts you need even if you never write SIP-handling code directly. |
| "AI-driven features and agentic systems, particularly... Customer Experience or Integrations domains" | This is the headline differentiator. Two concrete shapes: (a) AI *on* the call itself — transcription, summarization, live agent-assist; (b) AI/automation *around* the call — routing data into and out of third-party systems. Both get asked about. | Section 3.5 and the scenario in Section 4. |
| "data integrations (CRM/PMS/EHR)" | PMS = property management system, EHR = electronic health record. Mango Voice sells into multiple verticals including healthcare — this is why HIPAA shows up as a preferred qualification, not a throwaway line. | Section 3.6 — HIPAA is testable here even though it's listed as "preferred." |
| "unified Conversational AI and Communications Platform" | The architectural vision: telephony infrastructure and an LLM/agent layer are being merged into one platform, not bolted together. Expect a system-design question that spans both. | Section 4. |
| "Golang as the primary language... high-volume, production-ready systems" | Straightforward — this is asked deeply and often. | [Go section](../01-languages/go/README.md) — no gap, just depth. |
| "Shape Up framework" | A specific, opinionated delivery methodology (Basecamp's), not generic "Agile." If you've only done Scrum, say so honestly but show you understand the *appetite vs. estimate* difference. | Section 3.3. |
| "AWS (EC2, S3, RDS, Lambda, CloudWatch)" | Standard AWS fundamentals — no exotic services named, which suggests a fairly conventional compute/storage/DB/serverless/monitoring stack rather than a heavy K8s shop. | [AWS section](../04-aws/README.md). |
| "microservices principles, API design (REST/gRPC), scalability" | Standard system-design and API-design territory. | [System Design](../06-system-design/README.md), [API Design](../11-api-design/README.md). |
| "relational SQL and NoSQL datastores" | Likely RDS (Postgres/MySQL) for the relational store plus DynamoDB or similar for high-volume call-event/CDR-style data — NoSQL fits call detail records and time-series-ish telephony data much better than a relational table that grows by the millisecond. | [MySQL](../03-databases/mysql/README.md) / [PostgreSQL](../03-databases/postgresql/README.md), plus the DynamoDB coverage in [AWS](../04-aws/02-networking-and-databases.md). |
| "Experience with Python" (preferred) / "5+ years TypeScript" (poster-added requirement) | Two different signals worth noticing: Python-preferred likely maps to AI/ML tooling and scripting (see Section 3.5's LangChain/LlamaIndex note — both are Python-first). TypeScript appearing as a hard *requirement* despite Go being "the primary language" for the role is inconsistent with the rest of the JD — it may be boilerplate from a recruiter template, or it may signal a real Node/TS layer (an admin frontend, or a Node-based AI orchestration service) that the JD body doesn't mention. **Ask about this directly** — see Section 7. | Section 3.4. |
| "VoIP technologies (SIP, RTP, WebRTC) or messaging (SMS/MMS/RCS)" (preferred) | Listed as preferred, not required, but for a company whose product *is* VoIP, expect at least conceptual questions even from an interviewer who says "don't worry, we'll teach you the domain." | Section 3.2 (VoIP) and 3.7 (SMS/MMS/RCS) — the single biggest content gap in this pack. |
| "Terraform, Ansible, Bash... CI/CD, server management, migrations" | Standard IaC/automation — no surprises. | [Terraform](../05-devops/terraform/README.md), [Ansible](../05-devops/ansible/README.md), [GitHub Actions](../05-devops/github-actions/README.md). |
| "Healthcare Experience... HIPAA compliance" (preferred) | Given the EHR integration bullet above, this is more load-bearing than a typical "preferred" line. | Section 3.6. |
| "Mentor and provide technical guidance to more junior engineers" | Senior-level expectation on a small team — you're not just an IC, you're the technical anchor for the 3-5 person group. | Section 5 behavioral prep. |

## 2. Coverage map — what's already in this repo

Read in this order; each links to the file that covers it in depth. This section only covers what's **not** already handled elsewhere.

| JD requirement | Repo coverage | Read this first if... |
| --- | --- | --- |
| Golang for production, high-volume systems | [Go](../01-languages/go/README.md) — core concepts, concurrency, interfaces/errors, testing/performance | you haven't written Go under real load before |
| AWS (EC2, S3, RDS, Lambda, CloudWatch) | [AWS](../04-aws/README.md) — compute/storage, networking/databases, IAM/security/monitoring | rusty on Lambda cold starts or RDS failover specifics |
| Microservices, API design (REST/gRPC), scalability | [System Design](../06-system-design/README.md), [API Design](../11-api-design/README.md) | you haven't designed a public API contract or reasoned about sharding before |
| Relational + NoSQL datastore design | [MySQL](../03-databases/mysql/README.md), [PostgreSQL](../03-databases/postgresql/README.md), DynamoDB in [AWS networking/databases](../04-aws/02-networking-and-databases.md) | — |
| AI-driven workflows, LLM integration, RAG | [AI & LLM Integration](../14-ai-llm-integration/README.md) | before Section 3.5 below — that section assumes you've read this one |
| Data integrations, webhooks, third-party CRM/PMS/EHR sync | [API Design — operations](../11-api-design/03-api-operations.md) (idempotency, webhooks, versioning), [Messaging & Event Streaming](../10-messaging-and-event-streaming/README.md) (outbox, CDC, event-driven sync) | you haven't designed a reliable webhook-delivery or two-system-sync pipeline before |
| On-call, reliability, production troubleshooting | [Distributed Systems — correctness in practice](../13-distributed-systems/03-correctness-in-practice.md) (retries, idempotency, gray failures), [Datadog](../05-devops/datadog/README.md) if they use it for observability | — |
| CI/CD, IaC, server automation | [Terraform](../05-devops/terraform/README.md), [Ansible](../05-devops/ansible/README.md), [GitHub Actions](../05-devops/github-actions/README.md) | — |
| Auth/security baseline (relevant to HIPAA access-control questions) | [Security & Auth](../12-security-and-auth/README.md) | before Section 3.6 |
| Behavioral / mentoring / STAR | [Behavioral](../08-behavioral/README.md) | before Section 5 below |

## 3. The gaps — content specific to this posting

### 3.1 Framing the "Conversational AI and Communications Platform" pitch

When asked "why this role" or "walk me through your background," name the architectural thesis explicitly rather than talking about Go and AI as two separate interests: *"I'm interested in the point where real-time media and real-time AI meet — a call isn't just audio to route anymore, it's a stream you can transcribe, summarize, and act on while it's still happening, and that changes the reliability and latency requirements on both sides."* This signals you understand what makes this specific platform hard, not just that you know Go and know AI.

### 3.2 VoIP fundamentals: SIP, RTP, and WebRTC

This is the material a generic backend-interview-prep curriculum doesn't cover and the one most likely to catch you flat-footed, even in a "we'll teach you the domain" interview. You don't need protocol-implementer depth — you need enough to reason about where AI features plug in and what breaks.

**SIP (Session Initiation Protocol, RFC 3261) — the signaling layer.** SIP sets up, modifies, and tears down calls; it does not carry the audio itself. Core methods: `INVITE` (start a session), `ACK` (confirm), `BYE` (end), `CANCEL`, `REGISTER` (a phone/softphone announces "I'm reachable at this address" to a registrar). Responses follow HTTP-like status classes: `1xx` provisional (`100 Trying`, `180 Ringing`), `2xx` success (`200 OK`), `3xx` redirect, `4xx` client error (`401`/`407` trigger SIP digest authentication — a challenge-response handshake, not a bearer token), `5xx`/`6xx` server/global failure. A basic call flow: `INVITE` → `100 Trying` → `180 Ringing` → `200 OK` → `ACK` → media flows directly between endpoints → `BYE` → `200 OK`. The `INVITE`/`200 OK` bodies carry **SDP** (Session Description Protocol) to negotiate codecs and media transport addresses — this negotiation is why "the call connected but there's no audio" is usually an SDP/codec mismatch or a NAT problem, not a SIP signaling failure.

**RTP (Real-time Transport Protocol, RFC 3550) — the media layer.** Once SIP sets up the session, the actual audio/video travels over RTP, almost always over UDP (loss-tolerant, latency-sensitive — a retransmitted audio packet arriving late is worse than a dropped one). Each RTP packet carries a sequence number (detects loss/reordering) and a timestamp (drives the jitter buffer that smooths out network timing variance before playback). **RTCP** is the sibling control protocol — periodic sender/receiver reports carrying loss rate, jitter, and round-trip time, which is where call-quality metrics (and MOS — Mean Opinion Score — estimates) come from. **SRTP** is the encrypted variant, mandatory in WebRTC and increasingly standard in SIP trunking.

**WebRTC — browser/app-native real-time media.** Three problems WebRTC solves that plain SIP/RTP don't address for a browser client: NAT traversal, encryption by default, and a JS-native API. **ICE** (Interactive Connectivity Establishment, RFC 8445) finds a viable path between two peers behind NATs/firewalls, using **STUN** servers (cheap — just tell a client its public IP:port) and **TURN** servers (expensive — relay all media through the server when a direct path isn't possible, e.g. symmetric NAT). Media is DTLS-SRTP encrypted by default — there is no unencrypted WebRTC media mode. WebRTC does *not* mandate a signaling transport (how peers exchange SDP offers/answers and ICE candidates before a connection exists) — that part you build yourself, commonly over a WebSocket. `RTCPeerConnection` is the core browser API; `RTCDataChannel` carries arbitrary non-media data over the same peer connection.

**What this means for backend design questions:**

- Mango Voice's platform almost certainly bridges traditional SIP/PSTN trunking with browser/app WebRTC clients through a **media server** (Asterisk, FreeSWITCH, Kamailio, or a managed SIP-trunking provider) that handles the SIP-to-WebRTC gateway function — you are very unlikely to be asked to implement SIP parsing yourself, but you should know this component exists and roughly what it does.
- **Adding live AI to a call means tapping the media stream**, not the signaling. A media server can "fork" or "tee" the RTP audio to a separate consumer (a transcription pipeline) without affecting the primary call path — this separation is the answer to "how do you add a live transcription feature without risking call quality," and it's the same principle as not putting a slow consumer in the critical path, covered generically in [caching & microservices](../06-system-design/02-caching-and-microservices.md).
- **Latency budgets are brutal for live AI-on-a-call** (real-time transcription, a live "agent assist" suggestion, or a conversational AI actually speaking on the call): streaming STT → LLM → TTS round trips need to land in the hundreds of milliseconds to feel conversational, not the multi-second budgets acceptable for an async summarization job. Naming this constraint unprompted is a strong signal.
- **Call quality metrics** (jitter, packet loss %, MOS) come from RTCP — if asked "how would you know call quality is degrading before customers complain," the answer routes through RTCP stats aggregation, not application-level logging.
- **SIP trunking vs. WebRTC** is the "which protocol for which client" decision: PSTN/carrier connectivity uses SIP trunks; browser and mobile-app clients use WebRTC; a unified platform needs a gateway component translating between them.

### 3.3 Shape Up: what it actually is

Not generic "Agile" — a specific methodology from Basecamp (Ryan Singer's free book, *Shape Up*), built around **fixed time, variable scope** — the inverse of Scrum's fixed-scope, variable-time estimation model.

- **Six-week cycles**, followed by a **2-week cooldown** (no scheduled work — bug fixes, cleanup, exploration, whatever individuals want).
- **Shaping** happens before a cycle starts, usually by senior people: turning a raw idea into a **pitch** — the problem, the **appetite** (how much time it's worth: a "small batch" of 1-2 weeks, or the full six-week "big batch" — appetite is decided *before* solving, not estimated after), a rough solution sketch (breadboarding — naming the affected places/components and the connections between them, deliberately not a detailed spec), and explicit **rabbit holes** to call out and avoid.
- **The betting table**: stakeholders review shaped pitches each cycle and *bet* a team's time on them for the next cycle. Nothing gets built without being bet on — there's no ever-growing backlog of approved-but-unscheduled work.
- **Building**: once a team takes a bet, they work with real autonomy — no mandated daily standups, no ticket backlog to work through, just the pitch. If scope is running long, the answer is **cut scope**, not extend the deadline — "it's not that the feature isn't done, it's that the cycle is done, so what's the smallest real version we ship."
- **Hill charts**: Shape Up's signature progress visualization — not "% complete," but where work sits between "figuring out the unknowns" (uphill) and "known execution left" (downhill). A task stuck at the top of the hill for a while is a real risk signal in a way a ticket board doesn't show.

If asked how you'd scope a feature under Shape Up, the senior-level answer names appetite-driven cutting explicitly: *"I'd rather ship a smaller version that fits the appetite than ask for more time — the discipline is deciding what's actually essential before we start, and cutting scope, not slipping the cycle, when we're wrong."* If you've only worked in Scrum, say so plainly and pivot to that same principle — interviewers are checking for the mental model, not a specific résumé line.

### 3.4 The TypeScript requirement — likely worth a direct question

The JD body names Go as "the primary language," but the poster-added requirements list "5+ years of work experience with TypeScript" alongside AWS and Go as hard requirements. That's a real inconsistency worth surfacing rather than silently prepping around — see Section 7. If it does come up: the most plausible fit given the rest of the JD is a Node/TypeScript layer for **AI agent orchestration** (LangChain's JS/TS bindings exist and some shops prefer TS for the orchestration layer while keeping Go for the high-throughput telephony/API core) or a web admin console. Keep your prep proportional — don't over-invest in TypeScript specifics for a role whose core stack is clearly Go, but be ready to talk about async/Promise patterns, typed API contracts (generating TS types from an OpenAPI spec or a gRPC/protobuf definition — ties back to [API Design](../11-api-design/README.md)), and general willingness to be polyglot on a small team.

### 3.5 Agentic frameworks and the CX/Integrations split

[AI & LLM Integration](../14-ai-llm-integration/README.md) covers the concepts (RAG, tool calling, agent orchestration patterns, guardrails, cost/latency trade-offs) generically. This posting names specific frameworks and a specific domain split worth adding on top:

- **LangChain** — chains and agents as composable steps (LCEL, its declarative composition syntax), tool-calling abstractions, memory modules for multi-turn context. Its agent loop has largely moved toward **LangGraph** (explicit state machines/graphs for multi-step agent workflows) because plain "agent picks the next tool in a loop" proved hard to debug and control at production scale — know this shift happened if the interviewer asks "LangChain vs. LangGraph."
- **LlamaIndex** — more narrowly focused on the *data* side: connectors, chunking/indexing strategies, and retrieval — the RAG-pipeline half of an agentic system rather than the orchestration/control-flow half. A common real architecture pairs LlamaIndex for retrieval with LangGraph (or a hand-rolled loop) for orchestration.
- **The CX vs. Integrations split, concretely:**
  - **Customer Experience** shape: AI *on* the interaction — live call transcription, real-time agent-assist suggestions, post-call summarization, sentiment/intent extraction, possibly a conversational AI agent that can handle simple calls end-to-end. Latency-sensitive, media-pipeline-adjacent (see 3.2).
  - **Integrations** shape: AI/automation *around* the interaction — mapping data between Mango Voice and third-party CRM/PMS/EHR systems, likely including using an LLM to normalize unstructured data (a call summary, a free-text note) into a structured record for a downstream system with its own schema. This is less about real-time latency and more about **reliable, idempotent, schema-aware sync** — the outbox pattern, webhook delivery, and CDC material in [Messaging & Event Streaming](../10-messaging-and-event-streaming/README.md) and [API Design — operations](../11-api-design/03-api-operations.md) is the load-bearing prep here, with an LLM step inserted into an otherwise-standard integration pipeline.

### 3.6 HIPAA for a backend engineer

Listed as "preferred," but the EHR-integration bullet makes it functionally load-bearing if this team serves healthcare customers at all. You do not need to be a compliance officer — you need the backend-relevant shape:

- **PHI** (Protected Health Information) is the thing HIPAA protects: any individually identifiable health information. A call recording or transcript that mentions a patient's name and a health condition is PHI the moment both are present together.
- **The Security Rule's technical safeguards**, in backend terms: **access control** (unique user IDs, no shared logins, role-based access), **audit controls** (log who accessed which PHI record, when — not optional, and a common gap in systems bolted together quickly), **integrity controls** (detect unauthorized alteration), and **transmission security** (TLS in transit; encryption at rest for stored PHI).
- **Business Associate Agreements (BAAs)** — any third-party vendor that touches PHI on your behalf needs a signed BAA: your cloud provider (AWS offers BAAs for HIPAA-eligible services), and critically, **your LLM API vendor**. Sending a call transcript containing PHI to a third-party LLM API without a BAA in place is a real, common compliance mistake — this is the single most likely "gotcha" question in this area: *"if you're building AI call summarization for a healthcare customer, what changes about how you call the LLM?"* The answer: verify the vendor offers a BAA and that the specific product tier/endpoint is covered by it (not all a vendor's products are automatically in scope), and consider whether de-identifying the transcript before the LLM call is possible for the feature's actual requirement.
- **Minimum necessary standard** — systems and staff should only access the minimum PHI needed for the task, which shapes API/data-model design (don't return a full patient record when a feature only needs a status flag).
- **De-identification** as an escape hatch — if a feature (e.g., aggregate call-volume analytics) doesn't need patient identity, strip it before it ever reaches that pipeline, which removes that pipeline from HIPAA scope entirely.

### 3.7 SMS/MMS/RCS quick primer

Lower priority than VoIP for this specific role, but worth the basics:

- **SMS** is fundamentally a carrier signaling message, 140 bytes / 160 GSM-7 characters per segment; longer messages are split and reassembled client-side via a User Data Header (UDH) — this is why "SMS character limits" and multi-part message ordering are real bugs, not trivia. B2B/bulk sending typically goes through **SMPP** (a binary protocol to an SMS gateway/carrier) or a higher-level API (Twilio-style) that abstracts SMPP away.
- **MMS** adds multimedia via a different underlying protocol stack (built on WAP/HTTP concepts historically) — treat it as a separate delivery path from SMS in your mental model, not "SMS with an attachment."
- **RCS** (Rich Communication Services) is the modern successor — typing indicators, read receipts, rich cards — but carrier and OS support is still fragmented (Apple's RCS support is comparatively recent), so production systems generally need SMS as a fallback, not RCS-only.
- **Backend concerns that transfer directly from messaging patterns you already know**: delivery receipts (DLRs) arrive asynchronously as webhooks — treat them with the same idempotent-webhook-consumer discipline as any other webhook (see [API Design — operations](../11-api-design/03-api-operations.md)); outbound sending itself benefits from the same at-least-once/idempotent-producer thinking as the [SQS/SNS material](../10-messaging-and-event-streaming/04-aws-messaging-sqs-sns-eventbridge.md); and compliance (10DLC registration, opt-out/TCPA handling in the US) is a real operational concern, not just a legal footnote, because carriers actively filter/block non-compliant traffic.

## 4. Scenario question — design the whole thing end to end

This is the shape of question most likely to appear, combining the telephony and AI halves of the JD. Practice it out loud before the interview.

> **"Design a feature that listens to a live customer support call, produces a real-time summary, and pushes a structured record into the customer's CRM — which, for some customers, is an EHR — the moment the call ends."**

A strong answer moves through, in order:

1. **Media access layer** — get audio out of the live call via the media server's fork/tee capability, so a slow or failed AI pipeline never touches the primary call path or degrades audio quality (see 3.2). This is the layer most candidates skip straight past into "call an LLM."
2. **Streaming AI layer** — a streaming STT service feeding incremental transcript chunks; decide explicitly whether summarization runs incrementally during the call (higher cost, enables live agent-assist) or once at call end (cheaper, simpler, no live-assist feature). State the trade-off rather than picking silently.
3. **Reliability layer** — call-end triggers a durable event (the outbox pattern from [system design](../06-system-design/02-caching-and-microservices.md)), decoupling "the call ended" from "the CRM got updated" so a downstream failure is retryable and doesn't hold up call teardown or get silently lost.
4. **Integration layer** — a per-CRM/PMS/EHR adapter mapping the summary into that system's schema, delivered via an idempotent, signed webhook or API call (duplicate-safe on retry) — see [API Design — operations](../11-api-design/03-api-operations.md).
5. **Compliance layer** — if the target is an EHR: confirm the LLM vendor is covered by a BAA before the transcript ever leaves your infrastructure, encrypt the transcript at rest, and log every access to it (3.6). This layer is what separates a senior answer from one that only mentions compliance if prompted.
6. **Observability/on-call layer** — on a 3-5 person team with no dedicated SRE, name what you'd actually page on: AI pipeline latency/failure rate, dropped or stalled transcription streams, and failed CRM syncs (with a dead-letter/retry view) — and explicitly note that a small team needs *fewer, higher-signal* alerts, not a copy of a big-company alerting setup.

Naming all six layers, unprompted, including the compliance layer without being asked specifically about healthcare, is what separates a senior answer here.

## 5. Behavioral prep — this JD's specific angle

- *"Tell me about a time you owned a feature end-to-end on a small team."* — this JD has no separate SRE/QA/ops function; a story where you designed, built, deployed, and later got paged for the same thing lands better than a story where you handed off after building.
- *"Tell me about mentoring a more junior engineer."* — expected explicitly in the JD; have a specific story (a code review pattern you taught, a pairing session, not "I answer questions when asked").
- *"Tell me about working within a fixed-time, fixed-scope constraint and having to cut something."* — this is the Shape Up question in behavioral clothing; a story where you cut scope to hit a deadline (rather than the deadline slipping) is exactly the mental model in Section 3.3.
- *"Why does AI/agentic work in the Customer Experience or Integrations space interest you specifically?"* — a generic "I'm excited about AI" answer under-performs; have a concrete opinion about what's hard about real-time AI on a live call, or about reliable data sync between systems with different schemas, from Section 3.2/3.5.
- *"Tell me about an on-call incident you handled."* — expect a structured answer: detection, diagnosis, mitigation, and — since this is a small team — what you changed afterward given you likely own the fix yourself with no separate team to escalate to.

## 6. Technical question bank

Graded roughly junior → senior; the senior-tier and VoIP/AI-specific ones are where this loop will differ most from a generic backend interview.

**Core / must-have**

1. Walk through a basic SIP call setup (`INVITE` through `ACK`) and explain what SDP negotiates.
2. What's the difference between SIP and RTP, and why does WebRTC need ICE/STUN/TURN when plain RTP doesn't have that layer?
3. Design a REST or gRPC API for configuring a call-routing rule (e.g., "route calls from this number to this queue during business hours"). What does the schema look like, and what are the edge cases?
4. Explain the difference between at-least-once and exactly-once delivery, and why a webhook receiver (e.g., a delivery-receipt callback) must be idempotent regardless of which the sender claims.
5. You're storing millions of call detail records (CDRs) per day. Would you reach for the relational store or a NoSQL one, and why?

**Applied / senior**

6. Design the feature from Section 4 (live call → summary → CRM sync) without looking at the model answer first.
7. A customer reports that AI call summaries are randomly missing for calls that clearly connected and had audio. Walk through your diagnosis, given the media-fork architecture in 3.2.
8. You need to add real-time agent-assist suggestions during a live call. What's your latency budget end-to-end (STT → LLM → delivery to the agent's screen), and where would you look first if it's blowing that budget?
9. Design the ingestion pipeline for pushing call summaries into a customer's EHR. What changes about your design compared to pushing into a generic CRM?
10. How would you roll out a new AI feature (say, live sentiment detection) to a subset of customers first, and what would make you pull it back?
11. Explain the outbox pattern and why you'd use it between "call ended" and "CRM updated," rather than calling the CRM directly from the call-teardown code path.
12. A third-party CRM's webhook delivery is unreliable — you're missing update confirmations. How do you build a sync that's correct despite that?
13. What's your approach to versioning an internal API that multiple integration adapters (one per CRM/PMS/EHR) depend on, when you need to change its schema?

**Nice-to-have / breadth**

14. Compare LangChain/LangGraph and LlamaIndex — what does each actually solve, and where would you use both in the same system?
15. What do BAAs have to do with calling a third-party LLM API on healthcare-adjacent data, and what would make you say no to a proposed integration?
16. Explain appetite-based scoping (Shape Up) vs. story-point estimation (Scrum) — what does each optimize for?
17. How would you design SMS delivery-receipt handling so a burst of DLR webhooks doesn't create duplicate state updates?
18. What's your first-90-days plan for ramping up on a telephony domain you've never worked in before, while still shipping?

## 7. Logistics — questions worth asking them directly

- The JD says Go is "the primary language" but separately requires 5+ years of TypeScript — what is the TypeScript actually used for on this team (a Node service, an admin frontend, the AI orchestration layer)?
- Is the "AI-driven features and agentic systems" work greenfield, or is there existing infrastructure (a transcription pipeline, an agent framework already chosen) you'd be building on top of?
- Which domain is the more immediate focus right now — Customer Experience or Integrations — or is that still being decided?
- For the healthcare-adjacent customers specifically: is there an existing HIPAA compliance program/BAA structure in place, or would building that be part of this role?
- What does on-call actually look like on a 3-5 person team — rotation frequency, escalation beyond the team, and how much of the telephony infrastructure itself (vs. just the application layer) you'd be responsible for?
- What's the current media-server/telephony-core stack (self-hosted Asterisk/FreeSWITCH/Kamailio, a managed SIP trunking provider, or something proprietary)?

## 8. Suggested prep order

1. [Go](../01-languages/go/README.md) — full read if it's been a while; this carries the most interview weight by volume.
2. This file's Section 3.2 (VoIP) and 3.7 (SMS/MMS/RCS), out loud, until you can walk the SIP call flow and explain WebRTC's ICE/STUN/TURN role without notes.
3. [AI & LLM Integration](../14-ai-llm-integration/README.md), then this file's Section 3.5 on top of it.
4. [API Design — operations](../11-api-design/03-api-operations.md) and [Messaging & Event Streaming](../10-messaging-and-event-streaming/README.md) — the reliability/integration backbone for Section 4's scenario.
5. This file's Section 3.6 (HIPAA) and Section 3.3 (Shape Up) — shorter reads, but the two most likely to catch you unprepared precisely because they're easy to skip.
6. [AWS](../04-aws/README.md) and [System Design](../06-system-design/README.md) — standard depth, confirm you're not rusty.
7. [Behavioral](../08-behavioral/README.md) — draft the five stories in Section 5 before the interview, not during it.
8. Section 4's scenario question, out loud, until you can name all six layers unprompted.
