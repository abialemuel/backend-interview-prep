# AWS Interview Questions

Grouped by difficulty, with model answers focused on trade-offs (the thing
interviewers actually probe). Work through after reading 01–03.

---

## Easy

### Q1: Name the main EC2 instance families and when you'd pick each.

**Answer:** `t` (burstable) for low/spiky workloads like dev boxes and small
web tiers; `m` for balanced general purpose; `c` for compute-bound workloads
such as batch and video transcoding; `r`/`x`/`z` for memory-bound like in-memory
DBs and SAP HANA; `i`/`d` for high disk throughput with direct-attached NVMe
(huge DFS/Kafka); `p`/`g`/`inf`/`trn` for GPU/inference/training. Within each,
the generation matters: `7` is Ice Lake Intel, `7a` AMD, `7g`/`8g` Graviton
ARM — Graviton typically gives ~20% lower price and often +10-20% perf for
interpreted/JIT workloads (Go, Java, Python, Node).

### Q2: How do t-family burstable instances and CPU credits actually work, and when is "Unlimited" mode dangerous?

**Answer:** `t3`/`t4g` earn CPU credits when idle and spend them for bursting
above the baseline; below baseline you get the full core, and sustained load
burns credits and you're eventually throttled to baseline. With **Unlimited**
mode enabled, you keep bursting after credits run out and pay per-vCPU-hour for
the surplus (Surplus Credits Roared Into Debt). Use t-instances for dev, low-
traffic web, microservices with spiky but low average CPU; never for steady
high CPU (cluster on `c`/`m` instead). The danger is a runaway process in
Unlimited mode quietly racking up a large SPI bill — set CloudWatch alarms on
`CPUSurplusCreditBalance`.

### Q3: Instance store vs EBS — when do you use each?

**Answer:** Instance store is physically attached NVMe — fastest I/O and
lowest latency, but data is lost on stop/hibernate/termination and on host
failure. Use for caches, scratch, big-data data dirs the app can rebuild, HPC
scratch. EBS is network-attached block storage, replicated in the AZ, persists
independently of the instance and supports snapshots to S3, resizing, and
encryption. Use for boot volumes, databases, anything durable. Interviewers
reward recognizing that instance store wins on latency and cost per IOPS when
you can tolerate loss; EBS wins on durability and decoupling from host
lifecycle.

### Q4: What is IMDSv2 and why does it matter?

**Answer:** The Instance Metadata Service exposes information about the
instance (IAM role creds, user-data, tags) at `169.254.169.254`. IMDSv1 was
GET-only; vulnerable to SSRF — any code with localhost reachability could
fetch instance creds. IMDSv2 uses a PUT-then-GET session token pattern and you
should always require it (`HttpTokens: required` on the launch template). It's
default on new AMIs/launch templates and the AppSol SIG flags it as mandatory.
SSRF still matters — even with v2, scope your instance role narrowly.

### Q5: Summarize S3 consistency guarantees.

**Answer:** Since December 2020 S3 provides **strong read-after-write**
consistency for PUTs of new objects, overwrite PUTs, and DELETEs — any
subsequent GET returns the latest version. List operations (`ListObjectsV2`,
`ListObjectVersions`) remain eventually consistent — a PUT followed
immediately by a List may not show the new object, and cross-region
replication is always asynchronous. The classic "write then immediately read
back from a different process" no longer needs a retry-on-404 dance.

### Q6: S3 storage classes in one breath, and how a lifecycle policy would tier them.

**Answer:** Standard (hot), Standard-IA (infrequent, multi-AZ durable),
One-Zone-IA (cheaper, single-AZ), Intelligent-Tiering (auto-tiers via
monitoring fee), Glacier Instant Retrieval (archive with ms retrieval),
Glacier Flexible Retrieval (min-hrs), Deep Archive (long-term, hrs retrieval),
and (rare) Reduced Redundancy deprecated. A typical lifecycle: `Standard →
Standard-IA after 30d → Glacier IR after 90d → Glacier Flexible after 180d →
Deep Archive after 1y → expire after 7y for compliance`. Pair incomplete
multipart upload abort (7d) and noncurrent version transitions for
versioned buckets.

