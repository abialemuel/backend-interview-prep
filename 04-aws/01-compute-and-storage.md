# Compute & Storage on AWS

This file covers the compute and storage primitives a backend engineer must
understand, and the **decision framework** interviewers use to probe: EC2 vs
ECS vs EKS vs Lambda, and object vs block vs file storage.

---

## Compute

### EC2 — Elastic Compute Cloud (IaaS)

EC2 gives you virtual machines (instances) running on the Nitro hypervisor
fleet. You pick an instance family, size, AMI, network, and storage; AWS
manages the hardware and virtualization. It is the "maximum control" option
and the default escape hatch when managed services don't fit.

**Instance families** (the part interviewers quiz):

| Family | Optimized for | Examples |
|--------|----------------|----------|
| `m`/`t` | General purpose, balanced CPU/mem/network | `m7i`, `t3`, `t4g` |
| `c` | Compute-bound | `c7gn` |
| `r`/`x`/`z` | Memory-bound (in-memory DBs, SAP HANA, single-threaded hot processes) | `r7iz` (high freq), `x2iedn` |
| `p`/`g`/`inf`/`trn` | GPU / inference / training | `g6`, `p5`, `inf2`, `trn1` |
| `i`/`d`/`h` | High disk throughput, direct-attached NVMe (NoSQL, data warehousing) | `i4i`, `d3` |
| `f` | FPGAs | rare in backend interviews |

- `7` is the current Intel Ice Lake generation; `7a` AMD; `7g`/`8g` Graviton
  (ARM). Graviton is the price/perf story of the last few years — typical
  15–25% cost reduction at equal performance for JIT-compiled/interpreted
  workloads (Go, Java, Python, Node).

**Burstable instances (`t` family)** — earn CPU credits when idle, spend them
under baseline. Below baseline you get the full core; sustained high load
burns credits and is throttled to baseline. `t3`/`t4g` support
**Unlimited mode**: SPI after credits are exhausted, billed per vCPU-hour.
Use for dev, low-traffic web tiers, microservices with spiky load — but
**never** for steady high CPU; cluster on `c`/`m`.

**Placement groups** — control instance physical placement:

- **Cluster** — packed in one rack/raft, low latency, high bandwidth
  (400 Gbps/100 Gbps EFA HPC). Risk: a rack failure takes them all. Use for
  tight tightly coupled HPC / ML training / in-memory caches.
- **Partition** — spread across partitions (logical racks); up to 7 per AZ.
  Distinct partitions don't share rack power/network. Big data clusters
  (HDFS, Cassandra, Kafka) — partition-aware replication across the
  rack-failure domain instead of a single AZ.
- **Spread** — max 7 instances per group, each on a distinct rack. For
  small critical fleets where you cannot tolerate correlated rack failure
  (e.g., a handful of stateful leader nodes).

**AMIs** — the disk image (root snapshot + metadata) used to launch an
instance. Use a **golden AMI pipeline** (Packer, EC2 Image Builder) with
baked-in OS patches, agents, app code, then boot fast from user-data only.
Pin AMI IDs in CloudFormation/Terraform; rebuild on schedule to pull security
patches.

**User-data** — shell/cloud-init script run by the root user on boot
(`cloud-init` on AL2023, `EC2Config`/`EC2Launch` on Windows). Re-runs only on
`restart` with the cloud-init `Always`/`Once` semantics. In practice used
for small bootstrapping; heavy lifting should be in the AMI to keep boot
time deterministic.

**IMDSv2** — Instance Metadata Service. v1 was GET-only and vulnerable to
SSRF (an attacker who could curl `169.254.169.254` from inside the box
could steal instance creds). **IMDSv2 is session-token PUT-then-GET** and
you should require it (`HttpTokens: required`) on every launch template.
The AppSol SIG calls IMDSv2 mandatory; default on new AMIs/launch templates.

**Nitro System** — custom AWS hardware (Nitro Card for I/O, Nitro
Security Chip for hardware root of trust, Nitro Hypervisor lightweight).
Almost all current-gen EC2 is Nitro. Implications: encrypted EBS at no
cost penalty, network ENA offload, hardware-isolated tenants, **no
hypervisor access by AWS employees**, strong virtualization isolation.

#### Instance store vs EBS

- **Instance store** — physically attached NVMe/SSD on the host. Fastest
  I/O (millions of IOPS on `i4i`/`i3en`), no network. **Lost on stop,
  hibernate, or termination**, and lost if the host fails. Use for caches,
  scratch space, buffers, Hadoop/Kafka data dirs that the app can rebuild.
