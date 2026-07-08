# Networking & Databases on AWS

This file covers the network path from the user to your service (VPC, Route
53, CloudFront, ELB, API Gateway, Direct Connect/VPN) and the managed
database tier (RDS/Aurora, DynamoDB, ElastiCache; brief on DocumentDB,
Neptune, Redshift, OpenSearch).

---

## Networking

### VPC — Virtual Private Cloud

Your isolated, software-defined network on AWS. Regional resource
spanning multiple AZs. You carve out one or more **CIDR blocks** (e.g.,
`10.0.0.0/16` primary, plus secondary up to 5 IPv4 CIDRs), then subnet it
per AZ.

Core pieces:

- **Subnet** — AZ-scoped CIDR range. **Public** subnet has a route to an
  Internet Gateway (IGW); **private** subnet does not (uses NAT Gateway for
  outbound). "Public/private" is **purely a route-table property**, not an
  attribute on the subnet object.
- **Route table** — per-subnet (or VPC default) set of destination→target
  routes. `0.0.0.0/0 → igw-xxx` makes a subnet public. Private subnets
  typically route `0.0.0.0/0 → nat-xxx`. Local route (`10.0.0.0/16 → local`)
  is automatic.
- **Internet Gateway (IGW)** — one per VPC, horizontally scales, gives
  public IPv4 / IPv6 egress and is the target for inbound from the internet
  to public subnets.
- **NAT Gateway** — AWS-managed, **per-AZ**, redundant within the AZ,
  scalable up to 100 Gbps (more on quota request). Elastic IP attached.
  **Charges**: hourly + per-GB data processing fee. You must put one in
  each AZ where you have private subnets needing outbound, and use
  separate route tables per AZ to route to the local AZ NAT (otherwise a
  NAT failure in one AZ breaks outbound for another AZ — cross-AZ NAT
  traffic also incurs cross-AZ data charges).
- **NAT Instance** — a single EC2 running NAT in your account. **Legacy**,
  single point of failure, no auto-scaling, you patch it. Only use for
  lab/cost-constrained dev where the NAT Gateway hourly cost matters.
- **Security Groups (SG)** — **stateful**, attached to ENIs/instances.
  Default-deny inbound, allow-all outbound (stateful return). Rules are
  *references* to other SGs/peers/CIDRs, can be modified live (no
  reconnect). You can reference SG of a peer ALB/instance — preferred over
  CIDRs.
- **NACLs (Network ACL)** — **stateless**, attached to subnet. Rules
  evaluated in order; must explicitly allow return traffic (e.g., ephemeral
  ports 1024–65535 outbound). Use cases: deny-list specific CIDRs, additional
  defense in depth. Most production architectures leave NACLs at default
  (allow all) and rely on SGs.
- **VPC peering** — point-to-point, 1:1 connection between two VPCs
  (same/different account, same/different region). **Non-transitive**: VPC A
  peered with B and B peered with C does NOT mean A can reach C. Doesn't
  support edge-to-edge routing. Scales quadratically as you add VPCs.
- **Transit Gateway (TGW)** — regional hub-and-spoke (or mesh) router.
  Connect VPCs, VPN, Direct Connect gateways. Simplifies N×N peering, central
  routing, supports inter-region peering, can attach to Network Manager for
  topology. Standard choice for > few VPCs or hybrid connectivity.
- **VPC Endpoints** — keep traffic between your VPC and AWS services
  **inside the AWS network** (no public IP, no IGW):
  - **Gateway endpoint** — free, only S3 and DynamoDB; appears as a route
    in your route table (`pl-xxx` prefix list). Recommended over IGW for
    private-subnet S3/DDB access because it's cheap/free and private.
  - **Interface endpoint (PrivateLink)** — ENI in your subnet with a
    private IP for the service API (most AWS services, including SSM,
    Secrets Manager, KMS, SQS, SNS, STS, and your own PrivateLink
    services). Hourly + data charges. Supports custom DNS (private hosted
    zone) to override public service DNS to the private IP.