### Q7: Compare S3 encryption options and when to pick each.

**Answer:** **SSE-S3** — AWS-managed AES-256, free, transparent, default on
new buckets since 2023 — pick as the baseline unless you need audit/rotation
control. **SSE-KMS** — keys under your control via KMS, CloudTrail per-key
usage, granular access via key policies, customer-managed key rotation;
subject to KMS quotas so high-QPS GET can throttle, costs per KMS call. Pick
when you need per-key audit/revocation or compliance. **SSE-C** — you supply
the key per request; the key never leaves the caller; pick for regulated
scenarios where you operate your own key store (HSM). **CSE** — encrypt
client-side before upload; pick when absolutely no plaintext should ever
touch the AWS account.

### Q8: Why and how do you use S3 multipart upload?

**Answer:** Multipart upload splits large objects into parts (up to 10000),
part size 5 MB-to-5 GB, max object 5 TB. Each part is uploaded independently
and in parallel, failures resume only the failed part instead of the whole
object, and you can upload parts from different hosts. Required for objects
over 5 GB and recommended above 100 MB. Finalize with a CompleteMultipartUpload
call listing part numbers and ETags; abort with AbortMultipartUpload so
incomplete uploads don't accrue storage — guard with a lifecycle abort rule
(N days after initiation).

### Q9: Compare EBS volume types and when to pick each.

**Answer:** **gp3** is the modern default SSD — 3000 baseline IOPS, scalable
to 16000 IOPS / 1000 MB/s, decoupled from size, cheap; pick for almost all
boot and prod workloads. **io2 Block Express** for critical high-IOPS DBs
(Oracle, SAP HANA, large DBs) — up to 256000 IOPS, 99.999% durability,
multi-attach across 16 instances. **io1** is the older high-IOPS variant.
**st1** (throughput HDD) for big sequential like Hadoop/EMR logs; **sc1**
(cold HDD) for rarely accessed sequential. NVMe instance store remains the
fastest for transient scratch. Size matters: gp3 decouples IOPS from size,
whereas legacy gp2 scaled 3 IOPS/GiB capped at 16000 — a 100 GB gp2 gave only
300 IOPS, bad for small DBs.

### Q10: Public vs private subnet in a VPC — what defines the difference?

**Answer:** "Public" or "private" is purely a route-table property — a subnet
is public when its route table has a `0.0.0.0/0 → igw-xxx` route to an
Internet Gateway, allowing inbound from the internet (so instances there need
public IPs or an IGW-routed ALB). A private subnet has no such route; outbound
to the internet is via a NAT Gateway in a public subnet, and instances are
reachable only from within the VPC. Standard pattern: public subnets for the
ALB only, private subnets for app containers/EC2, dedicated DB subnets deep
in private, route tables separate per AZ with each private subnet routing to
its own AZ's NAT Gateway.

### Q11: NAT Gateway vs NAT Instance — give the trade-off.

**Answer:** **NAT Gateway** is AWS-managed, redundant within the AZ, scales up
to 100 Gbps, no patching, billed hourly plus per-GB data processing — the
production default. You deploy one per AZ and use AZ-specific route tables so
failure of a NAT in one AZ doesn't break outbound for another (and to avoid
cross-AZ data charges). **NAT Instance** is a single EC2 you patch and that's
a SPOF — only for labs/very-low-egress dev where the NAT Gateway hourly cost
is the deciding factor. For private subnets that never need outbound internet
(S3/DynamoDB-only), use a Gateway VPC Endpoint instead and skip NAT entirely.

### Q12: Security Group vs NACL — what are the differences?

**Answer:** **Security Groups** are stateful (return traffic allowed
automatically), attached to ENIs (instances), default-deny inbound/allow-all
outbound, rules reference other SGs/CIDRs and can be modified live. They're
your primary L3/L4 firewall. **NACLs** are stateless, attached to subnets,
evaluated in numeric order, and they require explicit allow for return traffic
(including ephemeral ports 1024-65535). NACLs are useful as a deny-list layer
or for additional defense-in-depth but most architectures leave them at the
default (allow all) and rely on SGs. SGs cannot deny — they only allow.

