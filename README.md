# Backend Interview Prep

**📖 Read this as a website: https://abialemuel.github.io/backend-interview-prep/** — searchable, dark mode, auto-updates on every push.

A structured, categorized study repository for backend engineering interviews. Covers languages, frameworks, databases, infrastructure/DevOps, plus system design, DSA, and behavioral prep. Build a reusable, technology-agnostic foundation you can adapt to whatever stack you work with.

Each topic is organized into:
- **Core concepts** — the fundamentals you must understand
- **Deep dives** — advanced patterns, internals, and best practices
- **Interview questions** — Q&A with model answers to self-test

> Content is kept current against the latest stable releases of each tool covered in the topic folders. Always cross-check the official docs for the latest before an interview.

---

## Repository Structure

```
backend-interview-prep/
├── README.md                          ← you are here (GitHub landing page)
├── mkdocs.yml                         ← website config (MkDocs Material)
├── .github/workflows/deploy.yml       ← auto-deploys site to GitHub Pages on push
└── docs/                              ← all study content lives here
    ├── index.md                       ← website home page
    ├── STUDY-GUIDE.md                 ← how to use this repo + study plan
    ├── 01-languages/
    │   ├── php/                       ← PHP 8.x core, OOP, performance, security
    │   ├── go/                        ← Go core, concurrency, interfaces, testing
    │   └── rust/                      ← ownership, borrowing, concurrency, traits, ecosystem
    ├── 02-frameworks/
    │   ├── laravel/                   ← architecture, Eloquent, queues, security
    │   └── nextjs/                    ← rendering, data fetching, API routes
    ├── 03-databases/
    │   ├── mysql/                     ← indexing, transactions, replication, schema
    │   └── redis/                     ← data structures, persistence, clustering
    ├── 04-aws/                        ← compute, storage, networking, IAM, monitoring
    ├── 05-devops/
    │   ├── terraform/                 ← IaC core, state, modules, best practices
    │   ├── ansible/                   ← config management, playbooks, roles
    │   ├── github-actions/            ← CI/CD workflows, runners, secrets
    │   └── datadog/                   ← observability, metrics, APM, logs
    ├── 06-system-design/              ← scalability, caching, microservices
    ├── 07-data-structures-algorithms/ ← patterns + practice problems
    ├── 08-behavioral/                 ← STAR method + common questions
    ├── 09-containers-and-kubernetes/  ← Docker, K8s core & advanced, debugging
    ├── 10-messaging-and-event-streaming/ ← Kafka, delivery guarantees, event-driven patterns
    ├── 11-api-design/                 ← REST, gRPC, GraphQL, idempotency, rate limiting
    ├── 12-security-and-auth/          ← sessions, JWT, OAuth2/OIDC, OWASP
    ├── 13-distributed-systems/        ← consensus, replication, clocks, correctness
    └── 14-ai-llm-integration/         ← LLM APIs, RAG, vector search, production LLM systems
```

## How to Use

1. Start with [`STUDY-GUIDE.md`](./docs/STUDY-GUIDE.md) for a recommended study plan and timeline.
2. Work top-to-bottom through each category. Read the core concepts first, then attempt the interview questions **before** reading the answers.
3. Track weak areas and revisit. Re-read model answers and rephrase them in your own words.

## Categories at a Glance

| # | Category | Focus | Files |
|---|----------|-------|-------|
| 1 | Languages | PHP, Go, Rust | core, OOP/modern, concurrency, ownership/borrowing, performance, security, Q&A |
| 2 | Frameworks | Laravel, Next.js | architecture, ORM, routing/queues, rendering, Q&A |
| 3 | Databases | MySQL, PostgreSQL, Redis | indexing, transactions, replication, MVCC, data structures, persistence, Q&A |
| 4 | AWS | Infrastructure | compute, storage, networking, databases, IAM, monitoring, Q&A |
| 5 | DevOps | Terraform, Ansible, GHA, Datadog | IaC, config mgmt, CI/CD, observability, Q&A |
| 6 | System Design | Architecture | scalability, load balancing, caching, microservices, Q&A |
| 7 | DSA | Problem solving | common patterns, practice problems, Q&A |
| 8 | Behavioral | Soft skills | STAR method, common questions |
| 9 | Containers & K8s | Docker, Kubernetes | images, deployments, probes, autoscaling, debugging, Q&A |
| 10 | Messaging | Kafka, queues | delivery guarantees, consumer groups, outbox, sagas, CDC, Q&A |
| 11 | API Design | REST, gRPC, GraphQL | HTTP semantics, pagination, versioning, idempotency, rate limiting, Q&A |
| 12 | Security & Auth | AppSec, identity | sessions, JWT, OAuth2/OIDC, passkeys, OWASP, Q&A |
| 13 | Distributed Systems | Theory + practice | consensus, replication, partitioning, locks, retries, Q&A |
| 14 | AI & LLM | LLM integration | LLM APIs, RAG, vector search, guardrails, cost control, Q&A |

## Contributing / Extending

This repo is designed to be extended. To add a new topic:
1. Create a new numbered directory under `docs/` (e.g., `docs/09-new-topic/`).
2. Add a `README.md` with an overview and a list of sub-files.
3. Follow the existing file-naming convention: `NN-concept-name.md` for content and `NN-interview-questions.md` for Q&A.
4. Optionally add a `.pages` file (`title: Pretty Name`) for a clean sidebar label.
5. Push — the website rebuilds and shows the new section automatically. No config edits needed.

## Sources & References

Content is synthesized from official documentation and well-known references. Always cross-check against the latest official docs before relying on specifics:
- PHP: https://www.php.net/docs.php
- Go: https://go.dev/doc/
- Laravel: https://laravel.com/docs
- Next.js: https://nextjs.org/docs
- MySQL: https://dev.mysql.com/doc/
- Redis: https://redis.io/docs/
- AWS: https://docs.aws.amazon.com/
- Terraform: https://developer.hashicorp.com/terraform/docs
- Ansible: https://docs.ansible.com/
- GitHub Actions: https://docs.github.com/actions
- Datadog: https://docs.datadoghq.com/

## Disclaimer

This is a study aid, not an exhaustive reference. Interview expectations vary by company. Use the interview-question files to practice articulating concepts out loud — communication matters as much as correctness.