- **EBS** — network-attached block device (`NvMe`/`Xen` block) replicated in
  the AZ. Persists independently of the instance. The default for boot
  volumes and most stateful workloads.

Trade-off interview answer: instance store wins on **latency and cost per
IOPS** when you can tolerate loss; EBS wins on **durability, snapshots,
resizing, and decoupling from the host lifecycle**.

### ECS — Elastic Container Service

AWS-native container orchestration. Simpler than Kubernetes; you don't
manage a control plane. Define a **task definition** (image, CPU/mem, ports,
env, IAM task role, logging, mount points) and run it as a one-off task or
behind an ECS **Service** (desired count, load balancer integration,
auto scaling).

**Launch types:**

- **Fargate** — serverless; AWS runs the containers, you specify CPU/mem.
  No EC2 to manage, no AMIs to patch, pay per vCPU/GB-second. The default
  for most app workloads. Limitations: a few supported regions/volumes,
  fewer host-level knobs (no privileged mode in some configs, no custom
  kernel sysctls), max 16 vCPU / 120 GB per task.
- **EC2 launch type** — you manage an ASG of container instances with the
  ECS agent. You must bin-pack tasks onto hosts and size instances; you get
  full host control (custom daemonsets, host networking, large workloads,
  Spot capacity for cost). More ops burden.

**Key concepts:**

- **Task role** vs **Task execution role** — the former is what your
  container assumes (e.g., to read S3); the latter is used by the ECS agent
  to pull the image, write logs, fetch SS
M secrets.
- **ALB integration** — target type `ip` or `instance`; with Fargate you
  must use awsvpc networking, one ENI per task, and an IP target group. Use
  path/host-based routing to split services behind one ALB.
- **Service Auto Scaling** — target tracking on `ECSServiceAverageCPU`
  /memory or ALB `RequestCountPerTarget`. Combine with
  `schedulable` capacity provider for Fargate or capacity provider strategy
  for EC2 (Spot + on-demand mix).

**When to choose ECS** (interview answer): you run containers, you want
AWS-managed simplicity, you don't need k8s-specific tooling or portability
(Helm charts, kubectl ecosystem), and you value Fargate's no-ops model.

### EKS — Elastic Kubernetes Service

Managed Kubernetes. AWS manages the control plane (API server, etcd,
multi-AZ HA across 3 AZs, SLA) for ~$0.10/hour per cluster plus worker
resources. You manage the data plane:

- **Managed node groups** — ASG-attached EC2 worker nodes, AWS-managed
  AMI (`AL2023_x86_64_STANDARD`), lifecycle hooks for graceful drain.
- **Fargate for EKS** — serverless pods, one ENI per pod, no nodes to
  patch. Higher per-pod cost, some limitations (no DaemonSets, no
  privileged pods, no hostPath), best for light/scale-out services.
- **Self-managed / hybrid** — your own AMI, bottlerocket, specialized
  networking, plus EKS Anywhere for on-prem.

**EKS vs ECS** — pick EKS when:

- You have k8s expertise or a multi-cloud strategy requiring portable
  manifests / Helm / kubectl / Argo / Istio standards.
- You want to run complex workloads needing DaemonSets, custom CNI,
  operators, CRDs, or a large ecosystem of CNCF tools.
- Your org already standardized on k8s for dev/test.

Pick ECS when you don't need k8s abstraction tax and prefer native AWS
integrations (Fargate pay-as-you-go, Cloud Map, ALB) and lower operational
overhead. EKS control plane overhead (per-cluster fee, IAM Roles for
Service Accounts setup, CNI IP exhaustion debugging) is real.

> **Common production failure**: EKS with `aws-vpc-cni` and `/28` Pod ENIs
> exhausting the subnet's secondary IPs in dense clusters. Mitigations:
  larger subnets, `WARM_ENI_TARGET`, custom networking, or security
  groups per pod.

### Lambda — serverless event-driven compute

Run code in response to events without managing servers. **Pay per request +
duration** (GB-second). Behavior worth knowing cold:

- **Memory: 128 MB–10 GB**. vCPU is **proportional to memory** (~1 vCPU at
  ~1.8 GB, full ~6 vCPU at 10 GB). Increasing memory also speeds CPU-bound
  code — a common interview gotcha.
- **Timeout**: up to **15 minutes**. Long workloads should fan out
  (Step Functions) or move to ECS/EKS/Fargate.
