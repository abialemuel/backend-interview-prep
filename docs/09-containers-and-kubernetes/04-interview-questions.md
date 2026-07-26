# Containers & Kubernetes Interview Questions

Graded junior → senior → staff. Junior questions test vocabulary and mechanics; senior questions test production judgment and debugging; staff questions test platform-level trade-offs and design under organizational constraints. Attempt each cold before reading the answer; the **Follow-up** lines are the direction a good interviewer takes the question next.

## Junior

### Q1: What is the difference between a container and a VM?
**Answer:** A VM virtualizes hardware and runs a full guest OS on a hypervisor; a container is a regular Linux process isolated with kernel namespaces (what it can see: PIDs, network, mounts, users) and cgroups (what it can use: CPU, memory), sharing the host kernel. Containers therefore start in milliseconds, are megabytes-to-hundreds-of-megabytes instead of gigabytes, and pack far more densely — but the isolation boundary is the kernel, which is weaker than a hypervisor. That's why multi-tenant platforms add layers (gVisor, Firecracker microVMs, user namespaces — GA in K8s 1.36) and why "containers are VMs but lighter" is the wrong mental model: they're processes with good fencing.

**Follow-up:** How do gVisor or Firecracker change the trade-off? (They reintroduce a stronger boundary — syscall interception or a microVM — at some performance cost, for untrusted-code platforms.)

### Q2: What is the difference between an image and a container?
**Answer:** An image is an immutable, layered filesystem snapshot plus metadata (entrypoint, env, ports), content-addressed by SHA-256 digest and stored in a registry. A container is a running instance of an image: the image's read-only layers with a thin writable layer on top plus a live process. Class vs instance — fifty containers from one image share the read-only layers on disk and each get their own writable layer. Anything written to the writable layer is lost when the container is removed, which is why persistent data goes in volumes. Bonus point: tags are mutable pointers, digests are immutable — production systems should deploy by digest.

**Follow-up:** Where does a stopped container's writable layer go? (It persists until `docker rm` — which is why `docker ps -a` disk usage surprises people.)

### Q3: Why does the order of Dockerfile instructions matter?
**Answer:** Each filesystem-changing instruction produces a layer, and the build cache reuses layers until the first instruction whose inputs changed — after which everything below rebuilds. So you order least- to most-frequently changing: base image, then dependency manifests (`COPY go.mod go.sum` + `RUN go mod download`), then the source (`COPY . .`), then the build. That way editing application code doesn't re-download dependencies. The second-order effect: deleting files in a later layer doesn't shrink the image (the earlier layer still ships), so install-and-cleanup must share one `RUN`.

**Follow-up:** How do BuildKit cache mounts change this? (`RUN --mount=type=cache` persists package caches across builds without baking them into layers.)

### Q4: What is a Pod, and why not just say "container"?
**Answer:** The Pod is Kubernetes' smallest deployable unit: one or more containers that share a network namespace — one IP, localhost between them — and can share volumes. Most pods are single-container, but the abstraction exists for tightly coupled helpers (sidecars: proxies, log shippers), which since 1.33 are natively supported as restartable init containers. Pods are mortal: they aren't rescheduled if their node dies, they're *replaced* with new pods (new name, new IP) by a controller. That's why you deploy via Deployments and address pods via Services rather than creating bare pods or caching pod IPs.

**Follow-up:** When would you actually put two containers in one pod? (Only when they must share fate and network — proxy, log shipper; if they scale independently, they are separate Deployments.)

### Q5: What does a Service do, and what Service types exist?
**Answer:** Pod IPs churn as pods are replaced, so a Service gives a stable virtual IP and DNS name that load-balances over pods matching its label selector, with endpoints updated automatically as pods come and go (gated by readiness). Types: `ClusterIP` (in-cluster only, the default), `NodePort` (opens a high port on every node), `LoadBalancer` (provisions a cloud LB), `ExternalName` (a DNS alias), and headless (`clusterIP: None` — DNS returns pod IPs directly, used by StatefulSets and gRPC client-side balancing). For HTTP routing of many services you put an Ingress or Gateway API in front rather than one LoadBalancer each.

**Follow-up:** Why does a Service sometimes have zero endpoints while pods are Running? (Label mismatch, or pods Running but not Ready — check EndpointSlices.)

