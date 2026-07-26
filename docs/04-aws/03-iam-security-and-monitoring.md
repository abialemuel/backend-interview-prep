# IAM, Security & Monitoring on AWS

This file covers identity, secrets, encryption, the security tooling layer,
and operations/observability/cost. The interview theme: every architecture
answer must include an explicit security and operations layer — services
that "just work" on day 1 will hurt you on day 365 if you skipped IAM
least-privilege, CloudTrail audit, and CloudWatch alarms.

---

## IAM & Security

### IAM — Identity and Access Management

Core concepts:

- **Users** — long-term identity with credentials (password for console,
  access key pair for API). **Effectively legacy for humans**: current AWS
  guidance is no IAM users at all — humans authenticate through **IAM
  Identity Center**, workloads use roles with short-lived STS credentials,
  and external systems (e.g., GitHub Actions) federate via **OIDC** instead
  of stored access keys. Keep IAM users only for the rare break-glass or
  legacy-integration case, with MFA and rotation.
- **Groups** — container of users with shared policies.
- **Roles** — identities assumed by **trusted principals** (AWS services,
  users, or services in other accounts) for a limited time; credentials
  delivered through STS. **Roles are the recommended primitive** for AWS
  service-to-service authorization (e.g., an EC2 instance role, a Lambda
  execution role, an ECS task role) and for cross-account access.
- **Policies** — JSON documents. Identity-based (attached to user/group/
  role) or resource-based (attached to a resource like an S3 bucket).
  **Permission boundaries** cap the maximum effective permissions of a
  principal — used to delegate IAM administration safely (e.g., let
  developers create roles but bound them by a boundary that forbids
  `*:*` or admin actions). **SCPs** at the Organizations level cap the
  account-level max — a different boundary mechanism; an SCP that
  denies `s3:*` denies even the root account.

A canonical identity-based policy:

```jsonc
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOwnObjectsPrefix",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::app-uploads-${aws:PrincipalAccount}/*",
      "Condition": {
        "StringEquals": {
          "s3:prefix": "${s3:prefix}",
          "aws:PrincipalTag/team": "backend"
        }
      }
    }
  ]
}
```

Building blocks interviewers expect you to recall:
**Effect** (Allow/Deny), **Action** (e.g., `s3:GetObject` or
`s3:*`), **Resource** (ARN), **Condition** (keys: `aws:SourceIp`,
`aws:SourceArn` for SSE-from-event, `aws:PrincipalTag/*`,
`aws:MultiFactorAuthPresent`, `aws:RequestedRegion`, `s3:prefix`,
`kms:ViaService`), **Principal** (only in resource-based policies).
**Explicit Deny wins** — even if another statement allows. Conditions
support `StringEquals`, `ArnLike`, `IpAddress`, `DateGreaterThan`,
`Bool`, etc.

**Identity-based vs resource-based**:
- Identity-based: "what can this principal do". Attached to user/role.
- Resource-based: "who can access this resource". Attached to the
  resource (S3 bucket, KMS key, SQS queue, Secrets Manager secret, SNS
  topic, Lambda invoke policy). Enables **cross-account** access without
  the caller assuming a role in your account — the resource policy grants
  the foreign principal directly. For cross-account use, **both** the
  caller's identity-based policy AND the resource's resource-based policy
  must allow the action (a logical AND), except for S3 and some services
  where the resource policy alone is enough.

**Least privilege** is the discipline of writing policies as granular as
the workload tolerates. Tools:
- **IAM Access Analyzer** — generates policies from CloudTrail activity
  ("this role only used X in last 90 days"), validates policy correctness,
  surfaces external access (resources shareable to accounts outside your
  org), and unattached/unused access findings.
- **IAM Access Analyzer policy validation** — catches wildcards,
  deprecated actions, etc.
- CloudTrail + Athena queries for "what did this role actually call".
- **IAM Identity Center (formerly SSO)** — single sign-on across all
  accounts in an Organization with one directory (internal or external
  IdP via SAML 2.0 / SCIM, including Okta, Entra ID, Google); per-account
  permission sets. Replaces the legacy per-account IAM users model and is
  **the AWS-recommended way** for human access.

### STS — Security Token Service

