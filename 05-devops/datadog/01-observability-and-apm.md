# Observability and APM

## The three pillars — and what observability actually means

The "three pillars of observability" are **metrics**, **logs**, and **traces**. Datadog implements each as a first-class signal and stitches them together with shared tags and a `trace_id` correlation so you can pivot from a metric spike to the logs and traces of the requests that caused it.

But the pedantic definition of **observability** (from control theory) is a property of a system: *you can understand its internal state from its external outputs.* A monitored system is not necessarily observable — if your only signals are `cpu_usage` and `request_count`, a spike from a specific user's bad payload is invisible because you lack the cardinality to slice by `user_id`. The practical test for observability is high-cardinality: can you group by `user_id`, `tenant_id`, `request_path`, `feature_flag`, `build_sha` without pre-aggregating? Datadog supports that via tags, which is why tag discipline is the single most important habit.

## The Datadog Agent

The Agent is a daemon you install on each host (or run as a sidecar/DaemonSet). It has three responsibilities:

1. **Collect host metrics and events** — CPU, memory, disk, network, plus integrations (nginx, postgres, redis, kubelet).
2. **Forward logs and traces** — applications send logs via the Agent's log collector and traces via the trace agent; the Agent batches and ships them.
3. **Run integration checks** — each integration is a small config (`conf.d/postgres.d/conf.yaml`) that tells the Agent how to talk to a service and which metrics to collect.

```
Host ──► Datadog Agent ──► Datadog intake (metrics/logs/traces)
              ▲
              │ integrations (postgres, nginx, kubelet, ...)
Applications ── dogstatsd (metrics) / trace library (traces) / log forwarder (logs)
```

On Kubernetes the **Cluster Agent** runs as a Deployment and talks to the API server for metadata, while DaemonSet Agents on each node collect pod metrics and traces. The **admission controller** (cluster agent webhook) can auto-inject the tracer library and standard tags into pods at creation, so you get tracing without modifying every deployment.

## Metrics

Datadog metric types:

| Type | Meaning | Example |
| --- | --- | --- |
| **GAUGE** | a value at a point in time | `system.mem.used` |
| **COUNT** | a number accumulated since the last flush | `http.request.count` (per flush interval) |
| **RATE** | a count normalized to per-second | derived from a count |
| **HISTOGRAM** | agent-side percentiles (p50/p95/p99/max) computed and stored as separate metrics | `http.request.latency` |
| **DISTRIBUTION** | raw values sent to Datadog, server-side percentiles; supports arbitrary group-by without pre-aggregation | `http.request.latency.distribution` |

HISTOGRAM computes percentiles on the Agent and stores them as fixed metrics (`p95`, `max`), which is cheap but locks the aggregation at submission time — you cannot later slice that p95 by `user_id` if you did not submit per-user. DISTRIBUTION sends raw values to Datadog and computes percentiles server-side, so you can group by any tag at query time; the cost is higher ingestion. Use DISTRIBUTION when you need flexible slicing; HISTOGRAM when you know the groupings in advance.

### Tags and cardinality

Every metric carries tags — `env:prod, service:api, region:us-east-1, version:1.4.2`. Tags are how you slice and aggregate at query time: `avg:http.request.latency{service:api,env:prod} by version`. The cost trap is **high-cardinality tags**: a tag like `request_id` or `user_id` with millions of distinct values multiplies the number of stored time series (each unique tag combination is a separate series) and explodes your custom-metric bill. Tag by dimensions you will actually aggregate over (`service`, `env`, `version`, `endpoint`), not by unique identifiers. Custom metrics (any tag combination not coming from an integration) are billed per unique combination, so cardinality is a cost issue, not just a query issue.

### Custom vs integration metrics

Integration metrics come from a supported integration and are included in the host price. Custom metrics are anything you submit yourself (via dogstatsd, API, or a tag combination not emitted by an integration) and billed per unique tag combination per month. Keep custom metrics bounded by design.

### dogstatsd

Applications submit custom metrics via dogstatsd, a statsd-compatible UDP/UDS protocol:

```python
from datadog import initialize, statsd

initialize(statsd_host="127.0.0.1", statsd_port=8125)

statsd.increment("api.requests", tags=["endpoint:/checkout", "env:prod"])
statsd.histogram("api.latency_ms", 42.3, tags=["endpoint:/checkout"])
statsd.gauge("queue.depth", 1024, tags=["queue:orders"])
```

The Agent listens on `127.0.0.1:8125` (or a UDS socket for lower overhead), aggregates, and flushes to Datadog every ~10s. UDS (Unix domain socket) is preferred over UDP for high-throughput apps because it avoids packet loss.

## APM / tracing

### Spans and traces

A **span** is a unit of work — an HTTP request handler, a DB query, a downstream service call — with a start time, duration, service name, operation name, resource, and tags. A **trace** is a tree of spans linked by parent/child relationships representing one request's path through the system. The root span is the entry point; child spans are nested operations.

