# Model Answers Based on Your CV

These are spoken rehearsal drafts based on **Abia New CV.pdf**. The CV supports your employment history and listed achievements; motivation, preferred working style, and future plans are drafts to adjust to your actual views.

## Why VSL?

> VSL brings together two areas I have worked on: AI integration and dependable production workflows. At Telkom, I built the AI Proxy and MCP Orchestrator powering TelkomGPT. At Careem, I've worked on integrations and retry logic where losing events had real consequences.
>
> Your product needs models, media assets, review, and delivery to work together. I can see how my experience would help build that workflow. The opportunity to work closely with the CTO also appeals to me because I've previously been the first engineer at RRQ and enjoyed owning a product from an early stage.

Use the final sentence only if it matches how you actually feel about the role.

## Why should we hire you?

> I bring production AI experience, strong backend engineering, and experience building from scratch. At Telkom I built a shared AI platform with access controls and token usage tracking. At Careem I implemented failure classification and ordered retries for a government event relay. At RRQ I was the first engineer and built the backend supporting community features and digital transactions.
>
> I would bring that experience to the reliability and delivery of VSL's core workflow. My deepest expertise is backend and AI systems; I would be transparent about my current frontend and media experience while developing the areas this role needs.

## Tell us about your GenAI experience

> At Telkom, I built an AI Proxy and MCP Orchestrator from scratch for TelkomGPT. The production platform supported language, vision, and embedding models across multiple business units. I implemented role-based access controls for tenant isolation and integrated token usage tracking to help manage AI costs.
>
> The platform also enabled an AI-assisted GitLab merge-request reviewer and a VS Code assistant extension. My main contribution was the platform behind those workflows. I can walk you through the interfaces and operational decisions I personally owned.

The CV says the platform enabled those developer tools. Do not imply you personally built their entire frontend or extension unless that is true.

## What experience do you have with voice, TTS, and video?

Your CV lists LLM, vision, embedding, RAG, MCP, and OCR experience. It does not list TTS, dubbing, lip-sync, or video production. If you have no additional relevant work, use:

> My direct production AI experience is with language, vision, and embedding models through Telkom's AI platform. I haven't shipped a production TTS or video-localisation workflow yet.
>
> I can contribute immediately to model integration, tenant access controls, job orchestration, observability, and failure handling. I would build media-specific depth through a small complete workflow: take a short recording, transcribe it, translate the script, generate speech, align the timing, and review the output. That is the workflow I would use to learn the integration points and failure cases.

If you have a real side project, state what it does, which parts you built, and what failed. A proposed prototype is not a completed project.

## How strong are you in JavaScript, TypeScript, and frontend work?

The CV does not document these skills. Choose wording that matches your actual level and add a real example if available:

> My strongest production language is Go. My CV also covers Ruby, Python, and PHP. Most of my documented ownership has been on backend and AI platform work. For JavaScript and TypeScript, I want to be precise about my current experience rather than overstate it.
>
> Could you explain how much of this role involves frontend delivery compared with backend orchestration and media processing? That would help me explain where I can contribute immediately and where I would need to build depth.

Be ready for practical follow-ups about asynchronous JavaScript, type safety, API integration, frontend state, and progress/error handling. Those are likely useful preparation topics based on the advertised stack, not confirmed interview questions.

## How would you use Go and Python here?

> Go is my strongest language for APIs, concurrent services, and production backend work. Python is listed in my skill set with FastAPI and FastMCP, but I would explain my actual Python projects before claiming the same depth as Go.
>
> For a localisation product, I would first understand your existing architecture. Go could handle API and orchestration work; Python could be useful where model libraries and media integrations already use it. I would choose based on the task and team conventions, and avoid adding another service boundary without a clear reason.

The proposed division is an architectural option; it is not a claim about VSL's implementation or your past deployment choices.

## Tell us about a difficult reliability problem

> At Careem, I built a reliability layer for the CERT government API event relay. The existing retry logic could silently drop regulated trip events. I separated failures into transient, terminal, and ordering-not-ready categories and implemented an SQS FIFO retry consumer with an ordering gate and exponential backoff.
>
> That addressed the silent event drops. The lesson I would apply at VSL is to make failure states explicit and preserve enough context to recover the right work. A media-generation timeout should not automatically cause a duplicate expensive job or silently lose an output.

Prepare the exact event-ordering mechanism and how you confirmed the fix. The CV does not state those details or an incident timeline.

## Can you build independently in a small team?

> At RRQ, I was the first engineer hired and helped shape the backend strategy alongside the Head of Products and Tech Advisor. I built and launched the backend for the community platform, including missions, competitions, raffles, digital-product transactions, and billing integration. I also integrated Auth0 and managed backend services on AWS.
>
> That gives me a concrete example of working with a small leadership team and taking a backend product from zero to launch. I can explain how I made technical decisions and coordinated dependencies on that project.

## Tell us about a measurable improvement

> At Telkom, I reduced memory usage by roughly 80% in a network-device monitoring workload. I changed the architecture from stateful to stateless, added Kubernetes autoscaling, and used Go worker-pool concurrency. That also reduced infrastructure costs.
>
> For VSL, I would use the same habit of measuring the bottleneck before optimizing it. I would look at provider cost, queue wait, stage duration, worker resource usage, and the cost of rework before deciding what to change.

The 80% figure is memory reduction for the monitoring workload. The CV does not give an exact cost reduction, throughput change, or AI-platform memory improvement.

## How would you work asynchronously with the CTO?

> I would keep updates concise: what shipped, what I'm working on, what is blocked, and which decision I need. For an ambiguous feature, I would write down the intended user flow and the main trade-offs, get alignment, then deliver a small usable increment.
>
> I would document provider interfaces, job states, and recovery steps so the system can be supported by other engineers. My experience with observability, shared backend standards, and mentoring at Telkom would help me contribute to that continuity.

This is a proposed working approach. Add examples of written updates or remote collaboration that you have actually used.

## What would you do in your first month?

> First I would understand the current upload-to-delivery flow, the customer pain points, and the boundaries between frontend, orchestration, and providers. I'd run the product locally and trace one representative job through the system.
>
> Then I would agree on a small customer-relevant feature or reliability improvement with the CTO, ship it, and use that work to learn the deployment and review process. Given my background, I could contribute quickly on AI integration and backend reliability while building familiarity with the voice and video stages.

## Why are you considering leaving Careem?

Your CV lists Careem from November 2025 to the present. It does not give a reason for considering a move. Prepare your real reason and keep it concise.

A possible interest statement, if accurate:

> I'm interested in a role where I can combine production AI experience with ownership of an early product. VSL's role seems to offer that opportunity, and I would like to understand the scope, expectations, and team before deciding whether it is the right fit.

Do not invent a contract end date, redundancy, dissatisfaction, or availability. Prepare your actual notice period and any employment constraints before the call.
