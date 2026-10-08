# Rakuten Viki — Senior Engineer, Core Services

Preparation for **Abia Darma Lemuel**, based on **Abia New CV.pdf**, the supplied job description, and the recruiter messages shared on **2 October 2026**.

**Codility: passed. Hiring-manager fit interview: passed with positive feedback. Talent Acquisition interview: passed.**

Progress so far:

- Initial screening: full name, notice period, visa status/nationality, salary expectation — **passed**
- Codility test — **passed**
- Hiring-manager interview — **passed**, positive feedback
- Talent Acquisition interview — **passed**
- Coding interview — **next**
- Ops interview
- System Design interview
- Hiring-manager interview / final discussion
- Final decision / offer

## Coding interview — next round

~45 minutes, language-agnostic, data structures and algorithms, LeetCode style, **easy/medium** level. Interviewers score how you break down a problem and communicate your thinking, not just the final code.

Practise plan:

1. Re-drill core patterns: arrays/strings, hash maps, two pointers, sliding window, stacks, linked lists, binary search, trees, BFS/DFS, and basic dynamic programming. See [data structures](../07-data-structures-algorithms/01-data-structures-fundamentals.md) and [common patterns](../07-data-structures-algorithms/02-common-patterns.md).
2. Solve easy/medium LeetCode by pattern until each solve is under ~20 minutes.
3. Practise out loud: restate the problem, clarify input ranges and edge cases, state the brute-force idea and complexity, then improve. Narrate trade-offs while coding.
4. Test with edge cases before saying done: empty input, single element, duplicates, negatives, large input.
5. Language choice is free — pick the one you write fastest (Go or Python).

## Ops interview — after coding

~45 minutes, CS fundamentals plus troubleshooting real-world operational scenarios. Expect "service is slow / erroring — what do you do" style walkthroughs. Practise with the [monitoring and observability guide](../05-devops/datadog/01-observability-and-apm.md), the [messaging guides](../10-messaging-and-event-streaming/README.md), and the database guides in this pack. Structure answers: observe, form hypothesis, check evidence, mitigate, then fix root cause.

## Start here

| Preparation | What to practise |
|---|---|
| [Data structures fundamentals](../07-data-structures-algorithms/01-data-structures-fundamentals.md) | **Next round: coding.** Core DS + complexity |
| [Common patterns](../07-data-structures-algorithms/02-common-patterns.md) | **Next round: coding.** LeetCode easy/medium patterns, practise out loud |
| [Historical technical question bank](03-interview-questions.md) | **Next round: coding.** Candidate-reported coding questions |
| [Datadog / observability](../05-devops/datadog/01-observability-and-apm.md) | **Next: ops round.** Metrics, logs, traces, troubleshooting flow |
| [Messaging and event streaming](../10-messaging-and-event-streaming/README.md) | **Next: ops round.** Retries, ordering, queue failure scenarios |
| [MySQL / PostgreSQL interview questions](../03-databases/mysql/04-interview-questions.md) | **Next: ops round.** Indexes, transactions, slow queries |
| [System design practice](02-system-design.md) | Later round. Streaming-scale design depth |
| [Hiring-manager fit interview](05-hiring-manager-fit.md) | **Passed.** Keep stories warm for final discussion |
| [Talent Acquisition call](01-recruiter-exploratory-call.md) | **Passed.** Compensation notes remain useful for offer stage |
| [Final rehearsal](06-final-rehearsal.md) | Study plan, mock questions, answer-quality checklist |

## What Viki and this role need

The supplied JD describes a global streaming platform specializing in Asian shows and movies, with subtitles in over **150 languages**. The role is based in **Singapore**, reports to an **Engineering Manager**, and covers backend components across APIs, business intelligence, users and communications, subscriptions and monetization, video encoding, and content delivery.

These are listed team responsibilities, not proof that this hire owns every component. Ask which services would be assigned to you and what the team needs most urgently.

Viki's official support documentation explains its community-created subtitles and regional content restrictions. These are useful product details to understand before the call. Do not claim to be a regular Viki viewer unless that is true.

## Your positioning

> Senior backend engineer with over eight years of experience in Go and Ruby, production microservices, cloud infrastructure, integration reliability, and shared platform engineering. You bring evidence of service ownership, measurable performance improvements, mentoring, and building products from scratch.

