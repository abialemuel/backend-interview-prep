# Behavioral

Backend interviews are not only about correctness and performance of code. Almost every loop includes a "behavioral" or "experience" round that probes how you have actually operated — handled an outage, made a trade-off under deadline, disagreed with a senior engineer, mentored a junior, owned a project that spanned teams. Strong engineering with weak communication will fail a final thumbs-up just as often as weak engineering.

For roles building and operating infrastructure (databases, job systems, runtime platforms), behavioral is even more weighted than average. The reason: infrastructure work has a long blast radius and slow feature surface, so signals about judgment, reliability, ownership, and collaboration matter more per-question than in product features.

## Why behavioral matters at infrastructure roles

- **Reliability is a behavior, not just an architecture.** Damage from a bad deploy, a missing alert, or a fat-fingered migration is bounded by the engineering choices AND by the surrounding process — who was on-call, who reviewed the change, what the rollback plan was, what the postmortem said. Behavioral rounds probe the process around your engineering.
- **Cross-team collaboration is the job.** Infra teams serve everyone; the work is rarely "go build my own thing." Most of the value is unblocking other engineers, aligning on standards, and absorbing their pain. Interviewers look for evidence of stakeholder management.
- **Independent judgment.** Infra engineers run production with constrained oversight. Interviewers care: do you make sound calls without escalation? Do you escalate when you should?
- **Investment compounds over years.** Hiring committees weigh "do I want this person owning production for years" — a wrong behavioral hire is much more expensive than a wrong coding hire, because eventually they own traffic that touches everyone.

## The STAR method (one-line summary)

Structure every answer as:

- **S**ituation — one or two sentences of context. The world before.
- **T**ask — what *you* were responsible for, or chose to be responsible for.
- **A**ction — the bulk of the answer. What **you** did, in concrete steps. Emphasize "I", not "we"; the interviewer cannot score "we."
- **R**esult — the outcome, ideally quantified. Numbers, metric movement, follow-on changes to process/code. Most people under-do this part.

Full treatment, story bank, and 20+ question-by-question scaffolding in `01-star-method-and-questions.md`.

## What changes at senior/staff level

The questions look similar but the scoring bar moves. At senior+ the interviewer expects: conflict stories where the counterpart is a tech lead or manager (not a peer you outranked), incident stories where you owned the postmortem and the systemic fix (not just the 3am mitigation), mentoring stories with a measurable outcome for the mentee, and disagree-and-commit stories where you genuinely committed to a decision you still think was wrong. Scope and altitude matter: "I fixed the bug" is a mid-level answer; "I fixed the bug, then changed the process so that class of bug can't ship again, and two other teams adopted the gate" is a senior one. See the dedicated section in `01-star-method-and-questions.md`.

One more 2026 addition: many loops now include a question about how you use AI tools in your work. Have a real, concrete answer ready — the file covers how to frame it.

## Files in this section

| File | Contents |
|------|----------|
| `README.md` | This file — overview, why behavioral matters for infra roles, STAR one-liner, sub-file table. |
| `01-star-method-and-questions.md` | STAR in depth, story-bank construction, Amazon Leadership Principles, 20+ common questions with skeletal bullet-point answers, senior/staff-level question variants, incident/on-call story framing using a blameless-postmortem shape, and how to talk about AI tool usage. |