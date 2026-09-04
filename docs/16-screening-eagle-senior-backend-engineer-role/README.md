# Screening Eagle / INSPECT — Senior Backend Engineer

This is a targeted preparation pack for the Singapore-based **Senior Backend Engineer** role supporting Screening Eagle and Proceq tools. It is tailored to Abia Darma Lemuel's background and starts with the HR screen before covering the likely technical loop.

> **Evidence note:** Screening Eagle does not publish a definitive engineering interview process. The stages below combine public candidate reports with a clearly labeled estimate. Salary figures are listing data, not a guaranteed offer.

> **Candidate-reported technical round:** A candidate described a two-hour panel covering system design and data structures. The main prompt was to design a resumable API for a 1 GB upload. See [Bulk File Upload System Design](02-bulk-file-upload-system-design.md) for a full practice run.

## 1. Company and product context

Screening Eagle Technologies combines Proceq's non-destructive testing hardware with cloud, mobile, data, and AI software originally developed by Singapore-based Dreamlab. Its mission is to **protect the built world** by helping engineers inspect structures and infrastructure more effectively.

**INSPECT** is its cloud-connected inspection workflow platform. Inspectors can capture observations in the field, associate them with maps, 2D plans, or 3D representations, collaborate on projects, and generate reports. The product also includes AI-assisted visual inspection and integrates with Screening Eagle's broader sensor ecosystem.

For a backend engineer, the interesting system shape likely includes:

- large projects containing structured inspection records, images, scans, and reports;
- field-to-cloud synchronization over unreliable connections;
- concurrent collaboration and multi-user consistency;
- background processing for reports, media, imports, and integrations;
- authorization and tenant isolation for commercially sensitive asset data;
- APIs and webhooks shared by web, mobile, sensor, and partner clients;
- long-lived engineering records where durability and auditability matter;
- observability and operational resilience across global users.

Useful official reading:

