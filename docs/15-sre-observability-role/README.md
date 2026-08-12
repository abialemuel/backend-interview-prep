# Senior Software Engineer / SRE — Observability Focus

This section is a targeted prep pack for one specific job posting shape: a **60–70% software engineering / 30–40% SRE** hybrid role, observability-focused, Kubernetes + AWS + Datadog stack, remote APAC contract with JST-overlap preference, enterprise environment. If that is the loop you are walking into, read this file first — it decodes the JD, maps every bullet to the deep-dive material already in this repo, fills the gaps that are specific to *this* posting (API-key/RBAC management, CI/CD observability integration, Python ops automation), and ends with a graded question bank and behavioral prep tuned to the exact phrasing used.

## 1. Decoding the JD

Strip the posting to what it is actually testing for:

| JD phrase | What it really means | Where interviewers probe it |
| --- | --- | --- |
| "60–70% SWE / 30–40% SRE" | You will write real code (services, APIs, automation) most of the time, but you own production behavior for a slice of systems too. Not a pure ops role, not a pure feature role. | Expect a coding exercise **and** a troubleshooting/incident scenario in the same loop. |
| "Enterprise experience strongly preferred" | Change management, ticket-driven work, multiple stakeholder teams, existing legacy tooling you must integrate with rather than replace. | Behavioral questions about navigating process, not just technical questions. |
| "Design, build, and maintain software, APIs, integrations, and automation" | You are expected to *write* the tools other engineers use to see and control the platform, not just consume a vendor dashboard. | Coding round likely involves calling or wrapping an API (Datadog's, AWS's, or an internal one). |
| "Build and maintain observability solutions with a focus on Datadog" | Datadog is the primary tool named — expect deep, specific Datadog questions, not generic "what is observability." | See [Datadog deep dive](../05-devops/datadog/README.md) — read it in full before this loop. |
| "Configure dashboards, alerts, APM, tracing, metrics, logging" | All four Datadog signal types plus the config layer (monitors, dashboards) — not just "we use Datadog." | Be ready to describe a monitor's threshold type, a dashboard's template variables, a trace's sampling policy, from memory. |
| "Integrate observability into AWS environments" | IAM roles for the Datadog integration, CloudWatch metric streams, Lambda log forwarding — not application code, infrastructure wiring. | Section 3.5 below. |
| "Integrate observability into CI/CD pipelines" | Deployment markers, synthetic/health-check gates, pipeline-as-telemetry (CI Visibility). | Section 3.4 below. |
| "Automate monitoring and operational tasks through scripting, Python preferred" | You are expected to script against the Datadog API (and AWS SDK), not just click in the UI. | Section 3.2 below — practice writing one of these cold. |
| "Manage API keys and secure configurations" / "manage user roles, permissions, access controls" | Datadog org administration: API vs application keys, Teams, custom roles, restricted access on dashboards/monitors. | Section 3.3 below — this is a distinct skill from "using" Datadog and is easy to under-prepare. |
| "Lead proactive maintenance… drive improvements… rather than only reactive support" | The nice-to-haves make this explicit: they want someone who finds and fixes problems before they page, not someone who only responds to pages. | Behavioral section — bring a specific story, not a general claim. |
| "Remote, APAC, JST overlap preferred, 40h/week, contract to 2027-03-31" | A fixed-term contractor role with a real end date — expect questions about contract-to-perm expectations, notice periods, and how you structure a working day with distributed teammates. | Section 5 — logistics questions to ask them. |

## 2. Coverage map — what's already in this repo

Read in this order; each links to the file that covers it in depth. This section only covers what's **not** already handled elsewhere.

