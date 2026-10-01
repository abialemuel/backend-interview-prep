# VSL Global — Fullstack + GenAI Engineer Interview Prep

Preparation for a first conversation with VSL's CEO and CTO.

## Understand the product

VSL presents itself as an enterprise audio and video localisation platform. The workflow on its website runs from uploading source assets through preparing translations and terminology, selecting models, generating voice/dubbing/lip-sync, reviewing outputs, and delivering approved versions. Its enterprise positioning also emphasizes team collaboration, permissions, version history, audit logs, and secure handling of customer content.

The engineering opportunity is broader than calling a voice model: it is building a dependable workflow around media assets, asynchronous processing, external AI providers, quality review, and customer delivery.

Useful links: [VSL website](https://www.vslglobal.ai/), [VSL use cases](https://www.vslglobal.ai/use-cases), [VSL privacy policy](https://www.vslglobal.ai/privacy-policy).

## 60-second introduction

Personalize the bracketed parts with accurate examples from your experience:

> I'm a software engineer focused on building **[systems or products you have worked on]**. My strongest experience is in **[languages and areas]**. I enjoy owning features end to end: understanding the user need, designing the data flow and API, implementing it, and making it reliable in production.
>
> I'm interested in VSL because localisation brings together a clear customer problem and challenging engineering across software, AI providers, and media workflows. I can contribute my experience in **[relevant experience]**, and I'm keen to deepen my hands-on work with voice and video pipelines.

If you have not shipped voice or video in production, be direct:

> I haven't shipped a production voice or video pipeline yet. My closest relevant experience is **[specific adjacent work]**. I understand the engineering concerns: large media files, long-running jobs, provider variability, retries, and reviewable outputs. I can contribute on the product and backend side while learning the media-specific details.

## Questions to practise

Prepare a specific example for each. Use situation, your actions, the result, and what you learned.

- Tell us about yourself and why VSL?
- What have you built end to end?
- Tell me about a production issue you owned.
- How do you make a feature reliable when it depends on a slow or unreliable third-party API?
- How comfortable are you with TypeScript, Go, and Python? Where would you use each?
- What have you done with generative AI, audio, or video? What would you need to learn?
- How do you work independently and keep a remote team informed?
- What would you focus on during your first month?

For your strongest project, be ready to explain the architecture, your individual contribution, one difficult trade-off, a failure you handled, and a measurable result.

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

## Questions to ask the CEO and CTO

Choose three or four that fit the conversation:

- What would you like this person to ship in the first 60 or 90 days?
- Where is the biggest engineering bottleneck today: workflow orchestration, provider integrations, media processing, or the user experience?
- How do you measure output quality across languages and providers?
- How do you handle long-running jobs, retries, and recovery when a pipeline stage fails?
- What does the current stack look like, and where do Go, Python, and TypeScript fit?
- How do you balance shipping quickly with enterprise needs such as permissions, audit history, and data handling?
- What does working directly with the CTO look like week to week?
- What are the next steps after this conversation?

## Last review before the call

- Rehearse the introduction and two project stories out loud.
- Keep notes on your examples, the questions you want to ask, and areas you are still learning.
- Be specific about your own contribution and candid about gaps.
- Keep answers direct, then add technical detail when they ask for it.

## Company references

- [VSL product overview](https://www.vslglobal.ai/)
- [VSL use cases](https://www.vslglobal.ai/use-cases)
- [VSL privacy policy](https://www.vslglobal.ai/privacy-policy)
