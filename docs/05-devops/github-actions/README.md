# GitHub Actions

GitHub Actions is GitHub's built-in CI/CD platform. Workflows are YAML files in `.github/workflows/` that trigger on repository events (push, pull request, schedule, manual dispatch) and run jobs on runners — either GitHub-hosted VMs or your own self-hosted machines. Each job is a sequence of steps that either run shell commands or invoke reusable **actions**. It is the natural CI/CD choice for repositories already on GitHub and has become the default for new projects.

This directory covers the workflow model and the CI/CD patterns most relevant to backend interviews.

Currency note (mid-2026): the patterns interviewers now treat as table stakes are **OIDC to cloud providers instead of long-lived keys**, **actions pinned by SHA** (post the 2025 tj-actions supply-chain compromise; immutable actions are GitHub's structural fix), **artifact/cache actions v4** (v3 was shut down in early 2025), **merge queues** for keeping `main` green, reusable workflows for org-wide standardization, and signed build provenance via artifact attestations. `ubuntu-latest` runs Ubuntu 24.04, and hosted arm64 and larger runners are mainstream.

Sub-files:

| File | Topic |
| --- | --- |
| `01-workflows-and-ci-cd.md` | Workflow syntax, jobs/steps, runners, secrets, OIDC, caching, reusable workflows, composite actions, security |
| `02-interview-questions.md` | Graded interview questions with model answers |

Suggested reading order: read the workflows reference first (the syntax and security material are tightly coupled), then attempt the questions and revisit sections as gaps surface.