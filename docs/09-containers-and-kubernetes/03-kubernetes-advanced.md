# Kubernetes Advanced

## Workloads beyond Deployments

### StatefulSets

Deployments treat pods as interchangeable cattle; **StatefulSets** give each pod a stable identity for workloads that care who they are — databases, Kafka, anything with per-replica state:

- Stable names (`kafka-0`, `kafka-1`, ...) and stable per-pod DNS via a headless Service (`kafka-0.kafka.prod.svc.cluster.local`).
- Per-pod storage via `volumeClaimTemplates`: each replica gets its own PVC that survives rescheduling and *reattaches to the same identity* — `kafka-1` always gets `kafka-1`'s disk. Deleting the StatefulSet does not delete the PVCs by default.
- Ordered, one-at-a-time startup, scaling, and rolling updates (pod N waits for N-1 to be Ready).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka            # the headless Service providing per-pod DNS
  replicas: 3
  selector: { matchLabels: { app: kafka } }
  template:
    metadata: { labels: { app: kafka } }
    spec:
      containers:
        - name: kafka
          image: apache/kafka:4.0.0
          volumeMounts: [{ name: data, mountPath: /var/lib/kafka }]
  volumeClaimTemplates:         # one PVC per replica, bound to the identity
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 100Gi } }
```

The honest senior-level caveat: a StatefulSet gives you stable identity and storage, *not* a managed database. Replication, failover, backups, and topology are the application's problem — which is why serious stateful workloads on K8s run under an operator (CloudNativePG, Strimzi) or stay on RDS. "Would you run Postgres on Kubernetes?" is a trade-off question, not a yes/no one: with a mature operator and a platform team, yes; as a one-off StatefulSet maintained by an app team, usually not worth it over a managed service.

### DaemonSets

One pod per node (or per node matching a selector), added automatically as nodes join. This is the deployment mode for node-level agents: log shippers (Fluent Bit), metrics (node-exporter), CNI and CSI components, security agents. If your answer to "how do we collect logs from every node" is a Deployment with lots of replicas, you've missed the primitive.

### Jobs and CronJobs

A **Job** runs pods to completion — `completions`, `parallelism`, `backoffLimit` (retries), `activeDeadlineSeconds` (timeout), and `ttlSecondsAfterFinished` for cleanup. Indexed Jobs give each pod a completion index for work-sharding. A **CronJob** creates Jobs on a schedule; the two flags that bite people are `concurrencyPolicy` (`Allow`/`Forbid`/`Replace` — what happens when the previous run is still going) and `startingDeadlineSeconds` (missed-schedule tolerance).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: reconcile-payments }
spec:
  schedule: "*/15 * * * *"
  concurrencyPolicy: Forbid          # skip if the previous run is still going
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: reconcile
              image: registry.example.com/reconciler:abc123
```

Design job pods to be idempotent: retries, duplicate runs after a missed deadline, and `Replace` policies make at-least-once the real contract, not exactly-once. And CronJob is best-effort scheduling — for "must fire exactly on time or page someone," pair it with a monitor on Job completions rather than trusting the schedule.

## CRDs and operators

A **CustomResourceDefinition** teaches the API server a new object type — after applying a CRD, `kubectl get postgresclusters` works like any built-in, with the object stored in etcd, covered by RBAC, and watchable. An **operator** is the controller that gives such objects behavior: a reconciliation loop encoding operational knowledge — "if desired version != actual, perform a rolling upgrade replica by replica, take a backup first, promote a replica on primary failure."

That's the whole pattern: **CRD = schema, operator = control loop**, exactly how built-in controllers work, just out-of-tree. It's how the ecosystem extends Kubernetes — Prometheus Operator, cert-manager (`Certificate` CRs), ArgoCD (`Application` CRs), Strimzi, Karpenter (`NodePool` CRs). Operators are typically built with controller-runtime/Kubebuilder in Go. The trade-off to voice: an operator is software you now depend on for correctness of critical infra — a buggy operator can delete data at machine speed, so prefer mature community operators over writing your own, and reach for an operator only when reconciliation logic is genuinely needed (Helm covers "just install these manifests").

## Service mesh

A service mesh moves cross-cutting service-to-service concerns out of application code into the platform: **mTLS everywhere** (encryption + workload identity), **traffic management** (per-request load balancing — fixes the gRPC pinning problem — retries, timeouts, circuit breaking, canary traffic splits), and **uniform telemetry** (consistent latency/error metrics across all services regardless of language).

