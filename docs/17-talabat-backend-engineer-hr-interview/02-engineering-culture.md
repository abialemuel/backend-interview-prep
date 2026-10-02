# Talabat Engineering Culture — HR Preparation

The supplied JD makes working practices part of role fit. Prepare a simple explanation of each practice and a truthful account of how you have used it. Your CV supports mentoring, delivery pipelines, production improvements, and backend standards; it does not establish regular TDD, pairing, mob programming, formal DDD, or trunk-based development.

## Explain the terms simply

| Practice | Plain explanation | What to connect to the role |
|---|---|---|
| Extreme Programming (XP) | A set of engineering practices that support frequent feedback, collaboration, testing, and evolving design | How the team can change software safely and learn quickly |
| TDD | Write a failing test for a behavior, make it pass, then improve the design while keeping it passing | Fast feedback on both behavior and interfaces |
| Pair programming | Two engineers work together on the same problem, actively discussing and reviewing decisions | Shared understanding, immediate feedback, and learning |
| Mob programming | A group works together on one problem, with participation and keyboard roles rotated | Collective problem-solving and fewer knowledge bottlenecks |
| Simple design and refactoring | Keep the design understandable and improve its structure while preserving behavior | Easier future changes and fewer unnecessary dependencies |
| DDD | Model business rules with shared terminology and explicit domain boundaries | Better alignment between code, product decisions, and operations |
| Lean | Focus on value, feedback, and improving the flow of work | Less waiting, unnecessary scope, and rework |
| Small batches | Deliver useful changes in small increments | Earlier learning and easier diagnosis when a change fails |
| Trunk-based development | Integrate small changes frequently into a shared mainline; keep branches short-lived when used | Reduce integration delays and keep feedback current |
| Continuous Delivery | Keep changes ready to release through repeatable builds, checks, and deployment automation | Release safely when the business needs it |

Continuous Delivery does not require automatically deploying every passing change. That describes Continuous Deployment. [Continuous Delivery](https://martinfowler.com/bliki/ContinuousDelivery.html)

Further reading: [TDD](https://martinfowler.com/bliki/TestDrivenDevelopment.html), [pair programming](https://martinfowler.com/articles/on-pair-programming.html), [DDD](https://martinfowler.com/bliki/DomainDrivenDesign.html), [trunk-based development](https://trunkbaseddevelopment.com/).

## What does “quality enables speed” mean to you?

Suggested answer:

> I understand it as making changes easier to trust. Clear design, automated feedback, and a dependable release process help engineers find mistakes early and spend less time recovering from them. Small changes also make it easier to see what caused a problem. Under a deadline, I would look for a smaller useful scope while keeping the safeguards the behavior needs.

Connect this to your CERT reliability work: handling failure explicitly reduced a class of silent event loss. Do not claim a measured increase in delivery speed unless you actually measured it.

## How would you work in a pair?

Suggested approach, not a claim about past pairing frequency:

> I would agree on the goal and the next behavior to implement, explain my reasoning, and invite the other engineer's view. We would switch roles so both people contribute. If we disagree, I'd try to make the assumption concrete through a test or small experiment. I also want the session to improve shared understanding, especially when one person knows the area better.

For a more experienced partner, ask questions and contribute observations. When mentoring a less experienced partner, give them time to reason and participate. A code review or mentoring session is related collaboration, but does not establish regular pair programming.

## What if pairing feels slower?

> I'd look at the complete outcome: time to a correct change, rework, and how much knowledge is shared. Pairing can help with uncertain or risky work and with learning a new area. I'd seek feedback on how the team pairs effectively and discuss improvements if sessions aren't helping us make progress.

This demonstrates curiosity without promising that every session has the same benefit.

## How would you use TDD?

Use a food-delivery example as a proposed exercise:

> For an order-state rule, I could first write a failing test describing which transition is allowed. Then I'd implement the smallest change that satisfies it and refactor while keeping the tests passing. I would add important cases such as an invalid transition or repeated event. That helps clarify the rule before introducing broader infrastructure concerns.

Prepare one real test-first example if you have it. Writing tests after implementation is useful test automation, but it is not the same practice as TDD. [TDD explanation](https://martinfowler.com/bliki/TestDrivenDevelopment.html)

## What does automated end-to-end quality mean?

Suggested answer:

> I would use automated checks at the level where they give useful confidence. Focused tests cover business rules; integration or contract checks cover boundaries; a smaller set of complete journeys checks that important user workflows connect correctly. The checks should give clear feedback and run reliably. I would also use production observability to understand behavior after release.

Example checks for an order workflow, if asked: an accepted order progresses correctly, repeated events do not create duplicate effects, and an unavailable dependency has defined behavior. These are proposed checks, not claims about tests in your Careem codebase.

Your CV lists RSpec and CI/CD tools. Prepare a real example explaining what ran, when it ran, and which failure it caught. Tool names do not show the breadth or effectiveness of a testing strategy.

## How would DDD apply to food delivery?

> I would start with the business language: what an order, payment, catalog item, and delivery mean to the people operating the product. Then I would clarify the rules and boundaries between those areas. For example, an order being accepted and a payment being captured are distinct facts. Clear ownership of those facts helps teams avoid ambiguous integrations.

DDD can influence service boundaries, but it does not require a new microservice for every noun. Your AlArabi work gives you relevant domain examples; describe actual boundaries from that project only when you can explain them. [DDD explanation](https://martinfowler.com/bliki/DomainDrivenDesign.html)

## How would you deliver a change in small batches?

Proposed catalog-integration example:

> I would agree on one useful partner workflow and the outcome we want to improve. I could start with a limited catalog update, validate it with a small rollout, and use the results to decide the next increment. I'd consider interface compatibility and recovery from failure so the initial change can be delivered safely.

Choose a real incremental-delivery example if available. Do not retrospectively describe an entire AlArabi rollout as small-batch delivery unless that is how it happened.

## What is your Continuous Delivery experience?

> At Hubbedin, I implemented a GitHub Actions pipeline for GCP deployments. My CV also covers other CI/CD tooling. I can explain the pipeline stages I actually built and how a change moved to deployment.
>
> For this role, I'd like to understand how the team keeps releases ready, which automated checks it relies on, and how production changes are monitored and recovered.

Prepare branch strategy, checks, deployment trigger, rollback, and any manual approvals from your actual project. A deployment pipeline alone does not prove frequent integration or an always-releasable service.

## What is your trunk-based development experience?

If you have used it, explain how frequently changes were integrated, how reviews worked, and how incomplete features stayed safe. If you have not used it consistently, a truthful answer is:

> I understand the aim of frequent integration and short-lived changes, but I haven't consistently worked in a trunk-based team. I'd want to learn your review and release conventions, contribute small changes, and help keep the mainline healthy.

Use that limitation only if accurate. Trunk-based development can still include code review and short-lived branches. [Reference](https://trunkbaseddevelopment.com/)

## How would you improve the team's practices?

> I would first understand where the team loses time or confidence: waiting for reviews, flaky checks, unclear ownership, or difficult deployments. I'd discuss one issue with the team, try a small improvement, and check whether it helped. I would want others involved in the change so the practice is useful and sustainable.

Your Telkom standards, mentoring, and trace-library work are evidence that you have contributed to team practices. Prepare how you got feedback and whether colleagues adopted the change; the CV does not describe the adoption process.

## What to remember for HR

Speak in terms of learning, customer impact, and how colleagues work together. Give one real example when possible. Familiarity with the vocabulary supports the conversation; actual experience and openness to feedback make the answer credible.
