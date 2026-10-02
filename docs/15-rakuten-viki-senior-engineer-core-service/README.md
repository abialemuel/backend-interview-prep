# Rakuten Viki — Senior Engineer, Core Services

Preparation for **Abia Darma Lemuel**, based on **Abia New CV.pdf**, the supplied job description, and the recruiter messages shared on **2 October 2026**.

**Codility: passed.** Your confirmed next interviews are a Talent Acquisition exploratory call and a hiring-manager fit interview. They are progressive: passing the first is required to proceed to the second, and either can come first. Both focus on overall fit. The TA call with **Ler Qi** is **30 minutes**; the hiring-manager duration has not been specified.

## Start here

| Preparation | What to practise |
|---|---|
| [Recruiter exploratory call](01-recruiter-exploratory-call.md) | Introduction, motivation, career history, relocation, compensation, availability, and questions for Ler Qi |
| [Hiring-manager fit interview](05-hiring-manager-fit.md) | Ownership, engineering judgment, production reliability, project delivery, mentoring, and product fit |
| [Final rehearsal](06-final-rehearsal.md) | A short study plan, mock interview questions, answer-quality checklist, and notes to prepare |
| [System design practice](02-system-design.md) | Optional technical depth if they ask how your experience applies to streaming systems |
| [Historical technical question bank](03-interview-questions.md) | Candidate-reported questions from other hiring processes; these do not establish your interview sequence |
| [Technical model answers](04-model-answers.md) | Background practice for technical follow-ups; prioritize the fit interviews first |

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

Your expected salary, notice period, current work location, reason for considering a move, and relocation timeline have not been confirmed in this conversation. Prepare accurate answers; the CV does not establish these logistics.

For compensation, distinguish **monthly SGD base**, **annual base**, and **annual total compensation**. Ask for the role's approved range and package structure before choosing an anchor. The earlier guide's fixed S$10,000–S$11,000 target was not your stated expectation or Viki's confirmed budget.

For relocation, ask whether this vacancy supports employer-sponsored Employment Pass applications and what assistance is available. MOM's current eligibility page sets a non-financial-sector minimum of S$5,600 that rises with age, with a higher schedule for new applications from 1 January 2027; COMPASS generally also applies. A salary threshold does not establish your eligibility. The employer can assess the application using MOM's Self-Assessment Tool.

The practical interview answer is a realistic start plan, subject to your notice obligations, pass approval, and agreed relocation arrangements.

## Confirmed process versus historical reports

The recruiter messages you supplied take priority over candidate reports. Coding, CS fundamentals, and system-design questions in this pack are background practice from other processes, rather than scheduled rounds for your application. Ask Ler Qi what follows the two fit interviews if you progress.

## References

- Your supplied job description and recruiter messages: source for role scope, TA duration, progressive interviews, and flexible order.
- Abia New CV.pdf: source for your professional experience. Private contact details are not included in this preparation pack.
- [Viki subtitle creation](https://support.viki.com/hc/en-us/articles/115015389988-Subtitle-Creation-on-Viki)
- [Viki Pass, subtitles, and regional restrictions](https://support.viki.com/hc/en-us/articles/115009896968-Does-Viki-Pass-include-subtitles-and-access-to-regionally-restricted-content)
- [MOM Employment Pass eligibility](https://www.mom.gov.sg/passes-and-permits/employment-pass/eligibility)
- [MOM Employment Pass application process](https://www.mom.gov.sg/passes-and-permits/employment-pass/apply-for-a-pass)