- **Sidecar model** (classic Istio, Linkerd): an Envoy (or linkerd2-proxy) container injected into every pod intercepts all traffic. Full L7 feature set, but per-pod CPU/memory overhead, added latency, and historically an upgrade tax across the entire fleet.
- **Ambient model** (Istio ambient, GA since Istio 1.24): no sidecars — a per-node L4 proxy (ztunnel) provides mTLS and identity for everything, and an optional per-namespace Envoy ("waypoint") adds L7 features only where needed. Pay-for-what-you-use; this is where the ecosystem has been heading through 2025–26. Cilium offers a similar sidecarless mesh built on eBPF.

The staff-level answer to "should we adopt a mesh": it buys the most in polyglot, many-team environments where mTLS/retries/telemetry can't realistically be standardized in libraries; a small homogeneous shop often gets 80% of it from a shared library and an ingress. Don't claim a mesh is free — you're inserting a proxy into every request path and adopting a significant operational dependency.

## Network policies

Default Kubernetes networking is flat: every pod can reach every pod, cross-namespace included. A **NetworkPolicy** is a pod-level firewall — select pods, then allowlist ingress/egress by pod/namespace selectors and ports. Once any policy selects a pod, that pod's traffic in the covered direction is default-deny except what's allowed. Enforcement is the CNI's job (Cilium, Calico); on a CNI without support the objects are silently inert — a nasty gotcha.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-ingress, namespace: prod }
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: gateway } }
      ports: [{ port: 8080 }]
```

Standard posture: default-deny per namespace, then explicit allows; always remember DNS egress to kube-system, or everything breaks mysteriously.

## RBAC

Authorization model: **Role** (namespaced) or **ClusterRole** (cluster-wide) lists allowed verbs on resources; a **RoleBinding**/**ClusterRoleBinding** grants that to users, groups, or **ServiceAccounts**. Rules are purely additive — no deny rules.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: deployer, namespace: prod }
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: ci-deployer, namespace: prod }
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: deployer }
subjects:
  - kind: ServiceAccount
    name: ci
    namespace: prod
```

The parts that matter for a backend engineer: every pod runs as a ServiceAccount, and that identity is what your app uses if it calls the K8s API — scope it minimally and set `automountServiceAccountToken: false` when the app doesn't need the API at all. On EKS, IRSA/Pod Identity binds a ServiceAccount to an IAM role, so pods get cloud permissions without node-wide credentials. Escalation paths to name in a security discussion: `create pods` in a namespace effectively grants any ServiceAccount in it (mount its token), and `exec` into pods is equivalent to the pod's identity — audit both. `kubectl auth can-i --as=system:serviceaccount:prod:ci create deployments -n prod` is the fastest way to verify a binding does what you think.

## PodDisruptionBudgets