| JD requirement | Repo coverage | Read this first if... |
| --- | --- | --- |
| Kubernetes deployment/ops/monitoring | [Containers & Kubernetes](../09-containers-and-kubernetes/README.md) — core + advanced + debugging | you haven't run a cluster hands-on recently |
| Datadog / observability platform, dashboards, alerts, APM, tracing, metrics, logs | [Datadog](../05-devops/datadog/README.md) — three pillars, Agent, SLOs, OTel | anything below assumes you've read this |
| AWS hands-on | [AWS](../04-aws/README.md) — compute/storage, networking/DB, IAM/security/monitoring | rusty on IAM roles or CloudWatch |
| API design, integration, consuming/implementing APIs | [API Design](../11-api-design/README.md) — REST/HTTP, gRPC/GraphQL, operations (idempotency, rate limiting, versioning) | you haven't designed a public-facing API contract before |
| Python/Node/Java proficiency, general SWE | [Languages](../01-languages/go/README.md) (Go/PHP tracks — apply the same rigor to whichever language you interview in) | — |
| CI/CD pipelines | [GitHub Actions](../05-devops/github-actions/README.md), [Terraform](../05-devops/terraform/README.md) | you haven't wired a pipeline gate before |
| Containerized/microservices monitoring | [System Design — caching & microservices](../06-system-design/02-caching-and-microservices.md) + Kubernetes advanced | — |
| Reliability, scalability, performance ownership | [System Design — scalability](../06-system-design/01-scalability-and-load-balancing.md), [Distributed Systems](../13-distributed-systems/README.md) | — |
| Behavioral / STAR | [Behavioral](../08-behavioral/README.md) | before Section 6 below |

## 3. The gaps — content specific to this posting

### 3.1 Framing the 60/40 split in an answer

When asked "walk me through your background" or "why this role," explicitly narrate the split rather than letting the interviewer infer it: *"Day to day I'm writing Go/Python services and APIs — that's the 60–70%. The other third is being on the hook for whether those services (and the platform underneath them) stay healthy: dashboards, alerting, on-call, postmortems."* This single sentence signals you read the JD carefully, which matters more than it sounds like at the senior level — it's the difference between "can do the job" and "already understands the job."

### 3.2 Automating observability with Python (scripting the platform, not just using it)

This is the most testable, most under-prepared JD line. Interviewers will ask you to sketch — or actually write — a script that touches the Datadog API. Know the shape of the client and the three things you're commonly automating: creating/updating monitors, managing dashboards, and bulk-tagging.

```python
from datadog_api_client import Configuration, ApiClient
from datadog_api_client.v1.api.monitors_api import MonitorsApi
from datadog_api_client.v1.model.monitor import Monitor
from datadog_api_client.v1.model.monitor_type import MonitorType
from datadog_api_client.v1.model.monitor_options import MonitorOptions

configuration = Configuration()  # reads DD_API_KEY / DD_APP_KEY from env

def upsert_latency_monitor(service: str, env: str, threshold_ms: int) -> None:
    query = (
        f'avg(last_5m):avg:trace.http.request.duration'
        f'{{service:{service},env:{env}}} > {threshold_ms}'
    )
    body = Monitor(
        name=f"[{env}] {service} p95 latency high",
        type=MonitorType("metric alert"),
        query=query,
        message=(
            f"{{{{#is_alert}}}}{service} p95 latency over {threshold_ms}ms in {env}. "
            f"@slack-{env}-alerts{{{{/is_alert}}}}"
        ),
        tags=[f"service:{service}", f"env:{env}", "managed-by:automation"],
        options=MonitorOptions(thresholds={"critical": threshold_ms}, notify_no_data=True),
    )
    with ApiClient(configuration) as api_client:
        MonitorsApi(api_client).create_monitor(body=body)
```

Talking points to hit when you present something like this, unprompted:

- **Idempotency of the automation itself** — re-running the script should update, not duplicate, the monitor. The real pattern is "monitors-as-code": store the JSON/YAML definitions in git, diff on PR, apply via script or Terraform's `datadog` provider in CI. Mention this explicitly — it's the enterprise-grade answer over "I'd click through the UI and script the repetitive parts."
- **Secrets handling** — `DD_API_KEY`/`DD_APP_KEY` come from a secrets manager or CI secret store, never hardcoded; this ties directly into the "secure configurations" bullet (3.3).
- **Failure mode** — what happens if the API call fails mid-batch when bulk-updating 200 monitors? Answer: make each upsert independent and idempotent (as above) so a partial run is safe to re-run, and log/alert on the failures rather than silently swallowing them.
- **Beyond monitors** — the same client pattern automates: bulk-tagging hosts/services after a reorg, exporting dashboard JSON for backup/version control, and querying the API to build a custom compliance report (e.g., "which services have zero monitors").

### 3.3 API keys, secure configuration, and access control (Datadog org administration)

