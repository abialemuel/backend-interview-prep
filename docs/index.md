# Backend Interview Prep

A structured, categorized study site for backend engineering interviews. Covers languages, frameworks, databases, infrastructure/DevOps, plus system design, DSA, and behavioral prep — with a focus on what world-class interview loops actually ask today.

Each topic is organized into:

- **Core concepts** — the fundamentals you must understand
- **Deep dives** — advanced patterns, internals, and best practices
- **Interview questions** — Q&A with model answers to self-test

!!! tip "How to use this site"
    Start with the [Study Guide](STUDY-GUIDE.md) for a recommended plan and timeline. Read core concepts first, then attempt the interview questions **before** reading the answers. Use the search bar (press ++slash++) to jump to any concept.

## Categories

| Category | Focus |
|----------|-------|
| [Languages](01-languages/go/README.md) | Go, PHP — core, concurrency, OOP, performance, security |
| [Frameworks](02-frameworks/laravel/README.md) | Laravel, Next.js — architecture, ORM, rendering |
| [Databases](03-databases/mysql/README.md) | MySQL, PostgreSQL, Redis — indexing, transactions, replication, caching |
| [AWS](04-aws/README.md) | Compute, storage, networking, IAM, monitoring |
| [DevOps](05-devops/terraform/README.md) | Terraform, Ansible, GitHub Actions, Datadog |
| [System Design](06-system-design/README.md) | Scalability, caching, microservices |
| [DSA](07-data-structures-algorithms/README.md) | Common patterns + practice problems |
| [Behavioral](08-behavioral/README.md) | STAR method + common questions |
| [Containers & Kubernetes](09-containers-and-kubernetes/README.md) | Docker, K8s core & advanced, debugging |
| [Messaging & Event Streaming](10-messaging-and-event-streaming/README.md) | Kafka deep dive, delivery guarantees, event-driven patterns |
| [API Design](11-api-design/README.md) | REST, gRPC, GraphQL, idempotency, rate limiting, versioning |
| [Security & Auth](12-security-and-auth/README.md) | Sessions, JWT, OAuth2/OIDC, OWASP, passkeys |
| [Distributed Systems](13-distributed-systems/README.md) | Consensus, replication, clocks, correctness in practice |
| [AI & LLM Integration](14-ai-llm-integration/README.md) | LLM APIs, RAG, vector search, LLMs in production |

## Keeping content current

Content is kept current against the latest stable releases of each tool. Always cross-check the official docs for the latest before an interview.

## Contributing / Extending

This site builds itself from Markdown. To add a new topic:

1. Create a new numbered directory under `docs/` (e.g., `09-new-topic/`).
2. Add a `README.md` with an overview — it becomes the section's index page.
3. Follow the naming convention: `NN-concept-name.md` for content and `NN-interview-questions.md` for Q&A.
4. Optionally add a `.pages` file with `title: Pretty Name` for a clean sidebar label.
5. Push. The site rebuilds and the new section appears automatically.
