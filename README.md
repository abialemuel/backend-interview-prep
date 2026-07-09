# Backend Interview Prep

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
├── README.md                      ← you are here
├── STUDY-GUIDE.md                 ← how to use this repo + study plan
├── 01-languages/
│   ├── php/                       ← PHP 8.x core, OOP, performance, security
│   └── go/                        ← Go core, concurrency, interfaces, testing
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
├── 06-system-design/              ← scalability, caching, microservices, API design
├── 07-data-structures-algorithms/ ← patterns + practice problems
└── 08-behavioral/                 ← STAR method + common questions
```

## How to Use

1. Start with [`STUDY-GUIDE.md`](./STUDY-GUIDE.md) for a recommended study plan and timeline.
2. Work top-to-bottom through each category. Read the core concepts first, then attempt the interview questions **before** reading the answers.
3. Track weak areas and revisit. Re-read model answers and rephrase them in your own words.

## Categories at a Glance

| # | Category | Focus | Files |
|---|----------|-------|-------|
| 1 | Languages | PHP, Go | core, OOP/modern, concurrency, performance, security, Q&A |
| 2 | Frameworks | Laravel, Next.js | architecture, ORM, routing/queues, rendering, Q&A |
| 3 | Databases | MySQL, Redis | indexing, transactions, replication, data structures, persistence, Q&A |
| 4 | AWS | Infrastructure | compute, storage, networking, databases, IAM, monitoring, Q&A |
| 5 | DevOps | Terraform, Ansible, GHA, Datadog | IaC, config mgmt, CI/CD, observability, Q&A |
| 6 | System Design | Architecture | scalability, load balancing, caching, microservices, API design, Q&A |
| 7 | DSA | Problem solving | common patterns, practice problems, Q&A |
| 8 | Behavioral | Soft skills | STAR method, common questions |

## Contributing / Extending

This repo is designed to be extended. To add a new topic:
1. Create a new numbered directory (e.g., `09-new-topic/`).
2. Add a `README.md` with an overview and a list of sub-files.
3. Follow the existing file-naming convention: `NN-concept-name.md` for content and `NN-interview-questions.md` for Q&A.
4. Link the new category in this README's structure tree and table.

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