This is genuinely a distinct skill from "using" Datadog day to day, and it's called out as its own bullet — prepare it as its own topic.

- **API key vs application key.** An **API key** authenticates the *submission* of data into an org (Agents, dogstatsd, the intake API) — one org typically has few, long-lived API keys, often per-environment. An **application key** authenticates *API calls made on behalf of a user* (dashboards-as-code scripts, the Terraform provider) and inherits that user's permissions — so revoking a departed employee's app key is part of offboarding, and app keys should be scoped per-service-account, not shared. Confusing the two is a common interview tell that you haven't operated Datadog beyond installing the Agent once.
- **Rotation.** Both key types should be rotatable without downtime: provision the new key, roll it out to Agents/scripts, confirm ingestion/calls succeed on the new key, then revoke the old one — never revoke-then-provision.
- **Secrets at rest.** API keys live in a secrets manager (AWS Secrets Manager/SSM Parameter Store, Vault) and are injected into the Agent via environment variable or a mounted secret (Kubernetes `Secret` + `DD_API_KEY` env, not baked into the DaemonSet manifest or a container image layer).
- **Teams and RBAC.** Datadog's access model layers **Teams** (group services/dashboards/monitors by ownership — who gets paged) on top of **custom roles** (fine-grained permissions: `dashboards_write`, `monitors_downtime`, `logs_read_data` scoped by restriction query) assigned via **role-based or attribute-based** access. Enterprise environments almost always disable the default "Datadog Admin gives everyone everything" posture in favor of least-privilege custom roles — know that this is the pattern to recommend even if you haven't configured it yourself.
- **Restricted access on individual assets.** Beyond org-wide roles, a specific dashboard or monitor can be locked to specific teams/roles — relevant when a platform team owns infra dashboards that shouldn't be editable by every app team with Datadog access.
- **SSO/SCIM.** Enterprise orgs provision users via SAML SSO + SCIM (auto-deprovisioning on offboarding) rather than local Datadog accounts — worth a one-sentence mention if asked how you'd manage access at scale.

### 3.4 Integrating observability into CI/CD

Three distinct integration points — name all three if asked "how does observability fit into your pipeline":

1. **Deployment markers.** The pipeline calls the Datadog API (or the `datadog-ci` CLI) to emit a deployment event at deploy time, tagged with `service`, `env`, `version`/git SHA. This lets every dashboard and monitor overlay "a deploy happened here" — the single highest-leverage signal for correlating a regression with its cause.

    ```yaml
    # GitHub Actions step, after a successful deploy
    - name: Notify Datadog of deployment
      run: |
        curl -X POST "https://api.datadoghq.com/api/v1/events" \
          -H "DD-API-KEY: ${{ secrets.DD_API_KEY }}" \
          -H "Content-Type: application/json" \
          -d '{
            "title": "Deploy: checkout-service",
            "text": "Deployed '"${GITHUB_SHA}"' to production",
            "tags": ["service:checkout-service", "env:prod", "deploy:true"]
          }'
    ```

2. **Pipeline health as telemetry (CI Visibility).** Test and pipeline execution itself gets traced — flaky-test detection, build-time regressions, and failure-rate dashboards for the pipeline, not just the app it builds.
3. **Observability as a deploy gate.** Post-deploy, a synthetic API test or a canary-analysis monitor (error rate/latency of the new version vs. baseline) gates promotion or triggers auto-rollback — this is the connective tissue between "SRE" and "CI/CD" the JD is testing for: deploys aren't done when they ship, they're done when telemetry confirms health.

### 3.5 Integrating observability into AWS