```
trace (root span: GET /checkout)
├── span: postgres SELECT users (18ms)
├── span: http call to pricing-service (40ms)
│   ├── span: redis GET sku:123 (2ms)
│   └── span: postgres SELECT prices (12ms)
└── span: kafka publish order.created (3ms)
```

Each span carries a `service` (which service produced it), `operation` (what kind of work), `resource` (the specific endpoint/query/template), and arbitrary `tags` (`user_id`, `tenant_id`, `error`, `http.status_code`).

### Distributed tracing and context propagation

For a trace to span multiple services, the trace context — `trace_id`, `parent_span_id`, sampling decisions — must be passed between them. Datadog's tracer libraries inject headers (`x-datadog-trace-id`, `x-datadog-parent-id`, `x-datadog-sampling-priority`) on outbound HTTP/gRPC/Kafka calls and extract them on the inbound side, so a request from `api` to `pricing` to `db` produces one connected trace. With OpenTelemetry, the equivalent headers are `traceparent`/`tracestate` (W3C) or B3 (Zipkin). Context propagation is what makes distributed tracing work; if a service does not propagate, the trace breaks at that hop.

### Automatic vs custom instrumentation

Datadog tracer libraries auto-instrument popular libraries (Express, Django, Flask, Rails, Spring, gin, net/http, psycopg, the AWS SDK) with zero code changes — you get HTTP and DB spans out of the box. **Custom instrumentation** adds spans or tags for business logic the auto-instrumentation cannot see:

```python
from ddtrace import tracer

@tracer.wrap("checkout.process_order", service="checkout")
def process_order(order_id, user_id):
    with tracer.trace("db.lookup_user") as span:
        span.set_tag("user_id", user_id)
        user = db.find_user(user_id)
    with tracer.trace("pricing.calculate_total"):
        total = pricing.total(order_id)
    return total
```

The rule: rely on auto-instrumentation for the framework/DB layer; add custom spans only for business-critical paths and add tags (`user_id`, `tenant_id`, `feature_flag`, `cart_size`) that you will query later. Tags are where the observability of traces comes from.

### Trace sampling

The volume of traces at production traffic is too high to ship them all. Two sampling strategies:

- **Head-based sampling** — the tracer decides at the root whether to sample; the decision propagates in the headers, so either the whole trace is sampled or none of it. Simple, consistent, but if the decision is made before the request fails you may miss errors (a 0.1% sample rate rarely catches the 0.01% of requests that error).
- **Tail-based sampling** — the Agent buffers the complete trace, then decides whether to keep it based on the full picture: errors, high latency, specific tags. Datadog uses tail-based sampling by default for APM, which is why it can guarantee 100% of error traces are captured even at a low overall sample rate. This is the killer feature for troubleshooting: you sample away the boring successful traces and keep the interesting ones.

Tune with retention filters: keep 100% of traces with `error:1`, 100% of traces slower than 2s, and 1% of everything else.

### Service map, flame graphs

The **service map** infers service-to-service dependencies from observed traces and draws a live topology with throughput, error rate, and latency per edge. The **flame graph** view shows a single trace as a waterfall of spans — width is duration, nesting is parent/child — so you can see at a glance which downstream call dominated the request. Both are derived from the trace store, so they reflect real traffic, not a static config.

### Trace / log / metric correlation

Every span carries a `trace_id`; Datadog injects `@trace_id` and `@span_id` into log records when the log library is configured (often via a log formatter or MDC). In the UI you click a slow span and pivot to "logs for this trace" or "metrics for this service" — same time window, same tags. The `trace_id` is the join key that makes the three pillars one workflow instead of three tools.

## Logs

Datadog ingests logs via the Agent's log collector (tail files, journald, container stdout) or the HTTP API. Once ingested, logs pass through **pipelines** — ordered **grok** parsers that extract structured attributes from raw text — and end up as JSON with facets you can filter and aggregate on.

- **Facets** are indexed attributes (`service`, `env`, `status`, `@request.path`, `@user.id`) that drive log search and analytics.
- **Log-based metrics** generate a metric from log query results, so you can alert on "count of ERROR logs per minute" without scraping.
- **Log indexes** are subsets of logs kept hot for querying; the cost model charges by the GB ingested and by the volume retained in indexes. Logs are the most expensive Datadog signal, so the discipline is: ingest only what you will query, parse with pipelines to extract facets, and archive the rest to S3 for cold storage.

Log ingestion pipelines can drop noisy logs at the source (Agent `log_processing_rules` with exclude matchers) before they count against your ingestion volume. Tier your logs: keep ERROR and WARN hot in indexes, INFO in a shorter-retention index, DEBUG archived to S3 only.

## Dashboards and monitors

A **dashboard** is a grid of widgets — timeseries, toplist, query value, heatmap, change, scatter — backed by metric, log, trace, or RUM queries. Dashboards support template variables (`$service`, `$env`) that rebind every widget's query, so one dashboard covers every service.

