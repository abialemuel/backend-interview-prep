# AWS — Backend Engineer Interview Prep

This section covers AWS services and architectures that backend engineers are
expected to understand when designing, building, and operating infrastructure on
AWS. The emphasis is on **building and operating infrastructure using AWS**:
which service solves which problem, what the trade-offs are, and how an
interviewer will probe your choices.

The mental model threaded throughout these notes is the **AWS Well-Architected
Framework**. Most architecture questions are implicitly asking whether your
design respects its six pillars:

1. **Operational Excellence** — running and monitoring systems to deliver
   business value, automating changes, responding predictably to events.
   Think: IaC (CloudFormation/Terraform), IaC drift detection, runbooks,
   SSM, CloudWatch, X-Ray, deployments with rollback (blue/green, canary).
2. **Security** — protecting data, systems, and assets. Identity (IAM least
   privilege), detection (GuardDuty, CloudTrail, Config), protection (WAF,
   Shield, SGs), data protection (KMS, encryption in transit/at rest).
3. **Reliability** — recovering from failures and meeting demand. Multi-AZ,
   Multi-Region where justified, RTO/RPO, auto scaling, health checks,
   idempotency, backpressure, retries with jitter, circuit breakers.
4. **Performance Efficiency** — using the right resource type and size as
   demand changes. Graviton, NVMe instance store, read replicas, DAX,
   CloudFront, intelligent tiering.
5. **Cost Optimization** — avoiding unnecessary cost. Right-sizing, Savings
   Plans / Reserved Instances, Spot for fault-tolerant batch, lifecycle
   policies, Graviton price/perf, stopping/staggering non-prod.
6. **Sustainability** — understanding the impact of your workloads and
   maximizing utilization (right-sizing), avoiding idle resources, choosing
   efficient instance types and regions, and minimizing data movement.

A common interview heuristic: when asked "design X on AWS", name your service
choices and justify them against **two or more pillars** that were traded off —
e.g. "I'd use Lambda for an event-driven ingestion pipeline for cost and
operational efficiency, but I'd add provisioned concurrency for reliability
against cold starts and a DLQ for failed events."

## Files in this section

| File | Description |
|------|-------------|
| `01-compute-and-storage.md` | EC2, ECS, EKS, Lambda, App Runner/Lightsail; S3, EBS, EFS, instance store; the compute + storage decision framework. |
| `02-networking-and-databases.md` | VPC, Route 53, CloudFront, ELB, API Gateway, Direct Connect/VPN; RDS/Aurora, DynamoDB, ElastiCache; data warehouse & search brief. |
| `03-iam-security-and-monitoring.md` | IAM, STS, KMS, Secrets Manager/Parameter Store; GuardDuty/Macie/Inspector/Config/Security Hub/WAF/Shield/ACM; CloudWatch, X-Ray, CloudTrail, EventBridge, SSM, cost tooling. |
| `04-interview-questions.md` | 48 interview questions grouped by Easy/Medium/Hard with a junior/senior/staff grading rubric, and model answers focused on trade-offs. |

## Recommended reading order

1. `01-compute-and-storage.md` — foundational primitives; most design questions
   start with "where does the compute run and where does the data live".
2. `02-networking-and-databases.md` — how the compute talks to users (networking
   + edge) and to data stores. Many reliability questions live here (HA, RR,
   failover).
3. `03-iam-security-and-monitoring.md` — security and operations layered on
   top of 01 + 02. Interviews love asking "and how do you operate/secure that?"
4. `04-interview-questions.md` — work through after the others; use them as
   self-test and as a forced-recall review tool.

## How to use this material

- **Trade-offs first.** Almost every AWS question is a trade-off question.
  Whenever you read a feature, ask: _when would I pick this, and when would I
  avoid it?_
- **Defaults and limits.** Memorize the handful of numbers that drive design
  choices (Lambda timeout 15 min / memory 128 MB–10 GB; DynamoDB item 400 KB;
  S3 single PUT 5 GB / multipart 5 TB; EBS gp3 baseline 3000 IOPS; Aurora
  Serverless v2 scales 0–256 ACU; RDS Multi-AZ standby is in a different AZ
  with synchronous replication; S3 has been strongly consistent since Dec
  2020). Numbers drift over releases — verify against current AWS
  documentation.
- **Know the current era.** Interviewers in 2026 expect Graviton4 (`8g`)
  instances as the price/perf default, IAM Identity Center instead of IAM
  users, multi-account Organizations structure, serverless cold-start
  mitigation (SnapStart, provisioned concurrency), and a coherent cost
  story. Naming deprecated generations or "create an IAM user for the app"
  dates you instantly.
- **Pillars as a checklist.** In a system-design answer, walk through the
  pillars explicitly; interviewers reward structured thinking.

> Conventions: service behavior described reflects AWS as of mid-2026. Always
> re-verify against current AWS documentation for exact limits and pricing,
> since these evolve.