A **PDB** bounds *voluntary* disruptions — node drains for upgrades, cluster-autoscaler consolidation, `kubectl drain` — via `minAvailable` or `maxUnavailable` over a label selector. The eviction API refuses to evict below the budget; drains wait. It does nothing for involuntary failures (node crash, OOM).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: api } }
```

Two failure modes worth naming: no PDB means a routine node-pool upgrade can take all replicas of your service down at once; a too-strict PDB (`minAvailable: 100%`, or a PDB over a single-replica app) blocks drains entirely and stalls cluster upgrades — platform teams will find you. PDB + multiple replicas + `topologySpreadConstraints` across zones is the standard availability trio.

## Cluster autoscaling and Karpenter

HPA adds pods; when pods go Pending because no node fits, something must add nodes:

- **Cluster Autoscaler** scales pre-defined node groups (ASGs) up and down. Solid but rigid: it can only pick from instance shapes you configured per group, and scale-up latency runs minutes.
- **Karpenter** (CNCF project, donated by AWS; v1 API stable) provisions nodes directly, no node groups: it reads the aggregate shape of Pending pods and launches best-fit instances from the full instance catalog in tens of seconds. Its **consolidation** loop actively replaces underutilized or expensive nodes with cheaper/fewer ones, and it handles Spot interruption gracefully — the pods-must-tolerate-disruption contract (PDBs, graceful shutdown) becomes essential. Configured via `NodePool` CRs (allowed instance families, capacity type spot/on-demand, expiry). It has become the default choice on EKS, with AKS support maturing.

Rounding out scheduling vocabulary: **taints/tolerations** keep pods *off* nodes (dedicated GPU pools), **affinity/anti-affinity** and **topologySpreadConstraints** pull pods toward/away from each other (spread replicas across zones), and **priorityClasses** decide who gets preempted when capacity is tight.

## Gateway API

The **Gateway API** is the successor to Ingress (which is feature-frozen), GA since v1.0 and at v1.3+ now. Improvements over Ingress:

- **Role separation**: infra teams own `GatewayClass`/`Gateway` (the LB itself, TLS, listeners); app teams own `HTTPRoute`s that attach to it — no more one giant Ingress or annotation collisions.
- **Expressive, portable routing** as first-class fields instead of vendor annotations: header/method matches, traffic weighting (native canary), redirects/rewrites, `GRPCRoute` and TCP/TLS routes.
- One controller ecosystem: Envoy Gateway, Istio, Cilium, nginx-gateway, and cloud LB controllers all implement it; it is also the standard way to configure mesh traffic (GAMMA).

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api, namespace: prod }
spec:
  parentRefs: [{ name: shared-gateway, namespace: infra }]   # attach to platform-owned Gateway
  hostnames: [api.example.com]
  rules:
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs:
        - { name: api, port: 80, weight: 90 }
        - { name: api-canary, port: 80, weight: 10 }         # native canary, no annotations
```

New platforms should default to Gateway API; existing Ingress isn't broken, just static. Migration is route-by-route.

## Debugging workflows

The interview scenario is usually "pod keeps restarting / won't start — walk me through it." Have a sequence, not a vibe:

```bash
kubectl get pods                         # STATUS + RESTARTS: which failure class?
kubectl describe pod <pod>              # Events at the bottom: probe failures, OOMKilled, image pulls, scheduling
kubectl logs <pod> --previous           # what the *crashed* container printed
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it <pod> -- sh            # live inspection (if there's a shell)
kubectl debug -it <pod> --image=busybox --target=app   # ephemeral container — the answer for distroless images
kubectl debug node/<node> -it --image=busybox          # node-level inspection
```

Failure signatures to pattern-match instantly:

| Symptom | Meaning | First moves |
| --- | --- | --- |
| `Pending` | Scheduler can't place it | `describe` → insufficient CPU/memory, unsatisfiable affinity, taints, or PVC unbound; check autoscaler |
| `ImagePullBackOff` | Image fetch failing | Typo/tag doesn't exist, missing `imagePullSecrets`, registry auth/network |
| `CrashLoopBackOff` | Container starts, exits, restarted with exponential backoff (caps at 5m) | `logs --previous` for the crash; check exit code in `describe`; config/env missing, bad command, liveness probe killing a slow boot |
| `OOMKilled` (exit 137) | Kernel killed it for exceeding memory limit | Raise the limit or fix the leak; check `kubectl top` vs limits; watch for JVM/Go runtimes unaware of cgroup limits (set `GOMEMLIMIT`, modern JVMs auto-detect) |
| `CreateContainerConfigError` | Referenced ConfigMap/Secret key missing | `describe` names the missing key |
| Ready 0/1, running fine | Readiness probe failing | Probe path/port wrong, or a dependency it checks is down; pod gets no traffic |
| Service unreachable | Selector/endpoints problem | `kubectl get endpointslices` — empty means label mismatch or nothing Ready; then check NetworkPolicy |

The CrashLoopBackOff drill specifically: (1) `describe` — was the last termination OOMKilled (137) or an app exit code? Are events showing liveness failures? (2) `logs --previous` — the current container hasn't crashed yet; the previous one has the stack trace. (3) If logs are empty, the process died before logging: bad entrypoint, missing binary, unreadable config — `kubectl debug` with an ephemeral container and inspect. (4) If it's the liveness probe killing a healthy-but-slow app, add a startup probe rather than loosening liveness. Exit code 137 during *shutdown* (not OOM) means the app ignored SIGTERM and got SIGKILLed at the grace-period deadline.

!!! note
    When you narrate this in an interview, say *why* each step: "logs --previous because the restarted container is fresh; the evidence is in the dead one." Demonstrating the reasoning is worth more than the command list.
