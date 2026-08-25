# Study Guide

How to use this repository effectively, plus a recommended study plan and self-assessment framework.

## Principles

1. **Active recall > passive reading.** After reading a concept, close the file and explain it out loud or write it from memory. Then check.
2. **Spaced repetition.** Revisit weak topics after 1 day, 3 days, 1 week.
3. **Depth over breadth, then breadth.** Understand a few things deeply first, then expand coverage.
4. **Articulate trade-offs.** Backend interviews reward "it depends" reasoning — know *when* and *why*, not just *what*.
5. **Practice coding by hand.** For DSA and language-specific questions, write code without an IDE/autocomplete.

## Recommended Study Plan (10–12 weeks)

Adjust based on your existing strength and interview timeline. If you have less time, compress weeks 1–4 (your working stack — mostly refresh) and protect weeks 7–11 (where most interview signal lives). Start DSA drills early in parallel (2–3 problems per week from week 1), so week 11 is consolidation rather than a cold start.

### Week 1 — Languages: PHP
- `01-languages/php/01-core-concepts.md`
- `01-languages/php/02-oop-and-modern-php.md`
- `01-languages/php/03-performance-and-security.md`
- `01-languages/php/04-interview-questions.md`

### Week 2 — Languages: Go
- `01-languages/go/01-core-concepts.md`
- `01-languages/go/02-concurrency.md`
- `01-languages/go/03-interfaces-and-errors.md`
- `01-languages/go/04-testing-and-performance.md`
- `01-languages/go/05-interview-questions.md`

**Optional — Rust**, only if a target role names it (a growing bonus/preferred skill at systems-heavy or performance-sensitive shops): `01-languages/rust/`. Read `01-core-concepts.md` (ownership/borrowing/lifetimes — the one topic no other language in this repo covers) and `05-interview-questions.md` at minimum; the concurrency and traits/error-handling files go deeper if the role is explicitly Rust-heavy. Most backend roles will not test this — don't let it crowd out Weeks 3+ unless the JD calls for it.

### Week 3 — Frameworks
- Laravel: all files in `02-frameworks/laravel/`
- Next.js: all files in `02-frameworks/nextjs/`

### Week 4 — Databases
- MySQL: all files in `03-databases/mysql/`
- Redis: all files in `03-databases/redis/`

### Week 5 — AWS + DevOps
- All files in `04-aws/`
- Terraform, Ansible, GitHub Actions, Datadog in `05-devops/`

### Week 6 — Containers & Kubernetes
- All files in `09-containers-and-kubernetes/`
- Priorities: requests/limits + QoS, the three probe types, "pod keeps restarting" debugging, zero-downtime deploys

### Week 7 — System Design + API Design
- All files in `06-system-design/`
- All files in `11-api-design/`
- These pair well: system design is the boxes, API design is the contracts between them

### Week 8 — Messaging & Event Streaming
- All files in `10-messaging-and-event-streaming/`
- Priorities: delivery guarantees, idempotent consumers, the outbox pattern, Kafka consumer groups

### Week 9 — Distributed Systems
- All files in `13-distributed-systems/`
- Builds directly on weeks 7–8; essential for senior/staff loops, skimmable for mid-level

### Week 10 — Security & Auth + AI/LLM Integration
- All files in `12-security-and-auth/`
- All files in `14-ai-llm-integration/`
- Both are "every loop asks at least one question" sections; neither needs a full week alone

### Week 11 — DSA + Behavioral
- `07-data-structures-algorithms/`
- `08-behavioral/` (build and rehearse your story bank out loud)

### Week 12 — Mocks + Final Review
- Mock interviews (system design and behavioral especially)
- Final pass over every `interview-questions.md` / `04-interview-questions.md` file
- Re-drill anything still rated 🔴/🟡 in your self-assessment

## Self-Assessment Framework

For each topic, rate yourself honestly:

| Level | Meaning |
|-------|---------|
| 🔴 Unfamiliar | I cannot explain the concept. |
| 🟡 Fragile | I can explain the basics but stumble on follow-ups. |
| 🟢 Solid | I can explain it clearly, give examples, and discuss trade-offs. |

Target 🟢 on all "core concept" files before moving to interview questions. Re-rate after each weekly review and update a personal tracker (e.g., a `PROGRESS.md` you create in the root).

## How to Use the Interview-Question Files

- Each `interview-questions.md` has questions grouped by difficulty (Easy / Medium / Hard).
- **Try first:** Read the question, say or write your answer.
- **Then reveal:** Compare with the model answer. Note gaps in a notebook.
- **Redo missed questions** after a few days.

## Mock Interview Tips

- Time-box answers: ~1–2 min for factual, ~4–5 min for system design.
- For system design: always clarify requirements → estimate scale → high-level design → deep dive → bottlenecks → trade-offs.
- For behavioral: use STAR (Situation, Task, Action, Result). Quantify the result.
- Have 2–3 concrete project stories ready (one scaling story, one debugging/incident story, one collaboration/conflict story).
- Ask your recruiter whether coding rounds are AI-allowed, AI-banned, or hybrid — the prep differs (see `07-data-structures-algorithms/README.md`). Also prepare a concrete answer to "how do you use AI tools in your work?" (see `08-behavioral/`).

## A Note on "Up-to-Date"

Tools evolve. Before your interview, quickly check the latest stable version and any recent major changes for each tool in your stack — interviewers sometimes ask "what's new in version X?" as a signal of engagement.
