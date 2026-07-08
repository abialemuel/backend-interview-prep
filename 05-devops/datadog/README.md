# Datadog

Datadog is a SaaS observability and monitoring platform that unifies metrics, logs, traces, real-user monitoring (RUM), synthetic tests, CI visibility, and cloud security posture into one product with a shared tagging model. Instead of stitching together Prometheus + Loki + Tempo + Grafana + PagerDuty + Sentry, you get one bill and one query language across all signals. The trade-off is vendor lock-in and a host/log/APM-based pricing model that demands discipline to control.

This directory covers the observability model and the Datadog product surface most relevant to backend interviews.

Sub-files:

| File | Topic |
| --- | --- |
| `01-observability-and-apm.md` | Three pillars, the Agent, metrics, APM/tracing, logs, dashboards/monitors, SLOs, RUM, synthetic, OpenTelemetry, AWS integration, cost |
| `02-interview-questions.md` | Graded interview questions with model answers |

Suggested reading order: read the observability and APM reference first (the three pillars and tracing model are the conceptual core), then attempt the questions and revisit sections as gaps surface.