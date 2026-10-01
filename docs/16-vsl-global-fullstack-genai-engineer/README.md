# VSL Global — Fullstack + GenAI Engineer Interview Prep

Preparation for Abia Darma Lemuel's first conversation with VSL's CEO and CTO, tailored to **Abia New CV.pdf**.

Start with the introduction below, then rehearse the [model answers](01-model-answers.md) and [project stories](02-project-stories.md). The answers are rehearsal drafts based on the CV; use only details you can explain from your own work.

## Understand the product

VSL presents itself as an enterprise audio and video localisation platform. The workflow on its website runs from uploading source assets through preparing translations and terminology, selecting models, generating voice/dubbing/lip-sync, reviewing outputs, and delivering approved versions. Its enterprise positioning also emphasizes team collaboration, permissions, version history, audit logs, and secure handling of customer content.

The engineering opportunity is broader than calling a voice model: it is building a dependable workflow around media assets, asynchronous processing, external AI providers, quality review, and customer delivery.

Useful links: [VSL website](https://www.vslglobal.ai/), [VSL use cases](https://www.vslglobal.ai/use-cases), [VSL privacy policy](https://www.vslglobal.ai/privacy-policy).

## Your fit for the role

Position yourself as a **senior backend and AI platform engineer with experience building products from scratch**. Your most relevant evidence is Telkom's production AI platform, Careem's integration reliability, and your first-engineer role at RRQ.

| VSL requirement | Evidence in your CV | How to explain the connection |
|---|---|---|
| Build core features with the CTO | First engineer at RRQ; built its backend from zero alongside the Head of Products and Tech Advisor | You have worked with a small leadership team, made technical decisions, and shipped a product |
| Go and Python | Go is your primary language; Python, FastAPI, and FastMCP are listed | Lead with your Go depth; prepare a concrete Python example before claiming equal depth |
| Production GenAI | Built Telkom's AI Proxy and MCP Orchestrator powering TelkomGPT, with LLM, vision, and embedding model support | You have operated AI integrations beyond a prototype |
| Enterprise reliability | Careem CERT relay: error classification, SQS FIFO retries, ordering gate, exponential backoff | You can handle failed, expensive, asynchronous processing with explicit state and recovery |
| Tenant isolation and technical continuity | Telkom RBAC, observability, token usage tracking, backend standards, and mentoring | You understand how to make a shared platform maintainable and operable |
| Fullstack JS/TS | JavaScript/TypeScript and frontend delivery are not documented in this CV | Discuss any real examples you have; describe your current depth precisely |
| TTS, audio, and video | These workflows are not documented in this CV | Explain the relevant AI and systems experience, then acknowledge the media-specific learning needed |

## 60-second introduction

> I'm Abia, a senior software engineer with over eight years of experience building backend and distributed systems. Go is my strongest language, and I've worked across e-commerce, telecom, and Careem.
>
> At Telkom, I built an AI Proxy and MCP Orchestrator from scratch for TelkomGPT, supporting language, vision, and embedding models across multiple business units. At Careem, I've delivered a restaurant-network integration and built a reliability layer for a government event relay.
>
> I've also been the first engineer at RRQ, where I built the backend platform from zero. That combination of AI integration, production reliability, and early product ownership is what I would bring to VSL. I'm interested in applying it to your localisation workflow and developing deeper hands-on experience with voice and video.

If they ask about direct voice and video experience, use the answer in [GenAI and media experience](01-model-answers.md#what-experience-do-you-have-with-voice-tts-and-video). The CV does not establish that experience; any additional side projects should be described separately and accurately.

## Questions to practise

Use the [CV-based model answers](01-model-answers.md) to practise. Lead with a direct answer, give one concrete example, and explain its relevance to VSL.

- Tell us about yourself and why VSL?
- What have you built end to end?
- Tell me about a production issue you owned.
- How do you make a feature reliable when it depends on a slow or unreliable third-party API?
- How comfortable are you with TypeScript, Go, and Python? Where would you use each?
- What have you done with generative AI, audio, or video? What would you need to learn?
- How do you work independently and keep a remote team informed?
- What would you focus on during your first month?

Use [Telkom's AI platform](02-project-stories.md#telkom-ai-proxy-and-mcp-orchestrator) for AI integration, [Careem's CERT relay](02-project-stories.md#careem-cert-event-relay) for reliability, and [RRQ](02-project-stories.md#rrq-first-engineer-and-product-launch) for builder ownership. The CV's **~80% memory reduction belongs to Telkom's network-monitoring architecture change**, not the AI platform.

## Technical discussion: localised video pipeline

If asked to design a workflow that takes an uploaded video and produces a dubbed, lip-synced version in another language, start with the product flow and then cover failure handling:

1. Upload the source to object storage using resumable or multipart upload. Validate the format and create a project record.
2. Submit a job and return its ID immediately. Track stages such as transcription, translation, voice generation, lip-sync, and review, with clear progress and status.
3. Run stages asynchronously through a queue. Make retries bounded and safe; use idempotency keys so retries do not duplicate outputs or costly provider work.
4. Put provider integrations behind interfaces so the system can route between models and handle provider limits, failures, or quality changes.
5. Store intermediate assets and outputs with versioned metadata: source, locale, provider/model, stage status, and output references. Make jobs resumable after worker or provider failures.
6. Allow users to review and edit translation, pronunciation, timing, and generated media before approval.
7. Measure stage latency, failure and retry rates, output quality, and cost. Protect media and voice data with access controls, retention rules, and consent checks.

Trade-offs worth discussing: synchronous versus asynchronous work, retrying versus failing fast, provider portability versus provider-specific features, and automated quality checks versus human review.

Tie this proposed design to your background: Telkom gives you experience integrating models and enforcing access controls; Careem gives you experience classifying failures and recovering ordered work. Describe the video architecture as a proposal, rather than a pipeline you have already built.

## Media concepts to understand before the call

- **ASR/transcription:** turn speech into text, ideally with timestamps. Speaker diarisation identifies who spoke when; it is distinct from transcribing the words.
- **Translation and localisation:** preserve meaning, terminology, and delivery while fitting the available speaking time. A grammatically correct translation may still be too long for the original segment.
- **TTS:** generate speech from text. Assess pronunciation, voice consistency, prosody, and duration. A general LLM evaluation does not establish audio quality.
- **Dubbing:** coordinate translated speech, speaker assignments, timing, and the original soundtrack. TTS is one component of the workflow.
- **Lip-sync:** adjust visual mouth movement to match speech. Face visibility, multiple speakers, scene changes, and visual artefacts affect the result.
- **Media processing:** tools such as FFmpeg can extract, resample, transcode, and combine streams. Re-encoding may be necessary when changing formats or combining incompatible assets.
- **Quality review:** measure timing and technical output validity, and use language reviewers to assess meaning, pronunciation, and naturalness. Keep corrections linked to the asset version being reviewed.

Be ready to explain these concepts; reading this section alone does not establish hands-on experience.

## Questions to ask the CEO and CTO

Choose three or four that fit the conversation:

- What would you like this person to ship in the first 60 or 90 days?
- Given my background in backend and AI platforms, how would you divide this role between frontend delivery, orchestration, and media processing?
- Where is the biggest engineering bottleneck today: workflow orchestration, provider integrations, media processing, or the user experience?
- How do you measure output quality across languages and providers?
- How do you handle long-running jobs, retries, and recovery when a pipeline stage fails?
- What does the current stack look like, and where do Go, Python, and TypeScript fit?
- How do you balance shipping quickly with enterprise needs such as permissions, audit history, and data handling?
- What does working directly with the CTO look like week to week?
- What are the next steps after this conversation?

## Last review before the call

1. **20 minutes:** rehearse the introduction, why VSL, and your precise JS/TS and voice/video experience.
2. **40 minutes:** rehearse Telkom AI, Careem CERT, and RRQ stories. Prepare the architecture and your individual contribution for each.
3. **30 minutes:** explain the proposed video pipeline and the media concepts above out loud.
4. **20 minutes:** prepare three questions for them, your availability, and your actual reason for considering a move from Careem.
5. **10 minutes:** check microphone, camera, connection, and call details. Keep a one-page set of notes within reach.

## Company references

- [VSL product overview](https://www.vslglobal.ai/)
- [VSL use cases](https://www.vslglobal.ai/use-cases)
- [VSL privacy policy](https://www.vslglobal.ai/privacy-policy)
