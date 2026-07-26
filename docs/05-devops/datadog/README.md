# Datadog

Datadog is a SaaS observability and monitoring platform that unifies metrics, logs, traces, real-user monitoring (RUM), synthetic tests, CI visibility, database monitoring, cloud security, and — as of the mid-2020s — on-call paging (Datadog On-Call), LLM observability, and AI-assisted investigation (Bits AI) into one product with a shared tagging model. Instead of stitching together Prometheus + Loki + Tempo + Grafana + PagerDuty + Sentry, you get one bill and one query language across all signals. The trade-off is vendor lock-in and a host/log/APM-based pricing model that demands discipline to control — which is why OpenTelemetry, the vendor-neutral instrumentation standard Datadog now ingests natively (including via its DDOT Collector distribution), features in nearly every 2026 observability interview.

This directory covers the observability model and the Datadog product surface most relevant to backend interviews, plus the vendor-neutral concepts (OpenTelemetry, SLO/SLI/error budgets) interviewers expect regardless of tooling.

Sub-files:

| File | Topic |
| --- | --- |
| `01-observability-and-apm.md` | Three pillars, the Agent, metrics, APM/tracing, logs, dashboards/monitors, SLOs, RUM, synthetic, OpenTelemetry, AWS integration, cost |
| `02-interview-questions.md` | Graded interview questions with model answers |

Suggested reading order: read the observability and APM reference first (the three pillars and tracing model are the conceptual core), then attempt the questions and revisit sections as gaps surface.