- **PrivateLink** — expose a service in your VPC to other VPCs/accounts
  over the AWS backbone as if it were an AWS service, without peering,
  without allowing VPC-to-VPC routing. Customers consume via interface
  endpoint. The standard way to offer a SaaS API on AWS privately.
- **DHCP options set** — per-VPC; control DNS servers, domain name, NTP.
  Most use defaults (Route 53 Resolver, AmazonProvidedDNS).
- **DNS via Route 53 Resolver** — VPC-resolver, recursive DNS automatically
  per VPC. **Inbound/outbound endpoints** let on-prem resolve your VPC
  private DNS and vice-versa via the Resolver — useful in hybrid models.

Design pattern interviewers expect: **two AZ public subnets** (ALB only),
**two AZ private subnets** (app containers/EC2), **two AZ DB subnets**
(RDS). ALB in public, fleet in private, DB deep in private with SG rules
from app SG only.

### Route 53 — DNS

Authoritative DNS, 100% availability SLA (via global anycast). Resources:
**hosted zones** (collection of records for a domain), **records** with
**routing policies**, **health checks**.

**Record types**: A/AAAA, CNAME, MX, TXT, NS, SOA, CAA, PTR. **Alias
records** are Route-53-specific extensions of A/AAAA that point to AWS
resources (ELB, CloudFront, S3 website, API Gateway) — **free** to query
against AWS records, support top-of-zone apex (`example.com`), and follow
the underlying target's IP changes automatically. Use Alias over CNAME for
AWS resources wherever possible ({{ CNAME cannot be at the zone apex }}).

**Routing policies** — the question interviewers love:

- **Simple** — one record, one (or a few) value(s); queries return all
  values in random order. No health checks.
- **Weighted** — multiple records, each with weight 0–255; traffic
  split proportional to weights. Use for canaries, blue/green, A/B.
- **Latency** — pick region/endpoint with lowest latency to the user from
  AWS's view. Use for multi-region active-active with latency optimization.
- **Failover** — primary + secondary with health checks. If primary
  unhealthy, switch to secondary. Classic DR mode.
- **Geolocation** — return values by user **country/continent** (where to
  route users fromcontinent X). Use for compliance (EU users → EU region)
  or localization.
- **Geoproximity** — bias traffic based on the **geographic location** of resources, with
  optional bias. Use to shift load between two AZs/regions.
- **Multivalue answer** — return up to 8 healthy records per query; client
  picks. Not as good as real LB but a cheap DNS-level LB/HA mechanism when
  you can't deploy an ELB.
- **IP-based** (rare) — by client subnet / EDNS0 Client Subnet IP.

**Health checks**: external (HTTP/TCP/HTTPS) and **CloudWatch-alarm-based**
checks. Records can require a passing health check before being eligible.
Combine failover+health-check+global accelerator or CloudFront for DR.

**Hosted zones**: public (resolvable from anywhere) and **private hosted
zone** (resolvable inside associated VPCs only) — use private zones for
internal service DNS (`api.internal.example.com` → ALB private IP).

### CloudFront — CDN

Global CDN of edge locations (400+, including Regional Edge Caches that
hold objects longer to backfill smaller edges). You create a **distribution**
with **origins** (S3 bucket, ALB, media store, any HTTP origin) and
**behaviors** (path patterns → origin, allowed methods, caching policy,
lambda/function associations).

Behaviors worth knowing:

- **Signed URLs / signed cookies** — time-limited access to private
  content; signed URL for one file or signed cookie for many files (set
  cookie once per session; works for whole CDN distribution).
- **Lambda@Edge** — Lambda functions running at edge POPs (viewer
  request/response, origin request/response); Node/Python, longer cold
  start tolerance, billed per invocation. Use for URL rewrites, A/B, geo,
  auth.
- **CloudFront Functions** — lighter JS-only runtime in edge, ms-scale
  init, cheaper, smaller footprint — preferred for header manipulation
  and simple viewer-request logic that fits in 2 MB deploy.