### Q13: VPC peering vs Transit Gateway — when?

**Answer:** VPC peering is a point-to-point 1:1 connection (same/different
account, same/different region), **non-transitive** (A-B and B-C does not
connect A-C), and you can only have so many peerings before the mesh gets
ungovernable. Pick for very few VPCs. **Transit Gateway** is a regional
hub-and-spoke router that connects many VPCs, VPN, and Direct Connect
gateways via attach, centralizes routing, supports inter-region peering, and
scales linearly. Pick for hybrid connectivity or when you have more than a
handful of VPCs to interconnect.

### Q14: VPC Endpoint — Gateway vs Interface. What's the difference?

**Answer:** **Gateway endpoint** is a route-table entry (a `pl-xxx` prefix
list target) into the VPC route table — only for S3 and DynamoDB — free, no
per-hour cost, just keeps traffic off the IGW/NAT for those services. Pick as
the default for S3/DDB from private subnets. **Interface endpoint** is an ENI
in your subnet with a private IP, fronted by AWS PrivateLink, for any support-
ed AWS service API (KMS, SSM, Secrets Manager, STS, SQS, SNS, dozens) plus
your own custom PrivateLink services; charges hourly plus per-GB. Pick when
you need to call an AWS service API from a private subnet without public
internet — note that with no IGW the pl prefix is unreachable, so Interface
endpoints are the option for those services.

### Q15: Route 53 routing policies — briefly.

**Answer:** **Simple** returns all values (no health check). **Weighted** for
canary/A-B/blue-green (per-record weight 0-255). **Latency** picks the
lowest-latency endpoint for a user — multi-region active-active.
**Failover** switches to a secondary on failed primary health check (DR).
**Geolocation** by country/continent (compliance/localization).
**Geoproximity** by location of your resources with optional bias (load shift
between AZs/regions). **Multivalue answer** returns up to 8 healthy records —
poor-man's DNS LB (not a substitute for ELB). **IP-based** (rare) routes by
client subnet.

### Q16: ALB vs NLB — when and how do they differ?

**Answer:** **ALB** is L7 (HTTP/2, gRPC, WebSocket) with path/host/query/
header rules, redirects, OAuth via Cognito, target groups of EC2/IP/Lambda,
sticky sessions, and a DNS name (not static IPs) — default for web/APIs. **NLB**
is L4 (TCP/UDP/TLS), ultra-low latency, preserves source IP, scales massively
and gives **static IPs per AZ** (your own Elastic IPs). Use NLB for non-HTTP
protocols (game servers, MQTT, legacy TCP), latency-sensitive TCP workloads,
or when you must give a partner a fixed IP. NLB can also front an ALB
(NLB→ALB) to give an ALB static IPs. NLB cross-zone is off-by-default and
paid per GB; ALB cross-zone is always on and free.

### Q17: API Gateway — REST API vs HTTP API. How do you choose?

**Answer:** **REST API** is the older feature-rich flavour — mapping templates,
request validators, WAF, usage plans, API keys, AWS service integrations,
Lambda non-proxy — at higher cost per million requests. **HTTP API** is newer,
cheaper (~70% lower) and faster, with OIDC/JWT + Lambda authorizer, but no
mapping templates, no usage plans, simpler stage-level throttling — best for
new HTTP-backed serverless APIs unless you need the specific features of REST
(JWT aside, e.g., request transformation). Use **WebSocket API** for
long-lived bidirectional. Default new serverless HTTP backends to HTTP API and
escalate to REST only when needed.

### Q18: IAM role vs user — explain the recommendation.

**Answer:** A user is a long-term identity with credentials (password and/or
access keys); a role is an identity assumed by a trusted principal for a
limited time, credentials delivered via STS (15 min-12h, default 1h). Best
practice: humans go through **IAM Identity Center (SSO)**, not individual IAM
users; AWS service-to-service (EC2 instance profile, Lambda execution role,
ECS task role) and cross-account access use roles. This eliminates long-lived
access keys that can leak from CI/laptops and lets you scope per workload.
Rotate by shortening role duration and using IAM Access Analyzer to find
unused access keys to delete.

