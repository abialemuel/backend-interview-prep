# Talabat — Backend Engineer HR Interview

Preparation for **Abia Darma Lemuel**, based on **Abia New CV.pdf** and the supplied Talabat JD. The confirmed interview is **HR**. The exact role title, team, location, duration, and later interview sequence have not been supplied; confirm them with the recruiter.

## Start here

| Page | What it prepares |
|---|---|
| [HR questions and spoken answers](01-hr-questions-and-answers.md) | Introduction, why Talabat, moving from Careem, strengths, ownership, feedback, mentoring, compensation, and logistics |
| [Engineering culture](02-engineering-culture.md) | XP, TDD, pair/mob programming, DDD, small batches, quality, trunk-based development, and continuous delivery |
| [Rehearsal and project stories](03-rehearsal-and-stories.md) | Your strongest examples, missing details to prepare, mock questions, and final call notes |

## Your strongest positioning

> Senior backend engineer with over eight years of experience, strong Go/Ruby foundations, directly relevant food-delivery integration work at Careem, production reliability experience, and a track record of mentoring and building services from scratch.

For this HR call, explain the user or business outcome before implementation details. Your AlArabi integration is a particularly relevant example because it connects engineering to restaurant operations.

## What the supplied JD emphasizes

The role combines backend engineering with discovery, customer empathy, shared problem-solving, small increments, and responsibility through deployment and operation. The JD explicitly names **XP, DDD, Lean, Continuous Delivery, TDD, pair/mob programming, refactoring, and trunk-based development**.

Talabat's official hiring page also emphasizes alignment with culture as well as ability to perform the role. The actual interview questions and internal evaluation rubric have not been confirmed. [Official hiring page](https://careers.deliveryhero.com/how-we-hire-at-talabat)

## CV-to-role mapping

| Role need | Documented evidence | What to explain |
|---|---|---|
| Food-delivery domain and end-to-end problems | Careem AlArabi integration: onboarding, catalog, orders, delivery tracking, promotions | The operational problem, your scope, collaboration, and how work reached users |
| Distributed systems and Go | Careem microservices, Kafka/SQS/DynamoDB; Go as primary language | A concrete service and its failure behavior |
| Reliability | CERT relay error classification, ordered retries, and backoff | Why silent event drops mattered and how you addressed them |
| Product ownership | First engineer at RRQ; backend launch with product leadership | Requirements, decisions, trade-offs, and delivery |
| Performance and simplification | Telkom stateless monitoring redesign, Kubernetes autoscaling, Go worker pools | Roughly 80% lower memory usage; prepare measurement details |
| Mentoring and improving practices | Telkom mentoring and backend standards; shared trace library | One real example of feedback and its effect |
| Databases, cloud, and observability | PostgreSQL, MySQL, Redis, DynamoDB; AWS/GCP/Azure; Datadog/OTel and others | Separate tools you operated deeply from those you have used |
| Test automation and delivery | RSpec in Bukalapak stack; GitHub Actions deployment pipeline at Hubbedin; CI/CD tools listed | A real test or pipeline example; tooling alone does not prove TDD or Continuous Delivery |
| .NET/C# | Not listed in the CV; Go is listed in the JD | Confirm the actual team language and expectations |
| Formal XP, pair/mob programming, trunk-based development | Not established by the CV | Describe your actual practice precisely and your willingness to learn |

## Lead with these stories

1. **Careem AlArabi:** direct business relevance and integration delivery. The CV reports significantly reduced manual onboarding work but no numerical percentage.
2. **Careem CERT:** responsibility for reliable processing and explicit recovery.
3. **Telkom mentoring:** evidence of helping the team improve, rather than relying only on individual delivery.
4. **Telkom monitoring:** measurable performance improvement; the 80% metric applies to memory usage, not cost or the AI platform.
5. **RRQ:** working with product leadership and building from zero.

The [story guide](03-rehearsal-and-stories.md) identifies details to prepare for each. The CV does not provide a complete example of disagreement, a personal mistake, or receiving feedback; choose genuine examples before the call.

## What to confirm with HR

- Exact title, level, team, product domain, and hiring location.
- Actual language mix, especially Go versus .NET/C#.
- Working arrangement and relocation/work-authorization support, if applicable.
- Approved compensation range and whether it refers to base salary or a package.
- What follows HR, whether there is a collaborative coding session, and how to prepare.

Your current work location, visa status, reason for considering a move, notice period, salary expectation, and relocation readiness remain personal facts to fill accurately. The Careem employer location on your CV does not establish your current residence or authorization.

## References

- Your supplied JD: source for the role's culture and responsibilities.
- Abia New CV.pdf: source for professional experience; private contact details are omitted.
- [Talabat hiring](https://careers.deliveryhero.com/how-we-hire-at-talabat)
- [Life at Talabat](https://careers.deliveryhero.com/life-at-talabat)
- [Official backend vacancy with similar responsibilities](https://careers.deliveryhero.com/job/software-engineer-ii-backend-in-dubai-uae-jid-4869): context only; not confirmation that this is your exact vacancy.