- **Origin Access Control (OAC)** — **replaces** the deprecated Origin
  Access Identity (OAI). OAC supports SSE-KMS and all S3 signings; bucket
  policy grants `s3:GetObject` to `Service: cloudfront` only when
  `AWS:SourceArn` = distribution ARN.
- **Field-level encryption** (legacy) — TLS to keep PII encrypted to a key
  specific origin in the same request; mostly replaced by Lambda@Edge.
- Default cache behavior reflection: cache by URL path + selected query
  strings + cookies (Whitelist/Include All/All-Except). Mis-configuring
  caching is the #1 cause of cache leaks across users — review session
  cookies carefully.
- **Field-level encryption** is legacy; prefer CloudFront Functions with
  response headers or Lambda@Edge.
- **Response headers policies** for HSTS, CORS, security headers — managed
  policies exist.
- **Real-time metrics and logs** — logs to Kinesis Firehose for near-real-time.
- **Origin Shield** — a regional edge cache tier in front of your origin,
  consolidating origin fetches and improving cache-hit/expelling cold
  starts; you pick a Region.

### ELB — Elastic Load Balancing

Three flavours + GWLB.

- **ALB (Application Load Balancer)** — **L7 (HTTP/HTTPS, gRPC, WebSocket,
  HTTP/2)**, **layer 7 features**: path-based and host-based routing,
  query-string/header-based rules, redirects, fixed responses, OAuth via
  Cognito, Lambda target type), ALB built-in SG protection
  (Instance SG needs `--vpc-...`). One **ALB per AZ assigned IP
  addresses** (or one per AZ), and a **DNS name** that fans out across AZs
  — the canonical ALB entry point is the DNS name, **not** static IPs.
  **Target groups** of EC2/IP/Lambda/instance; supports sticky sessions
  (duration-based or load-balancer-generated cookie), slow start for
  warming up new targets, deregistration delay (the "connection draining"
  knob — time LB waits for in-flight requests to drain before deregistered
  target is fully removed, default 300s).
- **NLB (Network Load Balancer)** — **L4 (TCP/UDP/TLS, TLS-passthrough)**,
  **static IPs per AZ** (one Elastic IP per AZ or one private IP per AZ),
  **ultra-low latency**, preserves source IP (e.g., visible to targets
  without PROXY protocol), enormous throughput, supports static IPs and
  cross-zone is optional/charged. Target groups can route to IP, instance,
  ALB (NLB fronting an ALB to give it static IPs).
- **GWLB** — for third-party security appliances; transparently inserts
  appliances in the traffic path with GENEVE tunnels. You don't load
  balance user traffic with it; it's for security vendors.
- **Classic ELB** — legacy; do not use for new designs.

**Cross-zone load balancing**: ALB always on (no charge); NLB **off by
default**, paid per-GB cross-AZ charge if enabled. If you want even
distribution across AZs regardless of target counts, enable.

**Connection draining / deregistration delay**: when a target deregisters
or fails health checks, LB stops sending new requests but waits
`deregistration_delay` (default 300 s) for in-flight responses. Set
lower for short-request APIs (e.g., 30 s) and higher for long-poll/upload.

### API Gateway

Fully managed API front door; supports three protocols:

- **REST API** — original; richest features (mapping templates, request
  validators, WAF, usage plans, API keys), $ per million requests + data
  transfer.
- **HTTP API** — newer, cheaper (~70% lower than REST), faster, simpler.
  No mapping templates, limited features, OIDC/JWT + Lambda
  authorizer. **Default choice** for simple HTTP backends unless you need
  REST-specific features.
- **WebSocket API** — long-lived bidirectional; route by `$connect`,
  `$disconnect`, `$default`, custom routes.

Integrations: Lambda (proxy or non-proxy with mapping template),
EC2/ECS via VPC link + NLB, HTTP (any URL), mock, AWS service (e.g., SQS
via `AWS_PROXY`).