A **monitor** is an alert on a query. Types:

- **Metric monitor** — threshold on a metric query (`avg:http.request.latency{service:api} by env > 250`).
- **Anomaly monitor** — detects deviation from a learned baseline using forecasting; good for "something is unusual" without a fixed threshold.
- **Outlier monitor** — flags a group (e.g. one pod) whose behavior diverges from its peers.
- **Forecast monitor** — predicts when a metric will cross a threshold (e.g. disk will fill in 3 days).
- **Composite monitor** — boolean combination of other monitors (A and not B).
- **Log monitor** — alert on a log query count or pattern.
- **Trace analytics monitor** — alert on APM metrics (error rate, p95 latency) grouped by service/env.

Each monitor has alert and warning thresholds, multi-alert over tag groups (alert per `env` or per `service` instead of one global value), and a notification message templated with the breached value. Route notifications to PagerDuty, Slack, email, or webhooks via **integration** destinations.

### SLOs and error budgets

An **SLO** (Service Level Objective) is a target on a service's reliability over a window — "99.9% of checkout requests succeed over 30 days." Datadog computes SLOs from metric monitors (success/total ratio) or from log/trace queries, and tracks the **error budget** — how many bad events you can still afford within the window before breaching. When the error budget is burning fast, you get a burn-rate alert that says "at this rate you will breach in 2 hours," which is more actionable than a raw error-rate alert. SLOs turn monitoring into a product-level conversation rather than a per-signal one.

### Synthetic monitoring and RUM

**Synthetic monitoring** runs API and browser tests from managed locations on a schedule — a check that hits `https://app.example.com/health` every minute from 12 regions and alerts on non-200 or latency regression. Use it for external-facing endpoints and to catch outages before users do. **API tests** are HTTP/gRPC assertions; **browser tests** drive a headless browser through a click flow.

**RUM** (Real User Monitoring) collects performance and error data from real user browsers and mobile apps — page load timing, frontend errors, XHR latency, session replay. RUM correlates with backend traces via the `trace_id` injected into XHR headers, so a user-reported "checkout is slow" can pivot from the RUM session to the backend trace that served it.

## Infrastructure

- **Host map** — color-coded grid of every host by a chosen metric; good for spotting the odd one out.
- **Live processes** — process list per host (like `top` but fleet-wide).
- **Containers / Kubernetes** — the Agent's container integration ships cgroup metrics, pod metadata, and kube-state-metrics; the Cluster Agent enriches with labels/annotations and provides the admission controller for auto-instrumentation.

## Integration ecosystem

Datadog ships 400+ integrations. The AWS integration is the canonical one: you create an IAM role that Datadog assumes, which lets the Agent pull EC2/CloudWatch metrics, CloudTrail events, and resource metadata. For services Datadog does not have a native integration for, you can stream CloudWatch metrics directly via Kinesis Firehose to the Datadog intake. CloudWatch metric streams avoid the API-per-metric polling cost of the older integration model.

## APM and OpenTelemetry

Datadog supports OpenTelemetry via OTLP ingest: applications instrumented with the OTel SDK can export traces and metrics to the Agent's OTLP receiver, and Datadog ingests them. The advantage is **vendor neutrality** — you are not locked into Datadog's tracer libraries and can switch backends by changing the exporter. The trade-off is that OTel's auto-instrumentation coverage and some Datadog-specific features (inferred spans, profiling integration, the trace-to-log correlation injection) are richer with the native libraries. A common compromise: use OTel SDK for custom instrumentation and tags, export via OTLP, and keep Datadog as the backend; you can switch to Tempo or Honeycomb later with a config change.

## Events and incidents

The **Incident Management** module turns a monitor alert into a structured incident: declare from the alert, assign a severity, attach a Slack channel, track a timeline, and run a postmortem template afterward. It integrates with on-call providers (PagerDuty, Opsgenie) for paging and with Slack for the incident channel. The value is a single record per incident that captures the timeline, the responders, and the impact, rather than reconstructing it from chat logs after the fact.

## Cost management

Datadog pricing is per-host (infrastructure + APM) plus per-GB (logs) plus per-custom-metric-combination (custom metrics). The levers:

- **Custom metrics** — every unique tag combination is a custom metric. Avoid high-cardinality tags (`user_id`, `request_id`); prefer DISTRIBUTION metrics where you need flexible slicing rather than per-series tagging.
- **Logs** — drop noisy logs at the Agent, tier indexes by retention, archive cold logs to S3. Logs are usually the biggest line item.
- **APM hosts** — traced hosts are billed separately from infrastructure hosts; trace only the hosts whose services you need to observe.
- **Profiling, RUM, synthetic, CI Visibility** — each is an add-on SKU with its own unit; turn on only where the value justifies it.

The discipline is: tag consistently, drop at the source what you will not query, use integrations over custom metrics, and tier log retention.