- **/tmp storage**: up to 10 GB ephemeral (use for downloads, scratch).
- **Cold start** — first invocation of a function in a fresh execution
  environment pays init time (load runtime, deps, code, init handlers).
  Mitigations: small deployment package, lighter deps, **Provisioned
  Concurrency** (keep N warm environments pre-initialized, pay hourly
  regardless of invocations), and the newer **SnapStart** for Java
  (restore-from-snapshot of init).
- **Execution env reuse** — global/static state persists across warm
  invocations. Use for connection pools (SDK clients, DB clients) outside
  the handler, but **do not** store user/session state there —
  containers can be replaced or scaled concurrency=N at any time.
- **Layers** — shared dependencies (e.g., a common logging layer) capped at
  5 layers, 75 MB zipped total.
- **Concurrency**: 1000/account/region default (can raise via quota), can
  reserve per function and set a per-function throttle.
- **Event sources**: S3, DynamoDB Streams, Kinesis, SQS, SNS, API Gateway
  (REST/HTTP), EventBridge, CloudWatch Logs, ALB, KMS, Secrets Manager
  rotation, and direct invocations (sync `RequestResponse` or async
  `Event`).
- **Versions + aliases + weighted traffic** for canary/linear deploys;
  use with API Gateway stages.

A minimal handler:

```python
import json
import os

def lambda_handler(event, context):
    return {"statusCode": 200, "body": json.dumps({"ok": True, "region": os.environ["AWS_REGION"]})}
```

Handler runtime pulls the event (e.g., an API Gateway proxy event or an
S3 event JSON) and returns `statusCode`/`body`/`headers`.

**When Lambda** (interview answer): event-driven workloads, sparse/uneven
traffic, small functions fitting in 15 min, pay-per-use beats idle
container cost, no infra management desired. Avoid for: steady high
throughput (cheap container beats per-ms billing), long-running jobs, heavy
background daemons, workloads needing OS-level control, or tight latency
SLAs unless you provision concurrency.

### App Runner & Lightsail (brief)

- **App Runner** — fully managed container service: point at a container
  image (ECR) or source repo, auto-build, auto-deploy, auto-scale, TLS,
  public URL. Simpler than ECS+Fargate+ALB; you give up fine-grained VPC
  and network control (Egress VPC support exists). Good for MVPs, internal
  tools, low-traffic services.
- **Lightsail** — virtual private servers, bundles of CPU/RAM/storage/band
  width at a flat price. Includes managed DB, CDN distribution, container
  services. VPS-style product; great for proof-of-concepts, small sites,
  training, NOT production multi-tenant systems.

### Compute decision framework

| Need | Pick |
|------|------|
| Max control, custom kernel, host networking, full OS | **EC2** (+ ASG, ALB) |
| Container simplicity, AWS-native, Fargate = no infra mgmt | **ECS (Fargate)** |
| Portability, k8s ecosystem, multi-cloud, CRDs/Helm/operators | **EKS** |
| Event-driven, pay-per-use, < 15 min, sparse traffic | **Lambda** |
| Point at a container image, get a URL, simplest ops | **App Runner** |
| Tiny static-ish service, flat price, no k8s/containers | **Lightsail** |

Interview "which would you pick" answer template: pick the **lowest-
abstraction** thing that satisfies your non-functional requirements, then
escalate only when you can justify the additional ops burden. E.g., start
Lambda+Fargate; move to EC2 only when you need a kernel module or lib that
Fargate disallows.

---

## Storage

### S3 — object storage

S3 stores **objects** (key + value + metadata + version) in **buckets**
(regional, flat namespace). Durability is **11 9's** (99.999999999%),
designed for **99.99% availability** for Standard (varies by class). No
"directories" — keys with `/` are a UI convention (use a prefix delimiter
in `ListObjectsV2`).

**Consistency** — since Dec 2020, S3 is **strong read-after-write** for
  PUTs of new objects and for overwrite PUTs/deletes. After a successful
  write, any subsequent GET (incl. after key overwrite or DELETE) returns
  the latest version. List operations are eventually consistent.

**Storage classes** (the interview table):