**Throttling**: account-level **10,000 RPS** default, per-stage/per-key
limits via **usage plans + API keys** (REST) or stage-level throttling
(HTTP). Returns `429`. Protect downsampled Lambda with reserved
concurrency or use SQS buffering.

**Caching**: per-stage cache (0.5–237 GB), TTL'd; keyable by method+path
(selected headers/query strings). Avoid caching with user-specific
identifiers unless you partition keys carefully.

**Auth**: IAM sigv4 (rare for browser), **Cognito user pool** (oidc/JWT
for REST/HTTP), **Lambda authorizer** (Bearer token, sync call), custom
authorizer can return context for downpipe permissions.

**Stages, deployments, usage plans**: deploy to a stage (`prod`, `v2`),
enable canary at stage level, throttling/key tiering via usage plans +
API keys mapped to stages.

### Direct Connect & VPN (brief)

- **Direct Connect (DX)** — dedicated 1/10/100 Gbps port at a DX location
  routing to your VPC over AWS backbone, **predictable latency**, separate
  from the public internet, supports BGP, **MACsec** for encryption on
  newer ports. Pair with redundancy (two paths, two routers) and a VPN
  backup for HA. Use for hybrid workloads with strict bandwidth/latency or
  compliance.
- **Site-to-site VPN** — IPsec tunnels, two for HA, over the public
  internet, transit gateway recommended, faster to provision than DX.
- **Client VPN** — OpenVPN-based, mutual or directory auth, per-user
  sessions, good for remote corporate access without SSH bastions.

---

## Databases

### RDS — managed relational

Managed MySQL, PostgreSQL, MariaDB, SQLServer, Oracle, plus **Aurora**.

**High availability vs scale**:

- **Multi-AZ deployment (HA)** — synchronous standby in a **different AZ**;
  primary commits block until standby durable. Disrupts writes for ~60 s in
  failover. Costs ~2× storage/compute. Used for HA / DR — _**not** for read
  scaling_.
- **Read replica** — **asynchronous**, can be in same AZ / cross-AZ /
  cross-region, used for **read scaling and offloading analytics**. Can be
  promoted in DR. Cross-region replication adds latency for writes.

**Backups**:

- **Automated daily backup** during a backup window, retained 1–35 days;
  point-in-time recovery (down to second in last 5 minutes) enabled
  automatically when retention > 0. Stored in S3. Storage I/O can be
  suspended during snapshot.
- **Manual snapshots** are kept until you delete them.

**Encryption** — at rest via KMS, can't be enabled after creation (copy
snapshot + restore to enable). Aurora encryption-then-replicate.

#### Aurora specifics