### Q6: ConfigMap vs Secret — and are Secrets secure by default?
**Answer:** Both inject config as env vars or mounted files so one image runs in every environment; ConfigMaps for non-sensitive values, Secrets for credentials. Secrets are **not** encrypted by default — base64 is encoding, not encryption, and anyone with RBAC read access (or raw etcd access) sees plaintext. Hardening: etcd encryption at rest via a KMS provider, tight RBAC on Secret reads, and preferably an external source of truth (Vault, AWS Secrets Manager) synced via External Secrets Operator or mounted via CSI driver. Also worth saying: env-var config needs a pod restart to pick up changes; mounted files update in place but the app must re-read them.

**Follow-up:** How would you rotate a Secret without downtime? (External manager + mounted files the app re-reads, or a checksum annotation to trigger a rolling restart.)

### Q7: What happens, end to end, when you run `kubectl apply -f deployment.yaml`?
**Answer:** kubectl sends the object to the API server, which authenticates, authorizes (RBAC), runs admission (defaulting, validation, policy webhooks), and persists it to etcd. The Deployment controller notices (via watch), creates a ReplicaSet; the ReplicaSet controller creates Pod objects; the scheduler sees pods with no node, filters nodes by fit (requests, affinity, taints) and binds each pod to one; that node's kubelet sees the binding, pulls the image via containerd, starts containers, runs probes, and reports status back. Everything is watch-based reconciliation against the API server — no component calls another directly, which is why the system converges even when steps fail and retry.

**Follow-up:** Where can this pipeline get stuck, and how does each stage surface it? (Admission webhook down → apply fails; unschedulable → Pending with an event; image missing → ImagePullBackOff.)

## Senior

### Q8: Explain requests vs limits and the QoS classes. What's your production policy?
**Answer:** Requests are what the scheduler reserves — a pod only lands on a node with that much unallocated capacity — and set the CPU weight under contention. Limits are runtime caps: CPU over the limit is throttled (latency spikes, no kill); memory over the limit is OOMKilled, because memory can't be throttled. QoS falls out of how you set them:

- requests == limits everywhere → `Guaranteed`, evicted last under node pressure
- some requests set → `Burstable`
- nothing set → `BestEffort`, evicted first

