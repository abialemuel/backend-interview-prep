# Hiring-Manager Fit Interview

The recruiter describes this as an assessment of **overall fit**. Your JD makes senior ownership, delivery, judgment, reliability, mentoring, and collaboration central. Expect project discussions and potentially technical follow-ups; coding or a standalone design exercise has not been confirmed for this interview.

All personal achievements below come from **Abia New CV.pdf**. Implementation questions identify details to prepare from memory. Proposed approaches are explicitly framed as proposals.

## What to demonstrate

| JD responsibility | What your answer should make clear | Best CV evidence |
|---|---|---|
| Own services or functional areas | Scope, decisions, dependencies, rollout, and ongoing operation | RRQ backend; Careem integration |
| End-to-end delivery | How you moved from requirements through release and support | Careem AlArabi; RRQ launch |
| Evaluate trade-offs and risks | Alternatives considered, chosen constraints, and consequences | Telkom monitoring redesign |
| Production health and reliability | Failure mode, diagnosis, mitigation, and durable improvement | Careem CERT relay |
| Code reviews and standards | Specific feedback that improved code or engineering practice | Telkom Board of Experts and mentoring |
| Cross-functional collaboration | How you aligned an engineering solution with a product need | RRQ product leadership; AlArabi integration |
| Performance and observability | Measured bottleneck, intervention, and evidence | Telkom memory reduction, indexing, trace library |

Treat this as a preparation framework derived from the JD, not Viki's internal scoring rubric.

## Two-minute introduction

> I'm a senior backend engineer with over eight years of experience, primarily in Go and Ruby. My background spans consumer platforms, integration work, and shared infrastructure.
>
> At Careem, I've delivered an end-to-end restaurant-network integration covering onboarding, catalog synchronization, orders, delivery tracking, and promotions. I also built the reliability layer for a government event relay, where I classified failures and implemented ordered retries to address silent event drops.
>
> At Telkom, I built an AI Proxy and MCP Orchestrator serving several business units, with role-based access controls and token usage tracking. I also redesigned a network monitoring workload, reducing memory usage by about 80%, and contributed to backend standards, mentoring, and observability.
>
> Earlier, I was the first engineer at RRQ and built its backend platform from scratch. I also worked on caching, integrations, and cloud migration at Bukalapak, and inventory and payment services at Tanihub.
>
> The Core Services role appeals to me because it combines ownership of backend services with reliability, performance, and cross-functional delivery. I'd like to understand your current priorities and which services this hire would own.

Use this as an outline rather than a memorized monologue. Spend more time on the examples most relevant to the manager's priorities.

## Tell me about a system you owned

Lead with **RRQ** for breadth of ownership:

> I was the first engineer hired at RRQ. I helped shape the backend strategy with the Head of Products and Tech Advisor, and built and launched the community platform backend. It supported missions, competitions, raffles, digital-product transactions, and billing integration. I integrated Auth0 and managed the backend services on AWS.
>
> The ownership was broader than implementing an endpoint: I had to make the backend work for a product being built from zero. I can walk you through one feature and the decisions I made with the product team.

Prepare one concrete feature rather than covering the whole platform superficially. Explain requirements, architecture, database decisions, interfaces with other teams, rollout, and post-release issues from your actual work. The CV establishes backend ownership; it does not establish that you built the entire frontend.

Likely follow-ups:

- What did you own independently, and what required agreement with others?
- Which alternative did you reject, and why?
- What did you simplify to deliver the first version?
- How did you test, deploy, and support the feature?
- What would you change if you built it again?

## Tell me about a production reliability problem

Lead with **Careem CERT**:

> The CERT government API event relay had retry logic that could silently drop regulated trip events. I implemented three failure categories: transient, terminal, and ordering-not-ready. I then built an SQS FIFO retry consumer with an ordering gate and exponential backoff. This addressed the silent drops.
>
> What I would emphasize is that failure handling was part of the system's correctness. We needed to preserve work and distinguish cases that could recover from cases that needed a different response.

Prepare the exact loss path and what you changed. The CV does not specify a single outage timeline, business penalties, an on-call shift, or a numerical loss rate; do not invent them.