Aurora is AWS-engineered Postgres/MySQL-compatible with a **distributed
storage layer**: 6 copies across 3 AZs (4/6 quorum writes — only 4 must ack
— fast), storage auto-scaling in 10 GB chunks, **up to 15 read replicas**
(MySQL/Postgres) Lag minimized; **cluster endpoint** (writer), **reader
endpoint** (load-balanced across replicas), **instance endpoint** (per
instance) — drivers can decide on writer vs reader. Failover typically
**60 seconds** or less (vs RDS Multi-AZ's 60+).

- **Aurora Serverless v2** — scales compute 0.5–128 ACU (1 ACU ≈ 2 GB RAM/2
  vCPU share) up and down in milliseconds based on load. Mix with provisioned
  instances in same cluster. Good for variable/multi-tenant workloads whose
  load shifts dramatically. Not recommended for steady high-throughput —
  provisioned is cheaper per-ACU.
- **Aurora Global Database** — single-region primary, replicate to up to 5
  read-only secondaries in other regions; storage-level replication
  (typically < 1 s lag), dedicated network channels. **Managed planned
  failover** and **unplanned detach-and-promote** for DR. Cross-region
  Recovery Point Objective (RPO) seconds, Recovery Time Objective (RTO)
  ~1 minute (RTO bounded by DNS update + driver reconnect).
- **Fast failover** features — fast cluster DNS, RDS Proxy helps too.

**RDS Proxy** — managed connection pooler in front of the DB; reduces open
DB connections (Lambda pooling!, multi-tenant SaaS), failover-aware
(reroutes connections on Aurora failover with app-side breakpoint only
around 5–30 s instead of minutes), IAM auth, secrets via Secrets Manager.
Strongly recommended for Lambda→RDS workloads to avoid connection
exhaustion. Reduced confl. — pay per vCPU-hour.

### DynamoDB — key-value & document NoSQL

Single-digit millisecond latency at any scale; fully managed; throughput or
on-demand pricing. Serverless. Schemaless except **key schema**.

**Key model:**

- **Partition key (PK)** only → hash partitioned. Pick a high-cardinality,
  uniform-access key.
- **PK + sort key (SK)** → items grouped under one PK, sortable/compared by
  SK. Enables **one-to-many** naturally: PK = `USER#123`, SK = `ORDER#2024-01-04#456`.

Item size limit **400 KB**. Table items live on a **partition** selected
by hashing the PK. A partition holds all items of one PK value (so a
single PK with many SK items can hot-partition if access to one PK is
unbalanced).

**Capacity modes:**

- **Provisioned (Rcu/Wcu)** — pay per provisioned unit/sec; autoscaling TPS
  proactive/target tracking on consumed. 1 RCU = 1 strongly consistent
  read/sec (or 2 eventual reads/sec) of 4 KB; 1 WCU = 1 write/sec of 1 KB.
  Cheaper for predictable load. **Adaptive capacity** auto-rebalances
  between partitions when one needs more than its fair share of throughput.
- **On-demand** — pay per request (`ReadRequestUnits`, `WriteRequestUnits`);
  2–3× more expensive than provisioned at full use; perfect for unknown /
  spiky workloads and to avoid capacity planning.

**Indexes:**

- **Local Secondary Index (LSI)** — alternative SK under **same PK**,
  defined **at table creation only**, up to 5 per table. Eventual
  consistency. Shares provisioned throughput with table. 10 GB per-PK
  size limit.
- **Global Secondary Index (GSI)** — alternative PK _and_ SK; can be
  created/modified any time, has its own throughput, eventual consistency,
  recommended for most access-pattern addition. Costs are on the index;
  hot GSI can throttle writes (set GSI WCU above table write rate).
- **Sparse indexes** — only items containing the index key attribute are
  in the index — a powerful pattern for filtering (e.g., index only
  `status=PENDING`).

**Reads**: strongly consistent (`ConsistentRead=true`, 2x RCU) vs default
eventually consistent. **Transactions**: `TransactWriteItems` (up to 100
items, consume 2 WCU each) and `TransactGetItems` — ACID across items /
tables in a single region.

**Streams** — ordered log of item changes (24-hour retention) at 1 shard
per partition; consumed by Lambda or Kinesis Adapter or DynamoDB Streams
Kinesis. Power **global tables** (multi-region active-active) which use it
internally, and event-driven fanout (trigger Lambda on change).

**DAX** — in-memory caching layer fronting DynamoDB; ms (microsecond at
scale) reads, write-through caching, handles cache invalidation internally.
4–10× cheaper read infra than adding replicas at high QPS. **Caveats**:
eventually consistent only, requires DAX SDK client; not all apps are worth
it (small QPS workloads should just provisioned Rcu).

**Global tables** — multi-region **active-active** replication via Streams;
pick regions, AWS handles replication. Read-after-write in a different
region has typical seconds lag.

**Best practices interviewers expect**:

- One-to-many via PK + SK (do not make N tables).
- Sparse GSI for alternative access patterns (`status=PAID`).
- **Avoid Scan** on hot tables — model it differently or use S3 export +
  Athena for analytics.
- Fanout via GSI; if you need multiple access patterns, multiple GSI are
  cheaper than scanning if GSI selectivity is high.
- Don't model relationships that need joins — denormalize or use adjacency
  lists.
- 400 KB item cap means put large blobs/payloads in S3 and store only keys.
- **Hot partition** — historically one partition was limited to 1000 WCU /
  3000 RCU; newer DynamoDB (since ~2018) splits hot partitions dynamically
  if you provision enough throughput (adaptive capacity). Calm the access
  by spreading the hot PK across many PKs (e.g., suffix-shard
  `USER#123#0..N`) or pin hash ranges.
- Use **conditional writes** (`ConditionExpression`) for idempotency and
  optimistic concurrency — return `ConditionalCheckFailedException` to
  signal to the client.

When Dynamo vs RDS (interview answer): if relationships and ad-hoc joins
matter, go relational; if the access pattern is **well-known in advance**
and you can denormalize, DynamoDB gives you ops-free scale at low cost.
DynamoDB shines for user profiles, session stores, shopping carts, event
logs by user, leaderboard (with SK and reverse-ordered scans). RDS shines
for order-management OLTP with many relations, reporting, complex SQL.

### ElastiCache — managed Redis / Memcached

Use for:

- **Cache** — in front of RDS/DynamoDB to absorb read load and protect
  against hot keys.
- **Session store** — externalized session for stateless app tier with TTL.
- **Leaderboards/counters/locks** — Redis `ZSET`, `INCR`, `SETNX`.
- **Rate limiting** — sliding window with sorted sets.

**Redis (cluster mode)** — sharding across shards, each shard has 1
primary + N replicas, multi-AZ, automatic failover if enabled (Multi-AZ
with automatic failover). **Cluster mode disabled** — single shard with
primary + replicas, similar to a standalone redis that happens to be
replicated. Use cluster mode for > single-shard throughput or when you
need sharding.

**Memcached** — multi-node, no persistence, no replication, no failover,
no pubsub. **Pick Redis almost always** — multi-AZ HA, persistence, TLS,
Geo/spatial, richer types; Memcached mainly for a couple niche cases
(threaded caching with very high get rate and no persistence reqs).

Encryption: at-rest/in-transit on Redis clusters; must be enabled at
creation (cluster mode on).

### DocumentDB, Neptune, Redshift, OpenSearch (brief)

- **DocumentDB** — managed MongoDB-compatible (PostgreSQL-engine tunes for
  JSON). Aurora-like storage, up to 15 read replicas, in-place restore from
  snapshot. Use as managed Mongo-alternative for content, catalogs.
- **Neptune** — managed graph DB (Property Graph + RDF/SPARQL). Use for
  social/fraud/knowledge graphs.
- **Redshift** — managed **columnar MPP** data warehouse. Petabyte scale,
  massively parallel; supports COPY from S3, federated queries against RDS,
  RA3 nodes with managed storage, serverless option (Redshift Serverless).
  For OLAP/analytics — not transactional workload.
- **OpenSearch Service** — managed OpenSearch/Elasticsearch fork; full-text
  search, log analytics (with perf topologies integration), anomaly
  KNN/vector search now popular for RAG.

### Object vs block vs file — recap for DB-side

- Don't put structured transactional data in S3 — it's not indexed, list
  performance is poor for ad-hoc queries.
- Use Aurora/RDS for relational, DynamoDB for stated-keyed NoSQL,
  ElastiCache for caching/session, Redshift for OLAP, OpenSearch for
  text-search/log analytics, S3 for blobs.

### HA & DR genres worth knowing

| Pattern | RPO | RTO | Mechanism |
|---------|-----|-----|-----------|
| Multi-AZ RDS | near-zero | ~60 s | synchronous standby + DNS |
| Aurora Global DB | seconds | ~1 min | storage replication + managed failover |
| DynamoDB Global Tables | ~1 s | < 1 min | Streams replication; active-active |
| Cross-region S3 replication (S3 RTC) | seconds | none — read the replica bucket | async replication in background |
| Cross-region read replicas (RDS) | seconds-minutes | promote replica | async replication; manual failover |

When interviewers ask RTO/RPO, the trade-off is **synchronous replication**
(low RPO, AP write latency) vs **asynchronous** (low write latency, RPO >
0, lower cost). Aurora Global's storage replication is between the two —
fast but async.