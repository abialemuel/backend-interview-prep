# Project Stories to Rehearse

These outlines use achievements documented in **Abia New CV.pdf**. Fill in implementation details from memory before the call; the prompts below are not additional claims about what you shipped.

## Telkom AI Proxy and MCP Orchestrator

**Use for:** production GenAI, model integration, tenant isolation, operating shared infrastructure.

> At Telkom I built the AI Proxy and MCP Orchestrator from scratch for TelkomGPT. It supported language, vision, and embedding models across multiple business units. I implemented RBAC for tenant isolation and token usage tracking for AI cost management. The platform enabled workflows including an AI-assisted GitLab merge-request reviewer and a VS Code assistant extension.

**Connection to VSL:** model integrations, enterprise access controls, cost visibility, and a shared platform serving different workflows.

Prepare these details from your own experience:

- A request's path from the caller to the model and back.
- What the proxy owned versus what the MCP orchestrator owned.
- Where you enforced tenant identity and resource access. RBAC alone does not explain data isolation.
- One MCP tool, its input schema, how authorization worked, and how you handled failure.
- How you collected token usage and what decisions that visibility enabled.
- Which components you personally built and which other engineers owned.
- One limitation or trade-off in the design.

The CV does not specify routing algorithms, streaming behavior, fallback strategies, exact latency, or traffic volume. Explain those only if you implemented them.

## Careem CERT event relay

**Use for:** reliability, asynchronous jobs, error classification, ordered processing.

> At Careem I worked on the CERT government API event relay. The retry logic could silently drop regulated trip events. I implemented three failure categories—transient, terminal, and ordering-not-ready—and an SQS FIFO retry consumer with an ordering gate and exponential backoff. This fixed the silent event drops.

**Connection to VSL:** expensive generation workflows need explicit state, safe recovery, and clear handling when a provider fails or returns an uncertain result.

Prepare:

- A concrete example of each failure category.
- The original loss path and why the previous logic missed it.
- How the ordering gate knew whether an event was ready.
- How message-group boundaries were chosen and how processing order was enforced.
- How duplicate deliveries and uncertain provider outcomes were handled, if implemented.
- Retry limits and the handling of terminal failures, if implemented.
- The evidence used to verify recovery and eliminate silent drops.

Do not promise end-to-end exactly-once processing because SQS FIFO is present. Explain the actual consumer and external API behavior.

## RRQ first engineer and product launch

**Use for:** builder mindset, small-team ownership, working with leadership, product delivery.

> I was the first engineer hired at RRQ and shaped the backend strategy with the Head of Products and Tech Advisor. I built and launched the community platform backend for missions, competitions, raffles, digital-product transactions, and billing integration. I integrated Auth0 for authentication and authorization and managed the backend services on AWS.

**Connection to VSL:** this is your strongest documented example of building from zero with a small leadership team.

Prepare:

- One feature's journey from a product idea to launch.
- Your role in deciding scope and architecture.
- A trade-off you made to deliver the first usable version.
- How you worked with frontend or product colleagues.
- One issue after launch and what you changed.
- What you documented to make continued development easier.

Your CV says you built the backend. Do not describe the entire web frontend as your own work unless it was.

## Telkom monitoring architecture and memory reduction

**Use for:** measurable results, concurrency, performance, infrastructure costs.

> At Telkom, I changed a network-device monitoring workload from a stateful design to a stateless architecture. I added Kubernetes autoscaling and Go worker-pool concurrency. The change reduced memory usage by about 80% and produced infrastructure cost savings.

**Connection to VSL:** media jobs can be long-running and resource-intensive, so bounded concurrency and measurement matter when deciding how to run workers.

Prepare:

- What retained state caused the original memory pressure.
- The before/after measurement and whether load was comparable.
- Which state became external or could be recomputed.
- Why a worker pool was needed and how you selected concurrency.
- How autoscaling affected workload distribution and dependencies.
- Any trade-off in latency, cost, or operational complexity.

Do not convert the memory reduction into a numerical cost reduction: the CV only gives the memory figure.

## Reserve story: Careem restaurant-network integration

**Use for:** feature delivery, external partners, integration complexity.

> At Careem I delivered an integration with AlArabi, a major restaurant network in Saudi Arabia. It covered automated onboarding, catalog synchronization, order lifecycle management, self-delivery tracking, and promotion synchronization. It significantly reduced manual onboarding work.

Prepare one concrete data flow, one partner API limitation, and how you coordinated rollout. Use an exact onboarding improvement only if you can substantiate it; the CV does not provide a number.

## How to keep answers focused

Give a 60–90 second overview first. Spend most of the answer on your actions and the result. When asked to go deeper, draw the request or event flow and explain one difficult choice. Close by connecting the experience to VSL's current needs, then give them room to ask follow-ups.
