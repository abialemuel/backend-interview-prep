# Study Guide

How to use this repository effectively, plus a recommended study plan and self-assessment framework.

## Principles

1. **Active recall > passive reading.** After reading a concept, close the file and explain it out loud or write it from memory. Then check.
2. **Spaced repetition.** Revisit weak topics after 1 day, 3 days, 1 week.
3. **Depth over breadth, then breadth.** Understand a few things deeply first, then expand coverage.
4. **Articulate trade-offs.** Backend interviews reward "it depends" reasoning — know *when* and *why*, not just *what*.
5. **Practice coding by hand.** For DSA and language-specific questions, write code without an IDE/autocomplete.

## Recommended Study Plan (6–8 weeks)

Adjust based on your existing strength and interview timeline.

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

### Week 3 — Frameworks
- Laravel: all files in `02-frameworks/laravel/`
- Next.js: all files in `02-frameworks/nextjs/`

### Week 4 — Databases
- MySQL: all files in `03-databases/mysql/`
- Redis: all files in `03-databases/redis/`

### Week 5 — AWS
- All files in `04-aws/`

### Week 6 — DevOps
- Terraform, Ansible, GitHub Actions, Datadog in `05-devops/`

### Week 7 — System Design
- All files in `06-system-design/`

### Week 8 — DSA + Behavioral
- `07-data-structures-algorithms/`
- `08-behavioral/`
- Mock interviews + final review of all `interview-questions.md` files

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

## A Note on "Up-to-Date"

Tools evolve. Before your interview, quickly check the latest stable version and any recent major changes for each tool in your stack — interviewers sometimes ask "what's new in version X?" as a signal of engagement.