Production policy to defend: memory requests = limits (incompressible resource — don't gamble with OOM), CPU requests set honestly from observed usage, and usually **no CPU limit** — throttling hurts tail latency while the request already guarantees fair share. Since 1.35 in-place pod resize is GA, so resizing no longer forces a restart. And honest requests matter beyond scheduling: HPA utilization and Karpenter bin-packing both key off them.

**Follow-up:** Why is CPU throttled but memory killed? (CPU is compressible — the kernel can defer cycles; memory is not — reclaiming it means evicting something.)

### Q9: Walk me through liveness, readiness, and startup probes — and how a liveness probe can cause an outage.
**Answer:** Readiness gates traffic: fail → removed from Service endpoints, no restart. Liveness detects a wedged process: fail `failureThreshold` times → container restarted. Startup covers slow boots: until it passes, the other probes are disabled. The outage pattern: a liveness handler that checks a dependency like the database. The DB blips, liveness fails fleet-wide, kubelet restarts every pod at once, and a brief degradation becomes a full outage with cold caches and connection storms — possibly a restart loop if the DB is still down. Rule: liveness checks only in-process health; dependency checks belong in readiness (and even there, carefully — dropping the whole fleet out of rotation because a shared dependency is slow can also be worse than serving degraded). Second failure mode: slow-booting app, no startup probe, tight liveness → killed mid-boot forever, a self-inflicted CrashLoopBackOff.

**Follow-up:** Your readiness probe checks the DB and the DB is degraded, not down. What behavior do you want? (Usually serve degraded rather than empty the endpoint list — readiness on shared dependencies needs a floor.)

### Q10: A pod keeps restarting — walk me through debugging it.
**Answer:** A structured pass, narrating why at each step:

1. `kubectl get pods` — confirm the class of failure: CrashLoopBackOff with climbing RESTARTS (vs Pending or ImagePullBackOff, which are different problems).
2. `kubectl describe pod` — read the last-termination state and the Events. Exit code 137 + reason OOMKilled → memory path. Events full of liveness failures → probe path. Otherwise → app-crash path.
3. **OOM path**: compare `kubectl top pod` to the memory limit; fix the leak or raise the limit; check the runtime sees the cgroup (GOMEMLIMIT, container-aware JVM). If usage looks fine but spikes kill it, it's a burst profile problem — limits are enforced instantaneously, not on averages.
4. **Probe path**: if the app is healthy but slow to boot, add a startup probe rather than loosening liveness; if the liveness handler checks dependencies, fix the handler.
5. **Crash path**: `kubectl logs --previous` — the *previous* container has the stack trace; the current one is a fresh restart. Empty logs mean it died before logging: bad entrypoint, missing ConfigMap/Secret key, unreadable file — `kubectl debug` with an ephemeral container to inspect, which works even on distroless images with no shell.
6. If it crashes only on some nodes or after a config change, diff env: `kubectl get pod -o yaml`, recent rollouts (`kubectl rollout history`), node conditions.

One more signature worth volunteering: exit 137 during *shutdown* rather than under memory pressure means the app ignored SIGTERM and got SIGKILLed at the grace-period deadline — a graceful-shutdown bug, not a resource problem.

**Follow-up:** The pod is fine but only on this deploy it crashes in prod, not staging. (Diff the environments: ConfigMaps/Secrets, resource limits, node architecture — arm64 vs amd64 images — and admission-injected sidecars.)

### Q11: How does a rolling update work under the hood, and how do you roll back?
**Answer:** Changing the pod template makes the Deployment controller create a new ReplicaSet and step it up while stepping the old one down, bounded by `maxSurge`/`maxUnavailable` (default 25%/25%) and gated by readiness — a new pod must be Ready before more old ones are removed. Old ReplicaSets are retained at zero replicas (`revisionHistoryLimit` deep), so `kubectl rollout undo` just re-scales the previous ReplicaSet — near-instant, no rebuild. `minReadySeconds` lets a pod bake before counting as available; `progressDeadlineSeconds` flags a stuck rollout. The catch candidates miss: during the rollout both versions serve simultaneously, so schema and API changes must be N/N+1 compatible — the rollback story dies the moment version N+1 runs a destructive migration.

**Follow-up:** Why might a rollback *not* fix the incident? (Destructive migration already ran, bad config lives in a ConfigMap not the template, or the trigger was external.)

### Q12: Design a zero-downtime deploy for an HTTP API, DB migrations included.
**Answer:** Three layers, and all three are required:

- **Kubernetes mechanics**: RollingUpdate with sane surge; a readiness probe that verifies the pod can actually serve; ≥3 replicas spread across zones via topologySpreadConstraints; a PDB (`minAvailable: 2`) so voluntary disruptions (node drains, consolidation) can't stack on top of the rollout.
- **Traffic/connection handling**: graceful shutdown — on SIGTERM stop accepting, drain in-flight, exit within `terminationGracePeriodSeconds` — plus a `preStop` sleep of a few seconds, because endpoint removal and SIGTERM race; without it requests land on a pod that already stopped listening. Verify the process actually receives SIGTERM (exec-form entrypoint, PID 1 handling).
- **Data compatibility**: expand/contract migrations — deploy the additive schema change first, then code that handles both shapes, contract in a later release. Never ship a migration that breaks version N while N and N+1 run side by side.

Operationally: `kubectl rollout status` as the CI gate, automatic `rollout undo` on failed post-deploy checks, and for riskier changes a canary slice via Argo Rollouts or Gateway API weighted routing before full rollout. If asked "how do you *know* it's zero-downtime" — a synthetic probe hitting the service continuously during deploys is the honest answer.

**Follow-up:** Your traffic is long-lived WebSockets — what changes? (Draining takes minutes, not seconds: longer grace periods, connection draining with client reconnect logic, and rollouts paced accordingly.)

### Q13: When do you use a StatefulSet over a Deployment — and would you run Postgres on Kubernetes?
**Answer:** StatefulSet when replicas are not interchangeable: stable names and per-pod DNS (`kafka-0.kafka...`), per-replica PVCs that reattach to the same identity on reschedule, ordered rollout — what quorum systems and databases need. Deployment for everything stateless, i.e. most backend services. On Postgres: a trade-off, not a taboo. With a mature operator (CloudNativePG) handling replication, failover, and backups, and a platform team owning the cluster, running it on K8s is reasonable and increasingly common. A bare StatefulSet maintained by an app team is the wrong answer — the StatefulSet gives identity and storage, not database operations — and for most product teams a managed service like RDS buys more reliability per engineer-hour. The interviewer wants the boundary reasoning, not dogma in either direction.

**Follow-up:** What breaks if you put a database behind a normal (non-headless) Service? (Writes load-balance across primary and replicas; you need per-pod DNS and topology-aware routing from the operator.)

### Q14: How do HPA and cluster-level autoscaling interact? Where does Karpenter fit?
**Answer:** Two layers. HPA changes `replicas` to hold a target metric — CPU utilization relative to *requests* via metrics-server, or queue depth/Kafka lag via KEDA (including scale-to-zero), with damped scale-down to prevent flapping. When new pods don't fit, they go Pending — that's the signal for the node layer. Cluster Autoscaler scales pre-defined node groups; Karpenter (CNCF, v1 API stable) provisions best-fit instances directly from the Pending pods' aggregate requirements — faster (tens of seconds), better bin-packing, first-class Spot handling — and its consolidation loop actively replaces underutilized nodes to cut cost. Two implications to flag: requests must be honest, since both layers key off them; and consolidation means nodes get recycled routinely, so PDBs and graceful shutdown stop being optional hygiene and become correctness requirements. VPA rounds it out for right-sizing requests — but never HPA and VPA on the same metric.

**Follow-up:** What stops HPA and Karpenter from thrashing against each other? (HPA stabilization windows, Karpenter consolidation settings and disruption budgets — and PDBs as the backstop.)

### Q15: Kubernetes networking is flat by default. How do you lock down east-west traffic?
**Answer:** By default any pod reaches any pod, cross-namespace included. NetworkPolicies are the pod-level firewall: select pods, allowlist ingress/egress by pod/namespace selector and port; once any policy selects a pod, the covered direction becomes default-deny beyond the allows. Posture: a default-deny policy per namespace, explicit allows per service pair, and an explicit DNS-egress allow to kube-system — forgetting it breaks everything in maximally confusing ways. Enforcement is the CNI's job (Cilium, Calico); on a CNI without support the objects are **silently inert**, which is worth testing rather than assuming. For identity-based enforcement plus encryption in transit, mesh mTLS adds workload identity on top — NetworkPolicy (L3/L4, IP/label-based) and mesh authorization (L7, identity-based) complement rather than replace each other.

**Follow-up:** How do you roll out default-deny to a brownfield cluster without breaking everything? (Observe flows first — Cilium Hubble or VPC flow logs — generate allows from observed traffic, apply namespace by namespace with a rollback path.)

### Q16: What are CRDs and operators, and when would you write one?
**Answer:** A CRD adds a new resource type to the API server — stored in etcd, covered by RBAC, watchable like any built-in. An operator is the controller that reconciles those objects: a loop encoding operational knowledge ("desired version changed → rolling upgrade with pre-upgrade backup; primary unhealthy → promote a replica"). CRD is the schema, operator is the behavior — the same pattern as built-in controllers, just out-of-tree; it's the extension mechanism behind cert-manager, Prometheus Operator, ArgoCD, and Karpenter. Write one only when you genuinely need continuous reconciliation of domain state that generic tooling can't express — if the need is "install these manifests with parameters," that's Helm/Kustomize, not an operator. And weigh the cost honestly: an operator is production software that can destroy data at machine speed, so prefer mature community operators over writing your own.

**Follow-up:** How does an operator upgrade safely when its CRD schema changes? (CRD versioning with conversion webhooks — and this is where operator quality really shows.)

## Staff

### Q17: Sidecar vs ambient service mesh — and how would you decide whether to adopt a mesh at all?
**Answer:** A mesh moves mTLS + workload identity, per-request traffic management (retries, timeouts, circuit breaking, canary splits — and it fixes gRPC connection pinning), and uniform telemetry out of application code into the platform. Sidecar model (classic Istio, Linkerd) injects a proxy per pod: full L7 features, but fleet-wide CPU/memory/latency overhead and upgrades that touch every pod. Ambient (GA in Istio since 1.24) splits the layers: a per-node L4 ztunnel gives everything mTLS and identity cheaply, and per-namespace waypoint proxies add L7 only where needed — pay-for-what-you-use, and the direction the ecosystem has moved; Cilium does similar with eBPF. Adoption call: a mesh pays off in polyglot, many-team orgs where you can't standardize this in libraries, or under a compliance mandate for encrypted, identity-verified traffic. A small homogeneous shop gets most of the value from a shared client library plus a good gateway, without inserting a proxy and a serious operational dependency into every request path. Adopting today, I'd start ambient/L4-only — mTLS and identity first — and add waypoints per namespace as concrete L7 needs appear.

**Follow-up:** What is the operational cost you are signing up for? (A control plane in the request path: proxy upgrades, certificate rotation, mesh-config outages — the mesh becomes tier-0 infrastructure.)

### Q18: Design multi-tenancy for 30 product teams on shared clusters.
**Answer:** Namespace-per-team-per-environment as the unit of tenancy, with guardrails in four layers:

- **Identity/access**: RBAC bound to IdP groups, team-scoped Roles, no cluster-admin outside platform. ServiceAccounts with IRSA/Pod Identity for cloud access so nodes hold no shared credentials. Audit `exec` and `create pods` — both are identity-escalation paths.
- **Resources**: ResourceQuotas and LimitRanges per namespace so no team can starve the cluster; priorityClasses for preemption order; Karpenter NodePools or tainted node groups to isolate special workloads (GPU, compliance) onto dedicated capacity.
- **Network**: default-deny NetworkPolicies per namespace with explicit cross-team allows; mesh mTLS for workload identity if the estate is polyglot.
- **Policy**: admission control via ValidatingAdmissionPolicy (CEL, in-tree) or Kyverno/Gatekeeper — non-root, no privileged, org-registry images only, mandatory requests — plus Pod Security Standards at `restricted`.

The honest caveat that separates staff answers: namespaces are a *soft* boundary — one kernel, one control plane, shared CoreDNS and ingress. For hostile tenants or hard compliance isolation, separate clusters (or gVisor/user-namespace runtimes) are the real boundary; shared clusters are a cost/agility optimization for mutually trusting teams. Deliver everything through a paved road — templated charts, GitOps app-per-team — so the secure path is the easy path.

**Follow-up:** A team asks for cluster-admin "temporarily." (Time-boxed break-glass via the IdP with audit, not a standing binding — and ask what capability they actually need.)

### Q19: Ingress vs Gateway API — you're building a new platform; which and why?
**Answer:** Gateway API, without much hesitation — Ingress is feature-frozen; Gateway API is GA (v1.3+) and is where controller investment goes. The concrete wins: **role separation** — platform owns `GatewayClass`/`Gateway` (the LB, TLS, listeners), app teams own `HTTPRoute`s that attach to it, ending both the shared-Ingress merge-conflict problem and per-team-LB cost sprawl; **expressive, portable routing** as typed fields instead of vendor annotations — header/method matching, weighted backends (native canary), rewrites, `GRPCRoute`, TCP/TLS routes — portable across Envoy Gateway, Istio, Cilium, and cloud controllers; and the same API configures mesh traffic (GAMMA), so north-south and east-west routing converge on one model. For an existing estate I wouldn't force-migrate working Ingresses — run both, migrate route-by-route, put new services on Gateway API. The trap is treating this as YAML aesthetics; the actual win is the ownership model.

**Follow-up:** How do you migrate 200 existing Ingresses? (Run both controllers side by side, ingress2gateway for the mechanical conversion, migrate by hostname with DNS cutover, keep Ingress read-only frozen during the move.)

### Q20: Your cluster is two versions behind. How do you run the upgrade safely, and what breaks in practice?
**Answer:** Kubernetes supports roughly the last three minors (~14 months of patches) and you can only skip zero — two behind means two sequenced hops, and the meta-point is that upgrading is a perpetual process, not a project. Per hop: control plane first (managed on EKS/GKE), then node pools by rolling replacement — cordon, drain (the eviction API respects PDBs), replace; Karpenter's drift feature rolls nodes onto the new AMI automatically. Before each hop: audit for removed APIs (Pluto, `kubectl-convert`) — manifests *and Helm release state* pinned to removed API versions are breakage source number one; check CNI/CSI/addon and operator compatibility matrices — the CNI is the outage-shaped one; and verify admission webhooks, because a down webhook with `failurePolicy: Fail` can block all pod creation cluster-wide. What actually breaks in practice: workloads with no PDB drained all-at-once by parallel node replacement, single-replica apps with downtime nobody signed off, apps that ignore SIGTERM losing in-flight work, and *too-strict* PDBs (minAvailable = replicas) stalling drains forever. Sequence staging → canary node pool → prod with a smoke suite between. Closing observation worth making: upgrade pain is a lagging indicator of workload hygiene — fix PDBs, graceful shutdown, and replica counts, and upgrades become routine.

**Follow-up:** How do you find every client still calling a deprecated API? (API server audit logs / `apiserver_requested_deprecated_apis` metric — before the hop, not after.)

### Q21: P99 latency doubled after migrating a service to Kubernetes. Same code, same DB. Where do you look?
**Answer:** Ordered by likelihood, each with its metric:

1. **CPU throttling** — a CPU limit sized from average usage throttles every burst; check `container_cpu_cfs_throttled_periods_total`. Fix: raise or drop the CPU limit, keep the request. This is the single most common answer.
2. **Load-balancing skew** — gRPC/HTTP2 keep-alives pin connections to one pod through a ClusterIP Service; one hot pod owns the P99. Fix: headless Service + client-side LB, or a per-request proxy/mesh.
3. **New hops and DNS** — mesh sidecar, ingress, conntrack; and the classic `ndots:5` issue where every lookup tries multiple search-domain suffixes first. Fix: FQDNs (trailing dot) or `dnsConfig` tuning, plus NodeLocal DNSCache.
4. **Runtime vs cgroup mismatch** — GC sizing off node resources instead of limits: set `GOMEMLIMIT`/`GOMAXPROCS`, confirm JVM container-awareness.
5. **Topology** — pods now a zone away from the DB (cross-AZ hop adds ~1ms and money); check zone spread vs the dependency.
6. **Churn** — aggressive HPA or Karpenter consolidation cycling pods with slow warmup; cold caches and JIT show up directly in P99.

The signal the interviewer wants: throttling and connection pinning are the two Kubernetes-specific usual suspects, and each hypothesis gets checked with a metric before anything is tuned.

**Follow-up:** Everything checks out but P99 is still 2x — what next? (Distributed tracing to find *where* the time goes before touching anything else; per-hop spans beat guessing.)

## Rapid-fire

One-liners that come up as warm-ups or tie-breakers — answer without hesitating:

| Question | Answer |
| --- | --- |
| Exit code 137? | SIGKILL — OOMKilled, or grace period expired after ignored SIGTERM |
| Exit code 143? | SIGTERM handled — a clean orchestrated shutdown |
| `cordon` vs `drain`? | Cordon marks unschedulable (nothing new lands); drain also evicts what's there |
| Why is my `latest` image not updating? | `imagePullPolicy: IfNotPresent` + mutable tag; deploy unique tags or digests |
| Pod stuck `Terminating`? | Finalizer not removed, or process ignoring SIGTERM; check `kubectl get pod -o yaml` finalizers |
| ConfigMap changed, pods unchanged — why? | Env vars snapshot at start; trigger a rollout (checksum annotation) |
| Difference between `kubectl apply` and `create`? | Apply is declarative upsert with merge; create fails if the object exists |
| What creates EndpointSlices? | The EndpointSlice controller, from Service selector + Ready pods |
| Init container vs sidecar? | Init runs to completion before app containers; sidecar (1.33+) runs alongside, restartable |
| Who kills an OOM pod — kubelet or kernel? | The kernel OOM killer, at the cgroup limit; kubelet *evicts* on node pressure |
| Can two containers in a pod bind the same port? | No — shared network namespace, one port space |
| What does `terminationGracePeriodSeconds` default to? | 30 seconds |
| HPA scales on 90% of what? | Utilization relative to *requests*, not limits, not node capacity |
| Taints vs affinity? | Taints repel pods from nodes (opt-in via toleration); affinity attracts pods to nodes/pods |
| Is a NetworkPolicy deny rule possible? | No — policies are allowlist-only; deny is the absence of an allow once selected |