### Q19: KMS envelope encryption — describe it.

**Answer:** Envelope encryption: you call KMS `GenerateDataKey` to receive a
plaintext data key and a copy of it wrapped by your Customer Master Key. You
encrypt the payload locally with the plaintext key, discard it, and store the
ciphertext + the wrapped data key. To decrypt, you ask KMS to decrypt the
wrapped key (after IAM/key policy allow it) and re-derive the plaintext key
locally. The CMK **never leaves KMS** — only the small data key is exposed
briefly. Benefits: per-object data keys mean compromise of one doesn't
compromise others; CloudTrail logs each KMS Decrypt; the cost is dominated by
data-key size, not payload size, so encrypting TBs of S3 data costs only a
few KMS API calls.

### Q20: Secrets Manager vs Parameter Store — when to choose what?

**Answer:** Secrets Manager is purpose-built for secrets with first-class
**rotation** (Lambda rotators; managed templates for RDS/Aurora/Redshift/
DocumentDB), cross-account and cross-region replication, JSON-structured
secret value — billed per secret-month plus per API call, more expensive.
Parameter Store is a hierarchical configuration store
(`/prod/payments/db/endpoint`); `String`, `StringList`, **`SecureString`**
(encrypted via KMS); standard tier free (4 KB cap); no first-class rotation.
Pick Secrets Manager for DB creds and anything with rotation, Parameter Store
for app config, feature flags, and non-rotated secrets where cost matters.

### Q21: CloudTrail vs Config — what's the difference?

**Answer:** CloudTrail is an **API audit log** of AWS API calls (who did what
when, from where) — answers compliance and forensic questions. Config is a
**resource state** recorder that captures the configuration of resources over
time and evaluates against Config Rules (managed or Lambda custom), with
**automatic remediation** via SSM Automation — answers "is this compliant"
and "how did this resource's config change over time." They're complementary:
CloudTrail tells you who changed the SG, Config tells you the SG was changed
and is now non-compliant and can auto-remediate.

### Q22: CloudWatch Logs Insights — what is it and when would you use it?

**Answer:** CloudWatch Logs Insights is a SQL-like query language over log
groups. You write `fields @timestamp, @message | filter status >= 500 | sort
@timestamp desc | limit 20` or aggregates `stats count(*) by requestId` and
patterns via `parse`. Use it to find the top-IP by 5xx count, average latency
by path, the slowest traces, or to dig into a specific service. Tip: filter
the time window and log group before running — Insights scans every ingested
log event under the query and the cost adds up. For dimensional metrics
prefer the embedded metric format which avoids a separate query.

### Q23: EventBridge — what problem does it solve?

**Answer:** EventBridge is a serverless event bus that decouples producers
from consumers. Producers post JSON events to a bus; rules match by source /
detail-type / content; targets (Lambda, Step Functions, SQS, SNS, Kinesis,
API Destination, Batch, ECS) receive them, optionally with DLQs per target.
It enables **event-driven architectures**: adding a new consumer (e.g., an
audit service that records `OrderPlaced` events) requires no change to the
producer infrastructure. It also subsumes CloudWatch Events and offers
**EventBridge Scheduler** for cron/rate schedules with one-time future
schedules. Failures go to a per-target DLQ so producers are unaffected by
consumer outages.

### Q24: X-Ray distributed tracing — describe.

**Answer:** X-Ray collects spans across services (Lambda, ECS, EKS, EC2, API
Gateway, SQS, Step Functions) via the X-Ray SDK or ADOT/OTel collector. A
**service map** visualizes the call graph and response-time distribution by
node, you can dig into a slow trace to find the offending sub-call, retention
is 30 days. **Sampling rules** bound cost — default 5% with a one-per-second
reservoir; tune for low-traffic services. CloudWatch **ServiceLens** combines
the X-Ray service map with Logs Insights so you can click from a slow node
into the logs of that trace. Forward path is OpenTelemetry (ADOT) so you can
emit OTel from your app and ship to a vendor (Datadog, Honeycomb) too.