Issues short-term credentials (access key + secret + session token) for a
role. Calls:
- **AssumeRole** — most common; principal passes a role ARN, optional
  external ID, optional session policy (downscopes), gets creds valid
  15min–12h (default 1h).
- **AssumeRoleWithWebIdentity** — exchange an OIDC token for role creds.
  Very much alive: it's how **GitHub Actions deploys without stored AWS
  keys**, and how EKS pod identities (IRSA) work. For human SSO, use IAM
  Identity Center instead.
- **AssumeRoleWithSAML** — for SAML IdPs.
- **GetSessionToken** — for MFA-enforced sessions for the calling user.

**Cross-account AssumeRole** convention:
- The trusting account (the one holding the resource) defines the role
  with a **trust policy** allowing the foreign principal
  (`arn:aws:iam::OTHER-ACCOUNT:role/...`).
- The trusting account adds a **Condition** requiring an **ExternalId**
  passed by the caller — a magic string the caller chooses; prevents the
  "confused deputy" problem (a malicious third party from the trusted
  account trying to invoke your role).
- The calling account's identity-based policy must allow
  `sts:AssumeRole` for that role ARN (or the role's trust policy can be
  resource-based-only).

Example trust policy with external ID:

```jsonc
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
    "Action": "sts:AssumeRole",
    "Condition": { "StringEquals": { "sts:ExternalId": "customer-456-random-string" } }
  }]
}
```

### Organizations & the multi-account pattern

Interviewers increasingly ask "how do you structure AWS accounts for a
growing org" — the account, not the VPC, is the real blast-radius,
billing, and quota boundary. The standard shape:

- **AWS Organizations** — a management account (no workloads in it!) plus
  member accounts arranged in **OUs** (organizational units), e.g.
  `Security`, `Infrastructure`, `Workloads/Prod`, `Workloads/NonProd`,
  `Sandbox`.
- **Foundational accounts**: a **Log Archive** account (org CloudTrail +
  Config + VPC flow logs land in locked-down S3) and a **Security
  Tooling** account (delegated admin for GuardDuty, Security Hub,
  Inspector, Access Analyzer across the org).
- **One account per workload per environment** (`payments-prod`,
  `payments-staging`) — isolates blast radius, gives clean cost
  attribution, and per-account service quotas.
- **SCPs / Resource Control Policies** attached to OUs cap what *any*
  principal in those accounts can do — deny leaving allowed regions, deny
  disabling CloudTrail/GuardDuty, deny root actions. SCPs don't grant,
  they bound.
- **IAM Identity Center** for all human access with permission sets per
  OU/account; **centralized root access management** (2024+) lets the org
  remove root credentials from member accounts entirely — a modern
  best-practice talking point.
- **Control Tower** automates this landing zone (account factory,
  guardrails); Terraform/CDK-based landing zones are the common
  alternative.
- Cross-account wiring: roles + `sts:AssumeRole` for control plane, **RAM**
  (Resource Access Manager) for sharing subnets/TGW, resource policies or
  PrivateLink/VPC Lattice for data plane.

### KMS — Key Management Service