| Class | Use | Retrieval | Cost (rel.) |
|-------|-----|-----------|-------------|
| **Standard** | Hot, frequently accessed | ms | 1x |
| **Standard-IA** | Infrequently accessed but durable (long-lived) | ms | lower storage, retrieval fee |
| **One Zone-IA** | Recreatable, non-critical infrequent data | ms but only 1 AZ | lower still — risk AZ loss |
| **Intelligent-Tiering** | Unknown access patterns; auto-moves objects between tiers via monitoring | ms | small monitoring fee |
| **Glacier Instant Retrieval** | Archive needing ms access (e.g., image thumbnails) | ms | low storage, retrieval fee |
| **Glacier Flexible Retrieval** (formerly Glacier) | Archive, minutes–hours OK | Expedited 1–5 min / Standard 3–5 h / Bulk up to 12 h | very low |
| **Glacier Deep Archive** | Long-term, rare access (compliance) | Standard 12 h / Bulk up to 48 h | lowest |
| **Reduced Redundancy (RRS)** | Deprecated for most uses | | |

**Lifecycle rules**: Transition (Standard → IA after 30 days minimum →
Glacier after 90+ → Deep Archive after 180+) and Expiration (delete old
object versions or abort incomplete multipart uploads). Common pattern:
`Standard → IA after 30d → Glacier IR after 90d → Deep Archive after
180d → expire after 7y for compliance`.

**Versioning**: once enabled, cannot be disabled (only suspended). Every
PUT creates a new version; DELETE creates a delete marker. ENABLE
VERSIONING BEFORE MFA-DELETE if you want to require MFA for delete/version
overwrite. Combine with **S3 Object Lock** for WORM/compliance (Governance
or Compliance mode, retention period, legal hold).

**Encryption**:

- **SSE-S3** — AWS-managed keys, AES-256, free, transparent — the
  default for most workloads since 2023 (default encryption on new
  buckets).
- **SSE-KMS** — AWS KMS Customer Master Key; gives you CloudTrail key
  usage, IAM/KMS policy control, key rotation; subject to KMS quotas
  (RequestQuota per region ~5500-50k decrypts/s) and per-object cost.
  Good when you need granular access audit/revocation.
- **SSE-C** — customer-provided key per request; the key never leaves
  the caller. S3 stores the encrypted object and a wrapped key header.
- **CSE** — client-side encryption before upload (e.g., AES-GCM in
  your app); S3 never sees plaintext.