Your AI platform experience is relevant as an example of operating shared services. Keep the conversation connected to Viki's backend responsibilities and user outcomes.

| JD requirement | Your CV evidence | Interview focus |
|---|---|---|
| 6–9 years; strong backend language | 8+ years; Go primary, Ruby, Python, PHP | Explain Go depth and the services you personally built |
| Distributed systems and microservices | Careem integrations; Telkom shared platform; Bukalapak migration work | State boundaries, failure modes, and operational responsibility |
| Relational databases | PostgreSQL/MySQL skills; Telkom query/index improvements | Prepare one actual query plan, index decision, and result |
| Messaging and caching | Careem Kafka/SQS/Redis; Tanihub Kafka/Redis; Bukalapak Redis caching | Discuss retries, ordering, cache consistency, and dependency failures |
| GCP/AWS | Bukalapak GCP migration; RRQ AWS ownership; Careem AWS | Distinguish components you configured from platforms you used |
| End-to-end delivery | Careem AlArabi integration; RRQ backend launch | Explain design, collaboration, rollout, and support from your actual work |
| Reliability and observability | Careem CERT relay; Telkom monitoring and tracing tools | Explain detection, mitigation, recovery, and evidence of improvement |
| Mentoring and standards | Telkom mentoring, Board of Experts, shared trace library | Prepare one specific example of helping another engineer |
| Consumer scale | Bukalapak, Tanihub, Careem | Use actual service metrics if known; company-wide users are not your service throughput |
| Video/media experience, preferred | Not documented in the CV | Explain transferable systems experience and a concrete learning plan |

## The three stories to lead with

1. **Careem CERT relay:** silent event drops, three failure categories, SQS FIFO retry consumer, ordering gate, and exponential backoff. Use for reliability and correctness.
2. **Telkom monitoring redesign:** stateless architecture, Kubernetes autoscaling, Go worker pool, and approximately **80% lower memory usage**. Use for performance and engineering judgment. This metric belongs to the monitoring workload, not the AI platform.
3. **RRQ product launch:** first engineer, backend strategy with product leadership, community features, transactions, billing integration, Auth0, and AWS ownership. Use for autonomy and delivery.

Keep **Careem AlArabi integration** available for cross-functional delivery and **Telkom mentoring/standards** for leadership. Story outlines and follow-up prompts are in the [hiring-manager guide](05-hiring-manager-fit.md).

## Compensation and relocation preparation

Salary expectation, notice period, and visa status were collected at initial screening. At offer stage, distinguish **monthly SGD base**, **annual base**, and **annual total compensation**. Ask for the role's approved range and package structure before choosing an anchor. The earlier guide's fixed S$10,000–S$11,000 target was not your stated expectation or Viki's confirmed budget.

For relocation, ask whether this vacancy supports employer-sponsored Employment Pass applications and what assistance is available. MOM's current eligibility page sets a non-financial-sector minimum of S$5,600 that rises with age, with a higher schedule for new applications from 1 January 2027; COMPASS generally also applies. A salary threshold does not establish your eligibility. The employer can assess the application using MOM's Self-Assessment Tool.

The practical interview answer is a realistic start plan, subject to your notice obligations, pass approval, and agreed relocation arrangements.

## Confirmed process versus historical reports

The recruiter messages, hiring-manager update, and TA emails take priority over candidate reports. Coding (45 min, LeetCode easy/medium, language-agnostic), Ops (45 min, CS fundamentals + troubleshooting), and System Design are confirmed upcoming rounds. The technical packs in this pack are direct preparation for them.

## References

- Your supplied job description and recruiter messages: source for role scope, TA duration, progressive interviews, and flexible order.
- Abia New CV.pdf: source for your professional experience. Private contact details are not included in this preparation pack.
- [Viki subtitle creation](https://support.viki.com/hc/en-us/articles/115015389988-Subtitle-Creation-on-Viki)
- [Viki Pass, subtitles, and regional restrictions](https://support.viki.com/hc/en-us/articles/115009896968-Does-Viki-Pass-include-subtitles-and-access-to-regionally-restricted-content)
- [MOM Employment Pass eligibility](https://www.mom.gov.sg/passes-and-permits/employment-pass/eligibility)
- [MOM Employment Pass application process](https://www.mom.gov.sg/passes-and-permits/employment-pass/apply-for-a-pass)
