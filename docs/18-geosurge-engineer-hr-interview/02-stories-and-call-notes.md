# geoSurge Project Stories and Call Notes

## Three stories to rehearse

### 1. Telkom AI platform — lead with this

> At Telkom I built the AI Proxy and MCP Orchestrator from scratch for TelkomGPT. It supported language, vision, and embedding models across multiple business units. I implemented tenant access controls through RBAC and token usage tracking for cost management. It enabled workflows including an AI-assisted GitLab merge-request reviewer and a VS Code assistant. That experience connects to your need for backend systems that make AI capabilities usable in production.

Before the call, prepare: who used it, one request flow, your precise contribution, one difficult choice, and how you knew it worked. Explain MCP in plain language if asked: a standard way for an AI application to interact with tools and contextual resources. Then explain what your orchestrator actually did.

### 2. RRQ first engineer — show ownership

> I was the first engineer hired at RRQ. I worked with the Head of Products and Tech Advisor to shape the backend strategy, then built and launched services for community activities, digital transactions, and billing. I integrated Auth0 and managed the backend on AWS. That gave me experience taking an early product idea through technical decisions and delivery.

Prepare one feature, a scope decision, a trade-off, and how you collaborated. Describe your backend ownership precisely; do not imply you built the entire frontend or founded the company.

### 3. Careem CERT relay — show production judgment

> At Careem I worked on a government API event relay where retry behavior could silently drop trip events. I separated failures into transient, terminal, and ordering-not-ready cases, and implemented an SQS FIFO retry consumer with an ordering gate and exponential backoff. This fixed the silent drops. The relevant lesson is that a production integration needs to handle failure explicitly, especially when event order matters.

Prepare the original failure path and how the fix was validated. Do not describe SQS FIFO as a guarantee of end-to-end exactly-once behavior.

Reserve example: Telkom monitoring redesign reduced memory usage by approximately 80%. Use it for efficiency questions; explain the measurement and keep it separate from the AI platform. More background is in the existing [CV-based project stories](../16-vsl-global-fullstack-genai-engineer/02-project-stories.md).

## Questions to ask Harry

Choose four or five; do not rush through the entire list.

| Priority | Question | What you learn |
|---|---|---|
| 1 | What would you want this person to deliver in their first three months? | Concrete expectations and current priorities |
| 2 | How is “multi-faceted” divided between backend, data pipelines, infrastructure, frontend, and research? | Whether the scope fits your strengths |
| 3 | Who would I report to, and how do engineers and researchers work together? | Support, collaboration, and decision-making |
| 4 | What locations can you hire in, what overlap hours are required, and is this an employee or contractor role? | Practical eligibility and working arrangement |
| 5 | What is the budgeted compensation range, including currency and equity? | Package alignment |
| 6 | What are the interview stages, and what should I prepare for the next one? | The next action and evaluation format |

If time allows, ask: “Is this a new position, and what has made hiring it a priority now?” Request the full JD and expected level. Ask about on-call or after-hours support if operating services is a substantial responsibility.

For business context, choose one: “Which customer workflow is seeing the most demand?” or “What milestones is the team working toward after the seed round, and how does this hire support them?” You can ask about runway without treating the funding amount as a proxy for it.

Save deeper questions for the engineering conversation if Harry cannot answer:

- Which languages and data systems are in production, and where is the biggest bottleneck?
- Can you walk me through a research result that became a customer feature?
- How do you measure a real improvement in brand representation, given that model answers vary and providers change their models?
- How do you distinguish changes caused by retrieval from changes in learned model knowledge?

These questions test the product and engineering assumptions without claiming knowledge of the implementation.

## Closing the call

> Thanks, Harry. What you described about **[specific need]** connects well to my experience with **[Telkom AI / RRQ / Careem]**. I'm interested in exploring that further. Is there anything about my background you'd like me to clarify, and what would the next step involve?

## Fill in before the call

| Item | Your answer |
|---|---|
| Current employment status and accurate role dates | ___ |
| Actual reason for exploring a move | ___ |
| Residence and permitted work locations | ___ |
| Notice period and earliest start | ___ |
| Sustainable overlap hours and travel/relocation preferences | ___ |
| Target base range, currency, and pay period | ___ |
| Employee/contractor preferences and essential benefits | ___ |
| Actual competing interview deadlines, if any | ___ |
| One genuine disagreement, feedback, and mistake example | ___ |

## Thirty-minute rehearsal

1. **5 minutes:** Say your introduction and “why geoSurge” aloud. Cut anything you cannot explain naturally.
2. **10 minutes:** Rehearse Telkom AI and RRQ; each should fit within 90 seconds. Keep CERT ready as the third story.
3. **5 minutes:** Practise the reason for moving and an honest answer about the limits of your research/frontend experience.
4. **5 minutes:** Fill in the logistics table and choose your four most important questions.
5. **5 minutes:** Check the meeting details, audio, and camera. Open the cheat sheet below.

## Keep this visible during the call

**Pitch:** Senior backend engineer; production AI platform; first-engineer ownership; reliable integrations.

**Evidence:** Telkom AI → model integration, access, usage tracking. RRQ → build and operate from zero. CERT → failure handling and recovery.

**Why this role:** Interest in turning research into useful production features and shaping the backend.

**Learn:** First three months; scope and stack; manager and research collaboration; hiring location and hours; package; next step.

**Remember:** Explain your contribution and result. Keep answers short enough for follow-ups. Distinguish experience from proposed approaches. Use your actual reason for moving and actual availability.