---

## Medium

### Q25: EC2 vs ECS vs EKS vs Lambda — give the decision framework.

**Answer:** Pick the **lowest-abstraction** option that meets your non-
functional requirements. **EC2** for max control (custom kernel, host
networking, daemonsets-as-host-processes, lib that Fargate forbids) — most
ops. **ECS with Fargate** for container simplicity without infra mgmt, pay-
per-task — the canonical choice for AWS-native services. **EKS** when you
need k8s portability, Helm/operators/CRDs, multi-cloud strategy, or you're
already standardized on k8s — you take on per-cluster control-plane cost,
CNI IP exhaustion debugging, IAM Roles for Service Accounts setup. **Lambda**
for event-driven workloads with sparse traffic, pay-per-invocation, < 15 min
runs, no infra mgmt — avoid for steady high throughput (containers beat per-
ms billing), long-running jobs, or workloads needing OS control. Escalate
abstraction only when you can justify the additional ops burden.

### Q26: Lambda cold starts — what causes them and how do you mitigate?

**Answer:** A cold start is the latency to initialize a fresh execution
environment for your function — pulling the container image, loading the
runtime, your code, and dependencies, and running the global init code. It's
worst on first invocation after deploy, in scaled-out concurrency, and
particularly heavy for Java/C# (long classloading). Mitigations, in order of
impact: shrink the deployment package (remove heavy deps, use layers for
shared code), preload SDKs and DB clients in global scope but keep init
small, **Provisioned Concurrency** (keep N environments warm and pre-init,
paid hourly regardless of invocations), **SnapStart** for Java (restore from
a snapshot of the post-init JVM — eliminates most Java cold-start latency),
or use lighter runtimes. Care: Provisioned Concurrency nullifies the "pay
only when invoked" cost model — measure first.

### Q27: RDS Multi-AZ vs Read Replica — and why people confuse them.

**Answer:** **Multi-AZ** is **synchronous** replication to a standby in a
*different AZ*, used only for **HA/DR** (failover in ~60 s when the primary
fails). The standby is **not** used for reads. You pay ~2× for compute +
storage but get availability. **Read Replicas** are **asynchronous** and can
be in same or different AZ/region, used for **read scaling and offloading**
(analytics, reporting) and as a promote-in-DR target. People confuse them
because both involve a second instance — the confusion costs when you point
heavy analytic reads at a Multi-AZ standby (which won't accept them) or treat
a read replica as HA (it lags and you have to manually promote). For both,
**Aurora** does it differently: storage is shared across AZs so a replica is
definitely the model.

### Q28: Aurora failover — why is it faster than vanilla RDS Multi-AZ?

**Answer:** Aurora storage is a **distributed, 6-copy / 3-AZ** layer that all
instances (writer and replicas) read/write through. On writer failure, a
read replica is **promoted in-place** — the storage layer doesn't move — and
the cluster endpoint flips DNS to the new writer, typically < 60 s, often in
the 10-30 s range. RDS Multi-AZ, by contrast, blocks writes while a
**different standby instance** is promoted, the volume is reattached, and DNS
flips — closer to 60-120 s. RDS Proxy further abstracts this for the client
side by holding connections and rerouting them across the failover, so
client app impact is mostly reduced to a few second retry window.

### Q29: Aurora Serverless v2 — what is it good for, and when is it the wrong choice?

**Answer:** Aurora Serverless v2 scales compute between 0.5 and 128 ACU
(each ACU ≈ 2 GB RAM / 2 vCPU share) up and down in **milliseconds** based on
load, and you can mix Serverless v2 with provisioned instances in the same
cluster. It's great for variable / multi-tenant workloads where load swings
dramatically (dev/test that runs hot during work hours and idle otherwise,
or a SaaS with hour-of-day peaks). It's the **wrong** choice for steady
high-throughput — at saturation you're paying a premium per-ACU, comparable
provisioned instances are cheaper. Also note: ACU changes don't help the
**write** path very much because storage is separate; reads scale well, but
the cluster writer still has to drive the storage commit.

### Q30: Why use RDS Proxy?

**Answer:** RDS Proxy is a managed connection pooler between Lambda/EC2/ECS
and your RDS/Aurora database. It pools DB connections, so brief-lived Lambda
invocations don't each open/close a connection (which would exhaust the DB's
`max_connections`). It's **failover-aware** — on Aurora failover it reroutes
pooled client connections to the new writer, so app impact is a brief retry
window, not a minute of broken connections. It also supports IAM auth and
Secrets Manager rotation transparently. The classic interview pattern:
"Lambda → RDS via RDS Proxy, with Secrets Manager rotation, in a VPC with
interface endpoints for KMS and Secrets Manager — that whole stack gives you
encrypted, rotated creds, no connection exhaust, and resilient failover."