**Presigned URLs**: time-limited (max 7 days when signed with long-term IAM user keys, longer only with STS
short-term creds since the signing key is the IAM user's long-term key or
the role's STS creds) signed URLs to upload/download without embedding
AWS creds in the client. Use for browser uploads, partner downloads, CI
artifact fetch.

**S3 Select**: push-down SQL projection/filter on the **object**'s
contents (CSV/JSON/Parquet) — returns only matching rows, saving bandwidth
and money.

**Event notifications**: publish to SNS, SQS, or Lambda on
`ObjectCreated`, `ObjectRemoved`, `ObjectAccessed`, `Replication`,
`LifecycleExpiration`. For fanout to multiple consumers prefer EventBridge
notification target (more flexible filtering and multiple targets) over
SNS.

**Performance**:

- **Multipart upload** — split objects ≥ 100 MB and required ≥ 5 GB; up
  to 10000 parts, part size 5 MB–5 GB, max object 5 TB. Parallelizes
  upload across threads/hosts and resumes failures from the failed part
  instead of the whole upload. **Always** for > 100 MB.
- **Byte-range fetches** — GET a part of an object by `Range` header,
  parallelizable; also great for reading only headers (e.g., the
  parse-the-JSONL-header use case).
- **S3 Transfer Acceleration** — upload via the closest edge location
  which forwards over the AWS backbone; great for cross-continent uploads.
- **Prefix scaling**: historical guidance to shard prefixes for 5500
  GET/s per prefix — **largely moot** since 2018; S3 scales automatically.
- **HTTP/2**, **TLS 1.3**, keep-alive on the SDK client for lower per-request
  latency.

### EBS — block storage for EC2

Network-attached block device, AZ-local — an EBS volume is in exactly one
AZ and attached to one EC2 instance in that AZ. To move across AZ, snapshot
to S3 and restore.

**Volume types:**

| Type | Family | Best for | Provisioning |
|------|--------|----------|--------------|
| **gp3** | General purpose SSD | Most workloads (boot, dev, prod) | 3000 IOPS baseline, up to 16000 IOPS / up to 1000 MB/s — **decoupled IOPS from size** unlike gp2 |
| **io2 Block Express** | Provisioned IOPS SSD | Critical high-IOPS / latency (Oracle, SAP HANA, large DB) | up to 256000 IOPS, 99.999% durability, multi-attach across up to 16 instances |
| **io1** | Provisioned IOPS SSD | Older high-IOPS; up to 64000 IOPS / 1000 MB/s | |
| **st1** | Throughput-optimized HDD | Big sequential (Hadoop, log processing, EMR) | up to 500 MB/s, 500 IOPS, cannot boot |
| **sc1** | Cold HDD | Rarely accessed sequential (cheap archive) | up to 250 MB/s, cannot boot |

Key behaviors:

- **Snapshots** are incremental, stored in S3, automatic point-in-time;
  EBS first-backup is full, subsequent deltas. Restored volume is
  lazily loaded from S3 (reads pull blocks over the network; pre-warm with
  `dd`/`fio` if you need steady state immediately). Fast Snapshot Restore
  (FSR) gives instant access for a fee per snapshot per AZ.
- Multi-attach (io1/io2 only) — one volume, up to 16 instances
  concurrently, for clustered filesystems; the application must coordinate
  writes.
- **Encryption** — at-rest encryption with KMS, transparent, no perf
  penalty on Nitro; cannot be removed once encrypted — encryption is on
  per-volume at creation. Enable by default at account level.
- **gp3 vs gp2** — gp3 decouples IOPS from size; gp2 IOPS scaled 3 IOPS/GiB
  capped at 16000, so a 100 GB gp2 gave only 300 IOPS (bad for small DBs)
  — gp3 is the modern default.

### EFS — shared POSIX filesystem (NFS)

Regional NFS filesystem, **multi-AZ by default**, mountable across many
EC2/ECS/EKS instances concurrently. Use for: shared content store, web
serving statics across an ASG, container shared volumes, WordPress
uploads, dev home directories. Throughput/IOPS scale with size unless you
buy **provisioned throughput**. Storage classes: Standard, IA, One-Zone-IA,
with lifecycle policies. Access Points, IAM auth, and encryption in transit
(TELPORT/TLS) supported.

**EFS vs FSx:**

- **FSx for Windows File Server** — SMB, Active Directory integration,
  Windows workloads, SQLServer data dirs.
- **FSx for Lustre** — HPC-grade parallel scratch filesystem, S3-linked,
  used for ML training data and scientific computing.
- **FSx for NetApp ONTAP / OpenZFS** — managed NAS for migration from
  on-prem NetApp/ZFS.

### Instance store (recap)

Ephemeral, physically attached NVMe/SSD. Fastest I/O and lowest latency;
data lost on stop/hibernate/termination or host failure. Use for caches,
scratch, buffers, and big-data data dirs where the app can tolerate
rebuild. Always pair with EBS or remote storage for anything that must
persist — never put a database's master data on instance store alone.

### Object vs block vs file — the rule

- **Object (S3)** when: data is large-ish, immutable once written, accessed
  less frequently than once per request-frame of a DB, logically a single
  blob with metadata, and you want 11-nines durability / lifecycle /
  cross-region cheaply. Anthem: "S3 is not a filesystem."
- **Block (EBS)** when: an OS or a database needs a POSIX block device with
  random R/W and low-latency small reads, attached to exactly one host
  (or a tightly-coupled cluster via multi-attach).
- **File (EFS/FSx)** when: multiple hosts need a shared POSIX/SMB tree at
  the same time without building cluster-fs yourself.

Interview heuristic: if the user said "low-latency random R/W attached to a
host", think EBS; if "payload blob like images/videos/backups", think S3;
if "shared home directory across a fleet", think EFS/FSx.

---

## Worked example: S3 lifecycle policy

A logs bucket accumulates billions of objects and you want to satisfy a
7-year compliance retention while keeping cost reasonable:

```jsonc
{
  "Rules": [
    {
      "ID": "logs-tiering-and-expire",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER_INSTANT_RETRIEVAL" },
        { "Days": 180, "StorageClass": "GLACIER_FLEXIBLE_RETRIEVAL" },
        { "Days": 730, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 30, "StorageClass": "GLACIER_FLEXIBLE_RETRIEVAL" }
      ],
      "Expiration": { "ExpiredObjectDeleteMarker": true },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    },
    {
      "ID": "object-lock-compliance",
      "Status": "Enabled",
      "Filter": { "Prefix": "compliance/" },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 2555 }
    }
  ]
}
```

On a real compliance bucket you would combine this with **Object Lock** in
Compliance mode and a default retention period, so even the root account
cannot shorten retention. Note you must enable versioning before applying
Object Lock, and Object Lock can only be enabled at bucket creation time.