Likely follow-ups:

- Give an example of each failure category.
- What made an event ordering-not-ready, and how did the gate know?
- How did you select message-group boundaries?
- What happened when retries were exhausted or an event was terminal?
- How did you handle duplicates or a timeout after the remote API had already accepted the event?
- How did you establish that the fix worked?

If discussing Viki subscription events, frame it as an analogy: webhook retries and uncertain outcomes also need explicit state and safe processing. SQS FIFO does not by itself prove end-to-end exactly-once behavior.

## Tell me about a measurable performance improvement

Lead with **Telkom monitoring**:

> I redesigned a network-device monitoring workload from a stateful architecture to a stateless one, introduced Kubernetes autoscaling, and used Go worker-pool concurrency. Memory usage dropped by roughly 80%, with infrastructure cost savings.
>
> I'd explain the original memory pressure, the measurement, and why the architectural change addressed it. For Viki, I would apply the same approach: establish the user and system impact, locate the bottleneck, and compare results under representative load.

Prepare the before/after measurement, workload comparability, where state moved, concurrency limits, and the trade-off you accepted. The CV does not give a specific cost saving, latency improvement, or throughput figure for this work.

If asked about database performance, use your **actual Telkom query/index optimization example**: query shape, execution plan, selected index, write/storage impact, and measured result. The CV mentions reduced p99 latency but gives no percentage.

## Tell me about cross-functional delivery

Lead with **Careem AlArabi**:

> I delivered an integration with a major restaurant network in Saudi Arabia. It covered automated partner onboarding, catalog synchronization, order lifecycle management, self-delivery tracking, and promotion synchronization. It reduced manual onboarding work significantly.
>
> I'd use one part of that integration to explain how I coordinated the API contract and product expectations, handled dependencies, and supported delivery.

The first paragraph is supported by the CV. Fill the second with an actual example: who needed to agree, where requirements were ambiguous, how you handled a dependency, how rollout worked, and what you personally did. Do not invent a percentage reduction or claim ownership of every partner-facing function.

## Tell me about mentoring or improving team standards

Your CV places these achievements at **Telkom**:

> At Telkom, I mentored junior engineers on Go practices, clean architecture, and observability. I was also appointed to the Board of Experts to help standardize backend practices, and authored an internal trace library to reduce vendor dependence.
>
> I would pick one concrete review or mentoring example and explain the issue, how I helped the engineer reason through it, and what improved afterward.

Choose an actual example before the call. Be ready to discuss how you gave feedback, avoided taking over the work, and determined whether the engineer could handle similar decisions independently. Do not present an invented mentee success story.

## Describe a disagreement, mistake, or unsuccessful decision

The CV does not contain a complete example of these. Prepare a genuine case with this structure:

1. The decision or mistake and the user/team consequence.
2. Your own contribution to the problem or disagreement.
3. How you compared evidence or understood the other person's concern.
4. What you changed and how you checked the outcome.
5. A specific habit or safeguard you adopted afterward.

For disagreement, explain competing priorities without making the other person sound unreasonable. For mistakes, show responsibility and a practical improvement. A retry bug you inherited is a reliability story; it is not automatically a personal mistake story.

## How do you drive a project when requirements are ambiguous?

Suggested approach; connect it to one actual RRQ or Careem example:

> I start by clarifying the user outcome, the constraints, and the decisions that are still open. I write down the smallest useful scope and the important interface assumptions, then review them with the people affected. If a dependency is uncertain, I try to validate it early. I track risks and communicate changes so stakeholders can make informed decisions.

Follow up with a real decision you made. Explaining a process without an example is weaker evidence of senior ownership.

## How do you balance deadlines and quality?

> I would reduce scope before quietly accepting risks to correctness or security. For the smallest useful version, I would still want clear failure behavior, observability, and a safe rollout. I would explain remaining work and its impact so the product team can choose knowingly. The specific trade-off depends on the service: a subscription decision needs different safeguards from a noncritical analytics feature.

This is a proposed judgment framework. Add a real delivery trade-off from RRQ or AlArabi; do not claim a rollout strategy you did not use.

## How would you own service health?