### Q31: DynamoDB partition key and sort key — what design problems do they solve?

**Answer:** The **partition key (PK)** hashes items onto a physical
partition; pick a high-cardinality, evenly-accessed attribute to avoid hot
partitions. The optional **sort key (SK)** enables a one-to-many relationship
under one PK and supports range queries (`begins_with`, `between`, `>`,
`<`). This unlocks querying all items for one user (`USER#123`) ordered by
timestamp (`EVENT#2024-07-09T...`). Item size is capped at 400 KB; for
larger payloads store the blob in S3 and keep only the key. If you need to
query the same data by another dimension, use a GSI with a different PK/SK.
A well-modeled DynamoDB table can serve 5+ access patterns without joins —
but the access patterns must be known in advance.

### Q32: LSI vs GSI in DynamoDB — differences.

**Answer:** A **Local Secondary Index (LSI)** uses the same partition key as
the base table but an **alternative sort key**; defined **only at table
creation**, up to 5 per table; shares provisioned throughput with the table;
10 GB per-PK size limit; only eventually consistent reads. A **Global
Secondary Index (GSI)** uses an alternative **partition key and sort key**,
can be created/modified any time, has its own throughput and scaling, recommended
for most add-on access patterns. GSI writes throttle when the GSI's own
provisioned capacity is exhausted — set GSI throughput above table write
rate or use on-demand. Use **sparse indexes** (only items containing the
indexed attribute appear) when you need a "find all `status=PENDING`"
pattern without scanning.

### Q33: DynamoDB capacity — provisioned vs on-demand.

**Answer:** **Provisioned** charges per RCU/WCU per second, autoscaling on
consumed vs provisioned via target tracking (1 RCU = 1 strongly consistent
4 KB read/sec or 2 eventual reads/sec; 1 WCU = 1 write/sec of 1 KB). Cheaper
for predictable load; if you mis-size you get throttled. **On-demand** charges
per request (`ReadRequestUnits` / `WriteRequestUnits`), 2-3× more expensive
at full utilization but no capacity planning — perfect for unknown or bursty
workloads. There's a third lever: **adaptive capacity** automatically
reallocates throughput between partitions of a busy table so you can poke a
hot partition without reassigning — useful in the new provisioned engine.

### Q34: DynamoDB hot partitions — what are they and how do you mitigate?

**Answer:** A hot partition happens when a disproportionate share of traffic
hits one partition key (e.g., a "leaderboard top" or a counter that all
clients increment). Historically a partition was capped at 1000 WCU / 3000
RCU and you'd get throttled. Mitigations: (1) **shard the hot key** by
appending a random suffix (`USER#123#0..9`) and reading across all shards,
(2) redesign the access pattern (e.g., use per-period counters instead of a
global counter), (3) **adaptive capacity** in newer DynamoDB automatically
splits hot partitions if you provision enough — but you must still provision
to the total rate you want, (4) put a **DAX** cache in front for read-heavy
hot keys. The right answer depends on whether you control read or write
patterns; never solve hot-partition with Scan.

### Q35: DAX — when is it worth it?