- [Screening Eagle careers](https://www.screeningeagle.com/en/about-us/career)
- [INSPECT product overview](https://success.screeningeagle.com/en-us/inspection-software)
- [How INSPECT digitizes inspection workflows](https://www.screeningeagle.com/en/about-us/news/screening-eagle-inspect-moves-industry-into-the-digital-age)
- [Screening Eagle platform and company background](https://www.screeningeagle.com/en/about-us/news/leveraging-augmented-reality-to-save-money%E2%80%93and-lives)

### A concise "Why Screening Eagle?" answer

> Screening Eagle interests me because the backend supports a real physical-world workflow. INSPECT brings together field data, cloud software, inspection hardware, real-time collaboration, and AI-assisted analysis, so reliability and data integrity have a direct connection to infrastructure safety. That is a compelling use of backend engineering. The role also matches the work I have been doing in Go, distributed systems, cloud infrastructure, reliability, and developer enablement, while giving me a new and meaningful problem domain to learn.

Avoid making the answer primarily about AI. The posting is much more explicit about reliable cloud applications, testability, APIs, data structures, and operations.

## 2. What the job description is really testing

| JD signal | Likely evaluation |
| --- | --- |
| "Take ownership" | Can you independently move from ambiguous requirement to production operation? |
| "Strategic choice of data structure and algorithms" | Can you explain complexity and domain trade-offs instead of reaching for LeetCode patterns mechanically? |
| "APIs, batch jobs, webhooks, integrations" | Can you design synchronous and asynchronous boundaries, retries, idempotency, and failure handling? |
| "Heavy emphasis on code testing and designing for testability" | Can you isolate dependencies, define contracts, and build a balanced test strategy? |
| "Continuously document design decisions" | Can you write ADRs/RFCs that capture context, options, trade-offs, and consequences? |
| "Continuous automated testing, releases and deployments" | Can you connect code ownership to CI/CD, rollout safety, observability, and rollback? |
| "Mentorship" | Can you raise team quality through reviews, pairing, standards, and context—not merely answer questions? |
| Go preferred | Expect practical Go questions around concurrency, interfaces, errors, testing, and service design. |
| SQL and Redis | Expect schema/index/query-plan discussion plus caching, TTL, invalidation, and consistency trade-offs. |
| Distributed systems and high availability | Expect failure scenarios, not definitions: duplicates, partial failure, ordering, timeouts, failover, and recovery. |
| Kubernetes, Terraform, Jenkins/GitLab CI | Expect evidence of production usage and troubleshooting rather than certification trivia. |

## 3. CV-to-role mapping

Abia exceeds the baseline requirement: **8+ years** of backend engineering against a stated minimum of four, with direct production experience across almost every named technology.

| Their need | Strongest evidence to use |
| --- | --- |
| Go backend ownership | Careem integration and government event relay; Telkom AI Proxy and MCP Orchestrator |
| Distributed systems and resilience | Careem CERT relay: error classification, SQS FIFO retries, ordering gate, exponential backoff, elimination of silent event loss |
| APIs and integrations | KSA restaurant-network integration covering onboarding, catalog, orders, delivery tracking, and promotions |
| Scalability and performance | Telkom monitoring redesign: stateful to stateless, worker pools, autoscaling, about 80% memory reduction |
| SQL performance | Targeted indexes and execution-plan optimization reducing p99 latency at Telkom |
| Redis | Production caching and distributed-system usage at Bukalapak, Tanihub, Telkom, and Careem |
| Cloud and Kubernetes | AWS, GCP, EKS/GKE, Kubernetes, Terraform, load balancing, containerized microservices |
| CI/CD and operations | GitHub Actions, GitLab CI/CD, Jenkins, Datadog, OTel, Jaeger, Prometheus, and ELK |
| Mentorship | Mentored junior engineers; appointed to Telkom's Board of Experts; standardized backend practices |
| Architecture decisions | First engineer at RRQ Guild; designed its backend strategy and platform from zero |
| Scale | Bukalapak/Mitra ecosystem serving 100M+ users and Telkom platform serving multiple business units |

### Ninety-second introduction

> I am a senior backend engineer with more than eight years of experience building production systems in Go and Ruby across e-commerce, telecommunications, startups, and now Careem. My focus has increasingly been distributed systems, reliability, and platform engineering. At Telkom Indonesia, I built an AI Proxy and MCP Orchestrator used across multiple business units, and I redesigned a monitoring workload into a stateless, autoscaled Go architecture that reduced memory use by about 80%. At Careem, I have delivered a large restaurant-network integration in Saudi Arabia and rebuilt a government event relay so transient failures and ordering delays no longer silently drop regulated trip events. I have also mentored engineers and helped define backend standards. Screening Eagle appealed to me because INSPECT applies these same cloud, integration, data-integrity, and reliability skills to a physical-world product where trustworthy software supports safer infrastructure.

### The short Careem-tenure question

Since Careem began in November 2025, HR will probably ask why another move is being considered. Keep the answer positive and avoid implying that the Careem work is unimportant.

> Careem has given me valuable experience with high-scale integrations and reliability, and I am not looking to leave because of a negative situation. I am selective about opportunities. Screening Eagle stood out because it offers long-term product ownership in Singapore, strong alignment with my Go and distributed-systems background, and a mission where backend reliability supports real inspection and infrastructure outcomes. I am open to relocation, so I felt it was worth exploring whether the role and long-term direction are a particularly strong match.

## 4. HR screen preparation

The first call will likely cover:

- career summary and motivation;
- why Screening Eagle and why Singapore;
- current role and reason for considering a move;
- Go, cloud, Kubernetes, SQL, Redis, and distributed-systems experience at a high level;
- mentoring and senior-level ownership;
- location, relocation, work authorization, and hybrid availability;
- notice period and possible start date;
- current and expected compensation;
- communication ability and English fluency.

### Likely HR questions and answer direction

**Why Singapore?**

> I have already worked with Singapore-based and international teams, and I am intentionally open to relocation for the right long-term role. Singapore offers a strong engineering environment and would let me collaborate closely with Screening Eagle's product and technology teams. I understand this is a hybrid role and am comfortable with that expectation.

**Why should we hire you?**

> I bring the combination this posting emphasizes: hands-on Go development, distributed-system reliability, cloud and Kubernetes operations, SQL and Redis performance work, and senior ownership. I have built systems from zero, repaired failure modes in regulated event pipelines, improved infrastructure efficiency by about 80%, and mentored engineers. I can contribute to implementation immediately while also improving testability, documentation, and operational standards around the services I own.

**What are you looking for next?**

> I am looking for durable product ownership: a team where I can design and build backend services, remain accountable for how they behave in production, and contribute to engineering standards and mentoring. I am especially interested in systems where correctness and reliability have visible customer impact.

## 5. Interview stages

No backend-specific process is publicly confirmed. Candidate reports available for other Singapore engineering roles suggest this probable sequence:

1. HR or recruiter screen.
2. Technical interview with a lead and senior engineer.
3. Live coding or another practical technical assessment, possibly combined with step 2.
4. Hiring-manager, team, or final management discussion.
5. References and offer.

Some candidates reported only an HR call and technical-lead round; another senior Singapore candidate reported a two-hour technical session with a senior engineer and lead. A product candidate reported a much longer multi-stage process. Treat **three substantive stages, with a possible fourth**, as a planning estimate—not a confirmed company policy.

Sources: [Glassdoor candidate reports](https://www.glassdoor.co.uk/Interview/Screening-Eagle-Interview-Questions-E4247061.htm) and [Indeed interview overview](https://www.indeed.com/cmp/Screening-Eagle-Technologies-1/interviews).

Ask HR directly:

> Could you walk me through the remaining stages, including who I will meet and whether the technical evaluation includes live coding, system design, or a take-home exercise?

## 6. Singapore compensation

The closest role-specific public listing gives a base range of **S$5,000–S$10,000 per month**, or **S$60,000–S$120,000 annually**. Indeed estimates Screening Eagle backend-developer pay around **S$7,378 per month** from a small sample, while NodeFlair shows historical senior software-engineering listing ranges extending to approximately **S$11,500 per month**.

Sources:

- [Role-specific listing syndicated from MyCareersFuture](https://www.foundit.sg/job/senior-backend-engineer-screening-eagle-singapore-pte-ltd-singapore-51888562)
- [Indeed Screening Eagle backend salary estimate](https://sg.indeed.com/cmp/Screening-Eagle-Dreamlab-Pte.-Ltd./salaries/Back-End-Developer)
- [NodeFlair Screening Eagle senior software engineer data](https://nodeflair.com/companies/screening-eagle-technologies/salaries/software_engineer-senior/Singapore)

Given Abia's eight-plus years, exact stack alignment, large-scale systems, and leadership experience, the appropriate position is near the top of the advertised band.

Suggested answer:

> Based on the senior scope, the published Singapore range, and my experience across Go, distributed systems, cloud platforms, and technical leadership, I would be targeting around S$10,000 per month in base salary. I am open to discussing it in the context of the exact level, responsibilities, bonus, relocation support, and overall package.

If HR will disclose first:

> I am flexible and would first like to understand the approved range and the complete package. Could you share the base range, variable compensation, and relocation or work-pass support budgeted for the role?

Also clarify bonus, equity, medical insurance, annual leave, relocation, Employment Pass sponsorship, and any on-call allowance.

## 7. Best questions to ask HR

Choose three or four depending on time:

1. **Could you walk me through the remaining interview stages and the format of the technical assessment?**
2. **Is this role dedicated to INSPECT, shared cloud services across INSPECT and Proceq, or another product area?**
3. **Is this a new position or a replacement, and what is the most important outcome expected in the first six months?**
4. **How is the Singapore backend team structured, and who would this role report to?**
5. **What does hybrid work mean in practice for this team?**
6. **What is the approved base range, and how are bonus and other compensation structured?**
7. **Does Screening Eagle provide Employment Pass and relocation support?**
8. **Does the backend team have an on-call rotation, and how is it organized across regions?**
9. **The opening has been reposted—has the role changed, or is the team hiring more than one engineer?**

The first-six-month question is particularly valuable because it turns a generic HR conversation into a discussion about the actual business need.

## 8. Technical preparation priorities

### Go

- goroutines, channels, mutexes, cancellation, bounded concurrency, and leak prevention;
- interfaces at consumer boundaries and dependency injection without a framework;
- error wrapping, classification, retries, and context propagation;
- table-driven tests, fakes, integration tests, race detection, benchmarks, and profiling;
- HTTP middleware, graceful shutdown, timeouts, connection pools, and backpressure.

Use the [Go track](../01-languages/go/README.md), especially concurrency, interfaces/errors, and testing/performance.

### SQL and Redis

- schema design, normalization, constraints, composite and covering indexes;
- `EXPLAIN`/execution plans, cardinality, join strategy, and p99 query diagnosis;
- transactions, isolation, lost updates, optimistic locking, and idempotent writes;
- cache-aside, invalidation, TTL jitter, cache stampede prevention, and hot keys;
- when Redis is a cache versus coordination or durable-state misuse.

Use [PostgreSQL](../03-databases/postgresql/README.md), [MySQL](../03-databases/mysql/README.md), and [Redis](../03-databases/redis/README.md).

### Distributed systems and integration

- webhook delivery with signatures, idempotency keys, retries, and dead-letter handling;
- batch jobs with checkpoints, safe reruns, bounded parallelism, and partial-failure reporting;
- at-least-once delivery, duplicates, ordering, outbox/inbox patterns, and reconciliation;
- timeout budgets, circuit breakers, bulkheads, load shedding, and graceful degradation;
- multi-region or multi-AZ failure analysis, recovery objectives, and data consistency.

The Careem CERT relay is the best anchor story because it demonstrates real classification of transient, terminal, and ordering-not-ready failures. Use [Messaging](../10-messaging-and-event-streaming/README.md), [API operations](../11-api-design/03-api-operations.md), and [Distributed Systems](../13-distributed-systems/README.md).

### Testability

The JD repeats testing strongly enough that it may be a differentiator. A senior answer should cover:

1. Keep domain logic pure where possible.
2. Put database, clock, queue, object-store, and HTTP dependencies behind narrow boundaries.
3. Use unit tests for decision logic and integration tests for real infrastructure contracts.
4. Add contract tests for external integrations and consumer expectations.
5. Use deterministic clocks/IDs and avoid sleeps in concurrency tests.
6. Test failure behavior: timeouts, duplicate delivery, partial writes, cancellation, and retry exhaustion.
7. Run race detection, static analysis, and tests in CI; track flaky tests as defects.

### Architecture documentation

Be ready to explain an ADR format:

- context and problem;
- decision drivers and constraints;
- options considered;
- chosen decision and rationale;
- consequences and operational risks;
- rollout, observability, and rollback plan.

Use the Telkom stateless redesign or Careem relay as an example of a decision worth documenting.

## 9. Likely technical questions

### Candidate tip: practise the parentheses question in Go

One candidate told me they got a parentheses problem. I do not have the exact prompt, so I would practise the common version without assuming it is guaranteed:

> Return `true` when a string of `()`, `[]`, and `{}` is properly balanced and nested.

Examples: `()[]{}` and `([{}])` are valid. `(]`, `([)]`, `((`, and `]` are not.

This is a stack problem. The next closing bracket has to match the most recent opening bracket.

```go
package main

import "fmt"

func isValid(s string) bool {
	stack := []rune{}
	pairs := map[rune]rune{
		')': '(',
		']': '[',
		'}': '{',
	}

	for _, char := range s {
		if opening, isClosing := pairs[char]; isClosing {
			if len(stack) == 0 || stack[len(stack)-1] != opening {
				return false
			}
			stack = stack[:len(stack)-1]
		} else {
			stack = append(stack, char)
		}
	}

	return len(stack) == 0
}

func main() {
	fmt.Println(isValid("()[]{}")) // true
	fmt.Println(isValid("([)]"))   // false
	fmt.Println(isValid("{[]}"))   // true
}
```

What I would say while coding:

> A counter works if there is only one bracket type, but it cannot catch `([)]`. I need the opening order, so I will keep a stack. A closing bracket must match the top item; otherwise I can return false immediately. At the end the stack must be empty.

Complexity is **O(n) time** and **O(n) space** in the worst case. Before starting, ask whether the input can be empty and whether it contains only brackets. This implementation assumes the input contains only bracket characters; otherwise every non-closing character would be pushed onto the stack.

Test these cases before saying you are done:

```text
""       -> true
"()"     -> true
"([{}])" -> true
"(]"     -> false
"([)]"   -> false
"(("     -> false
"]"      -> false
```

If the interviewer allows only `(` and `)`, simplify it to a depth counter: increment for `(`, decrement for `)`, fail as soon as depth becomes negative, and return `depth == 0`. Likely follow-ups are returning the first bad index or handling input as a stream.

1. How would you design an idempotent webhook receiver when the provider retries and delivers events out of order?
2. A batch report job processes a large inspection project and fails at 80%. How do you resume safely without duplicating output?
3. How would you synchronize edits from an intermittently connected field device with changes made in the web application?
4. Design an API for observations attached to locations in a large inspection project.
5. What data structure would you use to represent hierarchical assets or nested inspection locations, and why?
6. How would you diagnose a PostgreSQL query whose p99 rose from 100 ms to 3 seconds while median latency stayed stable?
7. When would you use Redis here, and what happens when Redis is unavailable?
8. How do you prevent a cache stampede when a popular project's cached representation expires?
9. Explain how you would make a Go worker pool bounded, cancellable, and free of goroutine leaks.
10. How would you test a service that writes to PostgreSQL, publishes an event, and calls a partner API?
11. What guarantees can Kafka provide, and why does "exactly once" not automatically make the whole business workflow exactly once?
12. Design a highly available upload and processing pipeline for inspection images and large sensor files.
13. How would you deploy a database migration without downtime while old and new application versions overlap?
14. A Kubernetes service is intermittently timing out but CPU and memory look normal. Walk through your investigation.
15. Tell us about an architecture decision you documented and how the decision changed after review.
16. Tell us about a junior engineer you mentored and how you measured whether the mentorship worked.

## 10. System-design practice scenario

> Design the backend for a cloud inspection platform where field engineers capture observations, images, and annotations against locations in an asset. Multiple inspectors may work on the same project, sometimes with unreliable connectivity. Office users need near-real-time updates and must generate auditable reports.

A strong answer should cover:

1. Clarify project size, media volume, offline duration, consistency requirements, report latency, tenant model, and retention.
2. Separate structured metadata from object storage for large media.
3. Use client-generated operation IDs/idempotency keys for safe offline retries.
4. Define conflict semantics explicitly rather than claiming eventual consistency solves conflicts.
5. Persist canonical changes transactionally and publish downstream work through an outbox.
6. Process thumbnails, AI analysis, and reports asynchronously with retry and dead-letter handling.
7. Push near-real-time project changes through a connection layer while preserving a durable resync API.
8. Maintain immutable audit history for important edits and report versions.
9. Apply tenant-scoped authorization at every data-access boundary.
10. Discuss multi-AZ deployment, backups, restore testing, observability, SLOs, and graceful degradation.

Connect this design to real experience: offline or delayed events map naturally to the Careem ordering gate; asynchronous integration maps to the KSA restaurant network; query/index concerns map to Telkom; scale and caching map to Bukalapak.

## 11. Behavioral stories to prepare

Prepare each as a two-minute STAR answer with measurable results:

- **Reliability under partial failure:** Careem government-event relay.
- **Performance and cost:** Telkom's 80% memory reduction.
- **Building from zero:** Telkom AI Proxy/MCP Orchestrator or RRQ Guild.
- **Complex external integration:** Careem's KSA restaurant network.
- **Database performance:** Telkom indexing and execution-plan optimization.
- **Influence without relying on hierarchy:** Telkom Board of Experts and trace library.
- **Mentorship:** junior Go engineering, clean architecture, and observability coaching.
- **Operating at scale:** Bukalapak migration, Redis caching, and 100M+ user environment.
- **A mistake or disagreement:** choose a real event, own your contribution, and show the changed mechanism—not merely a promise to communicate better.

## 12. Final preparation order

Before the HR call:

1. Rehearse the introduction and "Why Screening Eagle?" aloud.
2. Prepare the Careem-tenure answer without sounding defensive.
3. Decide the salary wording and relocation/start-date facts.
4. Select four HR questions, led by interview stages and first-six-month expectations.
5. Read the official INSPECT overview and be able to explain the product without jargon.

Before the technical round:

1. Review Go concurrency, testing, and graceful service operation.
2. Rehearse the Careem retry/ordering story at implementation depth.
3. Review SQL query plans and Redis failure modes.
4. Practice the inspection-platform system design aloud in 35–45 minutes.
5. Prepare code-testing and architecture-documentation examples.
6. Practice one medium algorithm problem daily, explaining complexity and test cases as you work.