> I would establish the service's user-facing expectations and failure modes, then make sure we can see when they are violated. I'd look at error rate, latency, saturation, dependency behavior, and the relevant business outcome. For event-driven services, queue age and stuck work matter too. Ownership includes a recovery path, understandable alerts, and follow-through after incidents.
>
> My relevant experience includes Careem's event recovery and Telkom's observability work. I would first learn Viki's existing tools and operational practices so improvements fit the team.

Prepare your actual monitoring, incident, and post-release responsibilities. Having tools in your CV does not establish a particular on-call rotation or formal SLO practice.

## How do you review code?

Suggested answer:

> I first check the intended behavior and the risky cases: authorization, state changes, concurrency, data access, and failure handling. Then I look at whether the code and tests make those behaviors understandable. I explain why a change matters, separate blockers from suggestions, and avoid turning personal style preferences into requirements.

Connect this to a real Telkom mentoring or standards example. Expect to explain one review comment that changed the design or prevented a failure.

## You have AI experience. Why a core backend role?

> The AI work was production platform engineering: shared services, access controls, observability, and cost tracking. My broader career is backend engineering across e-commerce, telecom, and Careem. This role uses the same foundations while adding streaming-specific workflows. I can bring relevant operational experience without assuming the team needs an AI project.

## How will you handle limited streaming experience?

> I haven't owned production streaming or encoding systems yet. I have relevant experience with APIs, cloud services, caching, messaging, performance, and reliable workflows. I would first learn Viki's playback and media-processing paths, trace a representative request or job, and contribute to a bounded change with the team. Then I would build domain knowledge through ownership and operational work.

If asked for technical detail, distinguish media processing from delivery: an encoding job produces media variants; a CDN distributes assets; backend services manage access and metadata. Describe proposed designs as proposals rather than Viki's actual architecture. See [system design practice](02-system-design.md).

## Have you used Viki? What would you improve?

Use Viki before the interview if access permits. Check browsing, title availability, episode selection, subtitle selection, and the subscription flow. Record one genuine observation; never claim an issue you did not observe.

If you have only explored it recently:

> I've been exploring Viki as part of preparing for this conversation. What stands out is the Asian-content focus and the subtitle contributor community. I would like to understand how you balance viewer experience with those community workflows.

For an improvement, use **observation → user impact → hypothesis → evidence to check → small experiment**. A question is acceptable if you have not identified a genuine issue:

> I learned that subtitle completeness can vary by episode and language. How does the team help viewers understand that availability when deciding what to watch? I'd be interested in understanding whether there is confusion worth addressing before proposing a change.

Viki already exposes subtitle completion information, and a subscription does not bypass territorial restrictions. Do not propose those as missing features or bugs without checking the product. [Official Viki explanation](https://support.viki.com/hc/en-us/articles/115009896968-Does-Viki-Pass-include-subtitles-and-access-to-regionally-restricted-content)

## What would your first 90 days look like?

> In the first month, I would learn the services assigned to me, their dependencies, and the deployment and incident practices. I'd trace representative flows and deliver a small change to learn the full release process.
>
> In the second month, I would own a bounded initiative agreed with the manager and participate in the operational responsibilities after onboarding. By the third month, I would aim to independently own a service area, deliver against an agreed outcome, and identify a measurable reliability or performance improvement.

This is a proposal to discuss with the manager; adapt it to the team's onboarding, access, and roadmap.

## Questions to ask the manager

Choose four, allowing time for follow-ups:

- Which services would I own initially, and where are the ownership boundaries with other teams?
- What is the most pressing customer or engineering problem this hire should address?
- How do you measure service health and customer impact, and how does on-call onboarding work?
- How are cross-team API changes, technical decisions, and rollout risks coordinated?
- What distinguishes a strong senior engineer on this team after six months?
- How do engineers contribute to mentorship and standards while delivering their own work?
- How much of the role touches encoding and content delivery versus APIs, subscriptions, and user services?
- What are the next steps after the two fit interviews if I progress?

## Closing

> The discussion about **[a need the manager actually described]** connects well to my work on **[one relevant project]**. Is there any part of my experience you would like me to explain further? I'd also be interested in your expectations for this person in the first few months.