**Answer:** DAX is an in-memory caching layer in front of DynamoDB that gives
single-digit-ms (often sub-ms) reads and write-through caching handled
internally. It's worth it when you have **read-heavy hot keys** (e.g., user
profiles, leaderboard, catalog) at high QPS where adding RCU replicas becomes
expensive; DAX beats provisioning 5× more RCU on read-heavy tables.
Caveats: DAX is **eventually consistent only**, requires the DAX SDK client
(per-language), needs to live in your VPC, and the cluster itself is a
managed component with its own failure modes; for very low QPS the
simplicity of "just bump provisioned RCU" wins.

### Q36: When would you choose DynamoDB over RDS (or vice versa)?

**Answer:** Choose DynamoDB when access patterns are well-known and limited in
number, you can denormalize, latency must be single-digit ms at any scale,
and you want serverless ops at low cost — user profiles, sessions, shopping
carts, event logs keyed by user, simple leaderboards. Choose RDS (relational)
when the data is genuinely relational — many entities with foreign-key
relationships, ad-hoc reporting and joins, transactions across multiple
entities, complex multi-row updates and SQL analytics. A good rule: model
with Dynamo only when you can enumerate **all** access patterns in advance;
otherwise start with RDS/Aurora Postgres. Hybrid (RDS for transactional OLTP
+ DynamoDB for hot-key lookups + Redshift/OpenSearch for analytics) is
common in real systems.

### Q37: STS AssumeRole with External ID — what problem does it solve?

**Answer:** The **confused-deputy** problem: a role in your account that
trusts another account's principal could be invoked by an attacker who
controls a different principal in that trusted account — e.g., a SaaS vendor
accountant who holds AssumeRole rights for any role "any customer asked the
vendor to assume". By requiring an **ExternalId** (a string the customer
chooses and includes in the AssumeRole call, checked via the trust policy's
`Condition`), the vendor must include the customer's specific ID when
assuming the customer's role — preventing one customer's request from
triggering a role assumption for another. The trust policy Condition is
`{ "StringEquals": { "sts:ExternalId": "<customer-id>" } }`.

### Q38: Cost optimization techniques on AWS — list and trade-offs.

**Answer:** (1) **Right-sizing** — drop over-provisioned instances per
Compute Optimizer / CloudWatch; the biggest single lever, no risk if you
load-test. (2) **Savings Plans** — commit $/hour for 1-3y, apply across
instance family, region, Fargate, Lambda; the modern default over RIs, ~30-
72% off. (3) **Spot** for interruptible batch/CI/stateless workers, up to 90%
off; 2-min warning, needs checkpointing/capacity-rebalance. (4) **Lifecycle**
on S3 + EBS snapshots (archive cold, abort incomplete multipart). (5)
**Graviton** for compatible workloads (~20% lower price + perf). (6) Stop
non-prod with EventBridge Scheduler/SSM Instance Scheduler. (7) CloudFront for
egress caching, VPC endpoints to avoid cross-AZ NAT fees, cross-region reads
cheaper than cross-region writes. Trade-offs: Spending commitments increase
finance ops overhead; Spot reduces reliability; cold-tier retrieval adds
latency to restores; Graviton adds build/test matrix complexity.

### Q39: Graviton — what's the strategic case?

**Answer:** AWS Graviton (`c7g`, `m7g`, `r7g`, `t4g`, plus Graviton4 `8g`) is
ARM — typically **~20% lower price** AND often **+10-20% performance** vs
equivalent Intel/AMD, giving roughly **25-40% better price/perf** for
compatible workloads (Go, Rust, Java on JDK 17+, Node, Python). It also has a
better per-core energy profile (Sustainability pillar). Caveats: native deps
need ARM builds (e.g., libpq, image libraries), CI matrix needs an ARM lane,
some vendors still ship x86-only images. The pattern: rebuild CI images on
Graviton, A/B the prod workload against the x86 baseline for a week, and
migrate if perf and cost both improve — most stateless HTTP services do.

### Q40: Multi-AZ vs Multi-Region — the trade-off.