Symmetric and asymmetric **KMS keys** (AWS retired the "CMK / Customer
Master Key" name, though interviewers still say CMK) backed by
FIPS-validated HSMs, or, for the highest compliance bar, **Custom Key
Store** backed by your own CloudHSM cluster or an external key store
(XKS) outside AWS.

**Envelope encryption** — the model:

1. You ask KMS to **generate a data key** (plaintext + encrypted copy
   under the CMK).
2. You encrypt your payload **locally** with the plaintext data key.
3. You discard the plaintext key from memory, store the encrypted data
   key alongside the ciphertext.
4. To decrypt, you ask KMS to **Decrypt** the encrypted data key using
   the CMK (after the calling principal's policy + key policy allow
   it), then decrypt the payload locally.

Why: the **CMK never leaves KMS**; only the small data key is exposed
to the caller briefly. Per-object data keys mean each object has its
own key, and one compromised data key doesn't compromise other
objects. You get per-object CloudTrail entries for KMS Decrypt calls,
which gives auditability.

**Keys:**
- **AWS-managed keys** (`aws/s3`, `aws/rds`, etc.) — free, rotated annually,
  usable by any principal in the account for that service. Don't use
  when you need cross-account control or granular audit per key.
- **Customer-managed keys (CMK)** — you define the key policy, enable
  rotation (usually annually, or on-demand for asymmetric keys),
  cross-account access via key policy + caller IAM, pay per key-month
  and per API call.
- **Aliases** — friendly names (`alias/prod-payments`) decoupled from the
  underlying key ID; rotate the underlying key without changing code.

**Grants** — a lighter-weight alternative to key policies for programmatic
delegation; e.g., a service grants *its own* short-term permission to a
CMK on your behalf (S3, EBS, and Lambda need grants to use your CMK for
server-side encryption). Use grants when you need to programmatically
delegate KMS use without IAM policy churn.

**Quotas** — region quotas on **Decrypt/GenerateDataKey** calls; shared
defaults around 5,500–100,000 RPS depending on region/key type. S3
SSE-KMS GETs/PUTs hit `GenerateDataKey`/`Decrypt` and count against this
quota — the common cause of SSE-KMS 5xx throttle during high QPS. Use
S3 Batch Operations KMS key aggregation or limit SSE-KMS encryption for
objects retrieved at high QPS.

### Secrets Manager vs Parameter Store

**Secrets Manager**:
- Purpose-built for **secrets** with rotation.
- First-class **rotation** via Lambda rotation functions; managed
  templates for RDS (MySQL/Postgres/Aurora), Redshift, DocumentDB,
  and many others.
- Cross-account and cross-region replication of secrets.
- JSON-structured secret value.
- Pay per secret-month + per 10,000 API calls. More expensive than
  Parameter Store.

**Systems Manager Parameter Store**:
- Hierarchical key/value store for **configuration**: `/prod/db/host`,
  `String`, `StringList`, and `SecureString` (encrypted with KMS).
- No first-class rotation; you can script it.
- Standard parameters (free, 4 KB cap) and Advanced (cost, 8 KB).
- Parameter hierarchies, paths, and IAM can be per-path
  (`/prod/payments/*`).
- Use for app config that is not a secret, or for secrets where you
  build your own rotation.

**Interview rule**: use Secrets Manager for DB credentials and
**anything with rotation**; use Parameter Store for app config and
non-rotated secrets at low cost. Don't gate app config behind Secrets
Manager just for security theater — the per-secret monthly cost adds
up across thousands of feature flags.

### Security services

- **GuardDuty** — managed **threat detection**; analyzes VPC flow logs,
  DNS logs, CloudTrail mgmt + data events, S3 data events, EKS audit logs,
  Runtime Monitoring (agent on EC2/ECS/EKS), and Malware Protection via
  EDR-style scans. Findings like `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`,
  `TorClient`, `PortSweep`, `Impact:PortProbe` → push to Security Hub.
- **Macie** — managed **data classification**; ML + pattern identifiers
  to find PII (and custom identifiers) in your S3 buckets. Computes
  `ManagedDataIdentifier` findings for ~30+ PII categories.
- **Inspector** — managed **vulnerability scanning** of EC2 (agentless /
  SSM-based) and container images in ECR. Findings scored by CVSS and
  pushed to Security Hub.
- **Config** — records **resource state** over time and evaluates against
  **Config Rules** (managed + custom Lambda-backed). E.g.,
  `restricted-common-ports`, `s3-bucket-versioning-enabled`,
  `root-mfa-enabled`. **Automatic remediation** via SSM Automation
  runbooks on rule non-compliance. A compliance and audit tool, not
  security enforcement; SG rules blocks live traffic, Config detects
  drift from policy.
- **CloudTrail** — **API audit log** of AWS API calls in your account:
  - **Management events** (control-plane: `CreateBucket`, `RunInstances`,
    `AssumeRole`) — recorded by default as a trail, 90-day EventView by
    default.
  - **Data events** (data-plane: `GetObject` on S3, `GetItem` on
    DynamoDB) — **off by default**, opt in, higher volume/cost.
  - **Insights events** — unusual manage-event anomaly detection.
  - Use an **Organization trail** in the management account logging to
    a centralized logging bucket for org-wide visibility.
- **Security Hub** — aggregates findings from GuardDuty, Macie,
  Inspector, Config, Firewall Manager, and partner tools into a single
  normalized `AWSSecurityHubFinding` schema; supports standards
  (CIS AWS Foundations, PCI-DSS, NIST 800-53, AWS Foundational Security
  Best Practices).
- **WAF & Shield & Firewall Manager**:
  - **WAF** — L7 firewall with managed rule groups (AWS Managed Rules,
    OWASP Top 10) attached to CloudFront, ALB, API Gateway, AppSync,
    Cognito. Web ACLs ~ Web ACL × Rule Group × Rule concept.
  - **Shield Standard** — free automatic Layer 3/4 DDoS protection for
    all AWS customers at the network edge.
  - **Shield Advanced** — paid DDoS mitigation + DRT (Response Team)
    access, financial protection, integrated with Route 53, CloudFront,
    Global Accelerator, ALB/NLB.
  - **Firewall Manager** — centrally manage WAF rules and SGs across
    accounts in an Organization.
- **ACM (AWS Certificate Manager)** — provisions and renews TLS
  certificates (public or Private CA); integrates with ELB, CloudFront,
  API Gateway. Public certs are free, auto-renewed. Private certs cost
  per issued cert + Private CA hourly. ACM certs cannot be exported;
  must use ACM-managed ELB/CloudFront TLS termination, or a private
  cert with managed renewal on EC2 requires the cert to be imported
  (or use ACM PCA + synchronized install).

### Network security best practices interviewers expect

- Private subnets for app/data tiers; only the ALB is in a public subnet
  (and even that debate — ALB in public, sometimes in private behind
  Gateway VPC endpoint for back-to-back services).
- Security Groups referencing other SGs, not CIDR 0.0.0.0/0; no
  0.0.0.0/0 inbound on ports other than 80/443 (and even then only on
  the ALB). Use **security group referencing** for app→DB, app→app
  traffic.
- NACLs for additional defense-in-depth; default NACL allows all
  traffic.
- VPC Flow Logs to S3/CloudWatch for forensics; query with Athena.
- VPC endpoints (Gateway for S3/DDB; Interface for KMS, SSM, Secrets
  Manager, STS) so private traffic never traverses the public internet.
- Bastion replaced by **SSM Session Manager** (no inbound 22; no keys
  to rotate, IAM auth, full session logging to S3/CloudWatch).

### Encryption in transit & at rest

- **In transit**: TLS 1.2+/1.3 — ACM-cert on ALB/CloudFront/API
  Gateway; ACM auto-renews. Enforce HTTPS-only S3 buckets
  (`aws:SecureTransport` condition deny).
- **At rest**: by-default or opt-in on every storage/data service:
  - S3 — SSE-S3 (default), SSE-KMS, SSE-C, CSE.
  - EBS — KMS at creation, no perf penalty on Nitro.
  - RDS/Aurora — KMS encryption at creation; cannot be reversed; TDE
    for in-DB cell-level on SQLServer/Oracle/Postgres.
  - DynamoDB — `SSESpecification` with KMS, account default KMS or
    CMK. (DynamoDB does not expose customer-managed envelope at item
    level.)
  - SQS/SNS/Kinesis — KMS-managed server-side encryption.
  - EFS — KMS encryption at filesystem creation.
- Rotate keys via KMS rotation (annual automated for symmetric), or
  custom rotation logic for app-level secrets via Secrets Manager Lambda
  rotators.

---

## Monitoring & Operations

### CloudWatch

- **Metrics** — namespaces (`AWS/EC2`, `AWS/Lambda`, your custom), 0 or
  more **dimensions** (e.g., `InstanceId`, `FunctionName`). 1-min
  resolution standard, 1-sec **high-resolution** for custom metrics.
  Many AWS services emit default metrics; dimensions let you filter.
- **Logs** — **log group** (named, with retention) → **log stream**
  (per source). Input via SDK, agent, or direct via Lambda/ECS/EC2.
  **Metric filters** create metrics from log patterns (legacy — prefer
  embedded metric format for dimensional metrics).
  **Logs Insights** — SQL-ish query language across log groups with
  `fields`, `filter`, `stats`, `sort`, `parse` patterns; great for
  finding the top-IP by 5xx count or avg latency by path.
  **Subscription filters** push log streams to Lambda, Kinesis, or
  OpenSearch.
- **Alarms** — metric thresholds (statistic + period + comparison +
  datapoints-to-alarm + evaluation periods); can be on metrics or
  math expressions of metrics. Sends to SNS, Auto Scaling, or
  EventBridge Scheduler actions.
- **Dashboards** — cross-region, cross-account (with cross-account
  sharing).
- **Unified CloudWatch Agent** — single agent for EC2/on-prem to push
  metrics + logs (replaces older scripts; enables StatsD/collectd).

**CloudWatch vs Datadog**: CloudWatch is native, lower setup cost, tight
integration with IAM and AWS services, and free tier generous. Datadog
(and similar: New Relic, Honeycomb, Grafana Mimir/Tempo/Loki) excel at
**multi-cloud** aggregation, richer trace→log→metric correlation, more
ergonomic query languages, and application-owned APM. Many teams run
both: CloudWatch for AWS infra alarms and Datadog for app/synthetic /
custom metrics. Cost is the decider — CloudWatch Logs and Custom
Metrics get expensive at high volume; tag-budget your dashboards.

### X-Ray & distributed tracing

- **AWS X-Ray** — distributed tracing across services (Lambda, ECS, EKS,
  EC2, API Gateway, SQS, Step Functions). **Daemon** (or OTel collector)
  ships spans; **service map** visualizes call graph and response time
  by node; requests and traces up to 30-day retention.
- **Sampling rules** — control the percent of requests traced to bound
  cost; default 5% with one-per-second reservoir; tune for low-traffic
  services.
- **CloudWatch Application Signals** — AWS's current APM story and where
  X-Ray is being folded in: auto-instrumented (or OTel-emitted) services
  get golden-signal dashboards (latency, error rate, request rate), SLO
  tracking with burn-rate alarms, and a service map, correlated with
  traces and logs. **Transaction Search** (2024+) indexes 100% of spans
  into CloudWatch Logs so you can search all transactions instead of a
  5% sample; X-Ray now exposes a native **OTLP endpoint** for traces.
- **OpenTelemetry** — AWS Distro for OpenTelemetry (ADOT) or vanilla OTel
  SDK is the forward path; emit OTel from your app, ship to the X-Ray
  OTLP endpoint or to Datadog/Jaeger/Honeycomb. Prefer OTel for new code —
  the X-Ray SDK is in maintenance mode.

### CloudTrail & Config (recap)

- **CloudTrail** answers "who did what AWS API call, when, from where" —
  an audit log for compliance and forensic investigation. Region-scoped
  by default but trails can be all-region. Org trails centralize.
- **Config** answers "what did/does this resource's configuration look
  like, and is it compliant with a rule" — a state-recording service for
  drift detection and remediation.

Together: CloudTrail is _API call audit_, Config is _resource state and
compliance_. Both ship to S3; query with Athena or Lake Formation.

### EventBridge

The evolution of CloudWatch Events. **Event buses** (default, custom,
partner) carry JSON events; **rules** match by event pattern (e.g.,
`source: aws.ec2`, `detail-type: EC2 Instance State-change
Notification`), and route to targets: Lambda, Step Functions, SQS, SNS,
Kinesis, ECS task, Batch, API Destination (HTTP webhook).
**Schedules**: **EventBridge Scheduler** (newer) — name, schedule
(cron/rate/at), target, flexible window, TZ, state — supersedes the
older scheduled-rules API and supports one-time schedules up to a
year in the future. **EventBridge Pipes** connect a source (SQS,
Kinesis, DynamoDB Streams) to a target with optional filter/enrich
steps — point-to-point without glue Lambdas.

Use EventBridge for **event-driven decoupling**: instead of polling, a
producer posts an `OrderPlaced` event; consumers (inventory, billing,
email) each have their own rule and target. Failures go to a **DLQ** per
rule target. Decoupling reduces coupling and lets you add consumers
without modifying the producer — a classic Serverless interview answer.

### Systems Manager (SSM)

- **Session Manager** — browser/CLI shell to EC2 & on-prem, no inbound
  22/3389, IAM auth, full session logging to S3/CloudWatch, no keys,
  no bastion. Replaces SSH bastions.
- **Run Command** — execute documents (`AWS-RunShellScript`,
  custom) across a fleet; output to S3/CloudWatch; integration with
  the SSM Agent (already on AL2/AL2023 AMIs).
- **Patch Manager** — patch baselines (CRITICAL/IMPORTANT), maintenance
  windows, patch compliance reporting.
- **Parameter Store** — config (see above).
- **Inventory** — software/hardware inventory of managed instances.
- **Document & Automation runbooks** — used for Config remediation and
  operations runbooks (e.g., "restart hung ECS service").
- **Hybrid Activations** — register on-prem VMs as managed instances (no
  VPC needed).

### Cost optimization

AWS bills on three axes: **compute** (hours/seconds), **storage**
(GB-month), **data transfer** (cross-region/AZ, out to internet). The
pillars of cost optimization:

1. **Right-sizing** — use Compute Optimizer / CloudWatch metrics / load
   tests to drop over-provisioned instances. The single biggest lever.
2. **Compute purchase options**:
   - **Savings Plans** (compute and EC2-instance) — commit to $/hour
     spend for 1/3 year; apply across instance family, region,
     Fargate, Lambda. More flexible than Reserved Instances — the
     modern default.
   - **Reserved Instances** — older, instance-family specific; still
     relevant for Steady-state workloads and can be sold on the RI
     Marketplace.
   - **Spot Instances** — up to 90% discount for interruptible
     workloads (batch, CI, stateless workers, EMR). 2-minute warning;
     use Spot Instances with checkpointing or in ASGs with capacity
     rebalance. Avoid for stateful single-leader workloads.
3. **Storage lifecycle**: S3 tiering (Intelligent/IA/Glacier/Deep
   Archive) + lifecycle policies; EBS snapshot lifecycle (Data
   Lifecycle Manager or custom) — drop old snapshots, archive cold
   ones; EBS gp3 instead of gp2 for cost/IOPS ratio.
4. **Graviton** — `c8g`/`m8g`/`r8g`/`t4g` (Graviton4 is current): ~20%
   lower price, often +10-20% perf → 25-40% better price/perf for
   compatible workloads, ~60% less energy. Applies beyond EC2: Lambda
   arm64, Fargate ARM, RDS/Aurora `db.r8g`, ElastiCache, OpenSearch.
   Re-check that your app's native libs / container base images have
   ARM builds.
5. **Stop non-prod** — stagger dev/QA staggering via Instance Scheduler
   (SSM Automation) or EventBridge Scheduler to stop evenings/weekends.
6. **Data transfer**: egress is the silent cost — CloudFront caches at
   edges (egress to CloudFront is free in many cases), VPC endpoints
   avoid cross-AZ NAT charges, Aurora/RDS cross-region reads cost less
   than full multi-region writes.

Tools:

- **AWS Cost Explorer** — visualize spend by service/tag/region; forecast
  next 3 months; Reserved Instance / Savings Plan recommendations.
- **AWS Budgets** — set budget (cost/usage), alerts at 50/80/100%, action
  policies (e.g., apply a tag, notify SNS).
- **Cost & Usage Report (CUR)** — most granular hourly CSV delivered to S3;
  the canonical input for Athena queries and cost-allocation dashboards.
- **Cost Anomaly Detection** — ML on usage patterns to flag spend spikes.
- **Compute Optimizer / Rightsizing recommendations** — derived over
  14-day CloudWatch history.

### KMS concept recap (mini worked example)

Your app stores customer files encrypted in S3 using SSE-KMS with a CMK
per customer. On upload:

1. App calls `KMS:GenerateDataKeyPair` via the encrypt path? No — for
   S3 SSE-KMS, S3 calls KMS on your behalf with a **grant**; you only
   authorize the CMK toward S3 via key policy + IAM.
2. To add per-customer isolation, create one CMK alias
   `alias/customer-{id}`; bucket policy uses
   `aws:kms:Decrypt` and the object is encrypted under that CMK.
3. Revoke a customer = disable their key or update key policy to deny
   their principal; new uploads decrypt-approved for customer but not
   revoked.
4. Rotation: enable annual KMS key rotation — the key material rotates
   but the key ARN stays; no app change needed.

That mirrors how S3, EBS, and RDS use KMS: the service holds a grant and
the data key, your app holds an IAM policy that permits the service to
use the key on your behalf. You almost never call `GenerateDataKey` from
app code — you let the storage service do it.