- **IAM-role-based AWS integration.** Datadog's AWS integration authenticates via a cross-account IAM role (an external-ID-scoped role Datadog assumes) rather than long-lived access keys — know this as the answer to "how would you connect Datadog to an AWS account" and be ready to name why (no long-lived credentials to rotate/leak).
- **CloudWatch Metric Streams.** For services without a native Datadog integration, CloudWatch Metric Streams push metrics via Kinesis Firehose to the Datadog intake in near-real-time, replacing the older poll-the-CloudWatch-API model (which is both higher-latency and rate-limited). This is the current recommended pattern — mention it over "the Agent polls CloudWatch."
- **Log forwarding.** The Datadog Forwarder Lambda subscribes to CloudWatch Log Groups (and S3 events, Kinesis) and ships logs to Datadog — the standard way to get Lambda/ECS/RDS logs in without installing an Agent on unmanaged compute.
- **EKS specifics.** On EKS, the Datadog Agent runs as a DaemonSet (node-level) plus a Cluster Agent Deployment (cluster-level API aggregation, admission controller for auto-instrumentation) — same architecture as any Kubernetes install (see [the Datadog Agent](../05-devops/datadog/01-observability-and-apm.md#the-datadog-agent)), with the AWS-specific addition of the Datadog integration pulling in EC2/EKS control-plane context alongside what the Agent sees inside the cluster.

### 3.6 API integration design (consuming and implementing)

The JD calls out API integration skills as their own bullet, separate from "SRE." Ground this in [API Design](../11-api-design/README.md) but bring the SRE-flavored angle specifically:

- When **consuming** a third-party API (Datadog's, PagerDuty's, an internal service's) for automation, design for its failure modes: respect its rate limits and `Retry-After`, use idempotent retries, and treat the client as a small owned module with its own tests — not inline `requests.post()` calls scattered through scripts.
- When **implementing** an internal API that other teams' automation will consume (e.g., an internal "service catalog" API that observability tooling queries for ownership metadata), apply the same operational rigor from [API Operations](../11-api-design/03-api-operations.md): versioning, pagination, clear error contracts — because at platform scale, *your* API becomes infrastructure other people's reliability depends on.

## 4. Scenario question — design the whole thing end to end

This is the shape of system-design question most likely to appear for this exact role. Practice it out loud before the interview.

> **"We're running a microservices platform on EKS, deployed via a GitHub Actions pipeline, and we've been told to 'get proper observability in with Datadog.' Walk me through your approach."**

A strong answer moves through, in order:

1. **Instrumentation layer** — Agent as DaemonSet + Cluster Agent for K8s-native metadata; admission controller for auto-instrumentation of APM where possible, custom spans/tags only for business-critical paths; dogstatsd for custom app metrics with a tagging convention agreed up front (`service`, `env`, `team`, `version`) to avoid a cardinality mess later.
2. **AWS wiring** — cross-account IAM role integration for AWS-level context (EKS control plane, RDS, ALB), CloudWatch Metric Streams for anything without a native Agent integration, Forwarder Lambda for any non-EKS logs (Lambda functions, RDS logs).
3. **Signal config** — dashboards per service with template variables for env, SLO-based monitors (not raw thresholds) for the handful of things that page, burn-rate alerting on error budgets, tail-based trace sampling with a retention filter that keeps 100% of errors.
4. **CI/CD hook-in** — deployment markers on every deploy, a synthetic smoke test or canary-error-rate check gating promotion, CI Visibility on the pipeline itself.
5. **Governance** — custom roles/Teams so the right people can edit the right dashboards, monitors-as-code in git so config changes go through PR review, key rotation policy for API/app keys.
6. **The proactive layer** (this is the line that answers the "own reliability, not just reactive support" nice-to-have) — a recurring review of alert noise (are monitors flapping? are engineers ignoring pages?), a quarterly SLO review, and a backlog of platform reliability work that competes for sprint time against features, not something squeezed in during firefighting.

Naming all six layers, unprompted, is what separates a senior answer from a mid-level one — most candidates stop at step 3.

## 5. Behavioral prep — this JD's specific angle

The nice-to-haves spell out exactly what story they want: **owning** an internal platform, **owning** reliability/scalability/performance outcomes, and **proactively** leading improvements rather than only reacting. Prepare STAR stories (see [Behavioral](../08-behavioral/README.md) for the method) that hit these specifically — generic "I fixed a production incident" stories under-perform here because they default to reactive framing.

- *"Tell me about a time you improved a platform before it became a problem."* — needs a story with a **leading indicator** you noticed (rising latency trend, growing log volume, an alert that fires just under threshold repeatedly) that you acted on **before** a page, not a postmortem-driven fix.
- *"Tell me about owning something end-to-end."* — pick a story where you made the call on architecture, alerting thresholds, or a rollout plan yourself, not one where you executed someone else's spec.
- *"Tell me about working with an existing/legacy setup you didn't design."* — enterprise-flavored; they want evidence you can integrate with what exists rather than always advocating a rewrite.
- *"Tell me about disagreeing with a decision about reliability priorities."* — tests whether you can push back (e.g., "we need to fix alert fatigue before adding new monitors") without stalling delivery.

## 6. Technical question bank

Graded roughly junior → senior; the senior-tier ones are where this specific loop will spend most of its time.

**Core / must-have**

1. What's the difference between a metric, a log, and a trace, and what does "high cardinality" have to do with whether a system is actually observable versus merely monitored?
2. Walk me through what happens between an application emitting a custom metric via dogstatsd and that metric appearing on a dashboard.
3. Your `p95` latency dashboard shows a metric type of HISTOGRAM. A teammate wants to slice it by `customer_id`, which wasn't in the original tags. Why can't they, and what would you change?
4. Describe the difference between a readiness probe and a liveness probe, and what happens to in-flight traffic if you get them backwards.
5. You're integrating Datadog with an AWS account that has no existing monitoring. What's the recommended way to authenticate the integration, and why not access keys?

**Applied / senior**

6. Design an alerting strategy for a service with an SLO of 99.9% over 30 days. What do you alert on, and why is a raw error-rate threshold the wrong primary signal?
7. A monitor is flapping — alerting and resolving every few minutes. Walk through your diagnosis and fix, including how you'd prevent the same class of flapping elsewhere.
8. You need to roll out a new Datadog API key across 40 services without downtime or a gap in data. What's your rollout and rollback plan?
9. Write (or describe in detail) a script that audits every Datadog monitor in an org and flags ones with no `service` tag or no notification channel attached.
10. How would you gate a deployment pipeline on post-deploy health, and what's your rollback trigger if the canary looks unhealthy?
11. A trace shows a request spending 40ms in a downstream call the service map says shouldn't exist. What's your hypothesis, and how do you confirm it?
12. How do you decide what's a custom metric vs. relying on integration metrics, and what's the cost consequence of getting it wrong at scale?
13. Explain head-based vs. tail-based trace sampling and why tail-based lets you claim "100% of errors captured" at a fraction of the ingestion cost.
14. Your logs bill tripled last month with no traffic increase. Walk through how you'd find the cause and the levers to bring it back down.
15. Design the RBAC model for a Datadog org shared by five teams, where platform-owned infra dashboards must not be editable by app teams but must be readable by everyone.

**Nice-to-have / breadth**

16. Compare Datadog to Prometheus + Grafana as an observability stack for a mid-size platform — what does each optimize for, and when would you pick one over the other?
17. Where does OpenTelemetry fit if the org standardizes on Datadog as the backend — what do you gain and what do you give up by instrumenting with OTel vs. Datadog's native tracer libraries?
18. You inherit a platform with reactive-only monitoring (pages fire, nobody looks until something breaks). What's your first-90-days plan to shift it toward proactive?

Model answers for the SLO/alerting design (Q6), the sampling comparison (Q13), and the OTel trade-off (Q17) are already written out in the [Datadog README](../05-devops/datadog/README.md) — use it to check your answers, don't just read the questions cold.

## 7. Logistics — this is a contract, ask about it

This posting is a fixed-term remote contract (through **2027-03-31**), APAC-based, JST-overlap preferred. Worth asking the interviewer directly rather than guessing:

- What hours of JST overlap are actually expected — a specific window, or just "reachable during incidents"?
- Is there a path from contract to permanent, or is the end date firm?
- Who is the on-call rotation shared with, and what does the escalation path look like across time zones?
- What's already in place vs. greenfield — an existing Datadog setup you're improving, or a from-scratch build?

## 8. Suggested prep order

1. [Datadog](../05-devops/datadog/README.md) — full read, both files.
2. [Containers & Kubernetes](../09-containers-and-kubernetes/README.md) — core + advanced.
3. [AWS](../04-aws/README.md) — IAM/security/monitoring file specifically.
4. [API Design — operations](../11-api-design/03-api-operations.md).
5. This file's Sections 3–4, out loud, until you can give the six-layer answer in Section 4 without notes.
6. [Behavioral](../08-behavioral/README.md) — draft the four stories in Section 5 before the interview, not during it.