**Answer:** **Multi-AZ** protects you from a single AZ failure (rare, but
high-impact) with **synchronous** replication in-region, low write latency, ~2×
cost — most production databases do this. **Multi-Region** protects against a
whole region failure (very rare, but catastrophic) with **asynchronous**
replication across regions (Aurora Global DB, DynamoDB Global Tables, S3 CRR),
some write-lag, RPO measured in seconds-to-minutes, and significantly higher
cost (replicated storage + compute in each region, plus cross-region data
transfer). Trade-off: more regions = better DR posture but higher cost,
operational complexity (failover playbook, traffic shifting via Route 53 /
Global Accelerator), and data-sovereignty concerns. Most applications need
**Multi-AZ** but NOT Multi-Region; reserve Multi-Region for apps with a
genuine regional availability SLA, regulatory geographic separation
requirements, or active-active latency optimization for global users.

### Q41: Designing for HA and DR — explain RTO and RPO and how AWS services map.

**Answer:** **RPO (Recovery Point Objective)** is how much data you can lose —
limit between event and recovery; **RTO (Recovery Time Objective)** is how
long until service is back. Synchronous replication gives ~0 RPO at the cost
of write latency (RDS Multi-AZ, Aurora in-region). Async replication gives low
write latency but RPO > 0 (Aurora Global DB ≈ seconds; DynamoDB Global Tables
≈ 1s; cross-region S3 RTC ≈ seconds). RTO depends on failover automation:
managed failover (Aurora DNS flip, Route 53 failover, ALB target switching)
gets RTO ~1 min; manual cross-region restore from backup can be hours. A
common pattern layered by tier: Multi-AZ for HA, scheduled snapshot backup for
RPO = hours, cross-region async replication (Aurora Global DB / DynamoDB
Global Tables) for RPO = seconds, and DR runbook + automated promotion for
RTO. Always test failover; lots of "Multi-AZ HA" designs have an unrun
runbook and surprise the team on day 1.

### Q42: Explain the Well-Architected pillars and how they map to interview pattern answers.

**Answer:** Operational Excellence (deployments, runbooks, automation, SSM,
IaC), Security (IAM least privilege, KMS, GuardDuty, CloudTrail, Config,
private subnets, SGs), Reliability (Multi-AZ, Multi-Region where justified,
auto scaling, health checks, retries with jitter, idempotency, DR), Performance
Efficiency (right resource type & size, Graviton, read replicas, DAX, Cloud-
Front, intelligent tiering), Cost (right-sizing, Savings Plans, Spot,
lifecycle, stop non-prod), and Sustainability (maximize utilization, energy-
efficient instance types, minimize data movement). When interviewers ask
"design X", name choices and justify against **two or more** pillars that
were traded off — e.g., "Lambda for cost & ops efficiency, with provisioned
concurrency trading cost for reliability against cold starts, plus a DLQ for
reliability of failed events, encrypted at rest with KMS for security." Structured
pillar-by-pillar answers are scored higher than a stream of features.

---

## Hard

### Q43: You're designing a media pipeline: ingest big files from partners, transcode, store originals, deliver. What services and why?

**Answer:** Partners upload via **S3 presigned URLs** (or Transfer Family +
SFTP for partners who need FTP semantics) into a "raw" bucket with
versioning + Object Lock (Compliance for originals keeping 7y). An S3
event notification to **EventBridge** fans out to multiple consumers:
start a **MediaConvert** job (managed transcoding) and emit an audit event.
MediaConvert writes results into a "processed" bucket with SSE-KMS using a
per-tenant key. Lifecycle rules tier originals to Glacier IR after 90d and
Deep Archive after 730d; expire failures after 30d. Delivery is via
**CloudFront** with an **OAC** to the bucket, signed URLs/cookies for paid
content, CloudFront Functions for light header auth, Lambda@Edge for geo
gating. Monitoring via CloudWatch + X-Ray; errors to a DLQ and EventBridge
→ SNS → Pager. Trade-off: pre-signed URLs vs Transfer Family (cost +
partner UX vs protocol support); S3-tiered storage balances cost
optimization with retrieval latency; CloudFront vs direct S3 access depends
on whether the content is geo-distributed; Object Lock vs lifecycle-only
delete-markers for compliance.