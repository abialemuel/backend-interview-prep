# Kubernetes Core

## What Kubernetes is

Kubernetes is a declarative container orchestrator: you submit YAML describing desired state ("3 replicas of this image, 256Mi each, behind a load balancer") to an API server, and a set of **controllers** run reconciliation loops that continuously drive actual state toward desired state. That reconciliation model is the key idea — the same loop that performs the initial deploy also replaces a pod when a node dies, which is why Kubernetes self-heals "for free." If a Terraform apply is a one-shot diff-and-converge, Kubernetes is that diff-and-converge running forever.

## Architecture

**Control plane** (managed for you on EKS/GKE/AKS):

- **kube-apiserver** — the front door. Everything (kubectl, controllers, kubelets) talks to the API server; nothing talks to etcd directly except it. Handles authn/authz/admission and validation.
- **etcd** — the strongly-consistent Raft-based key-value store holding all cluster state. Every object you `kubectl apply` lives here. Runs as 3 or 5 nodes for quorum; losing quorum makes the cluster read-only-ish (running pods keep running, but nothing can change).
- **kube-scheduler** — watches for pods with no node assigned, filters nodes that fit (resource requests, nodeSelector/affinity, taints/tolerations), scores the survivors, and binds the pod to the winner. It only *decides* placement — it doesn't start anything.
- **controller-manager** — hosts the built-in reconciliation loops: Deployment, ReplicaSet, Node, Job, EndpointSlice controllers and dozens more.

**Worker nodes:**

- **kubelet** — the node agent. Watches the API server for pods bound to its node, tells the container runtime (containerd via CRI — Docker-as-runtime was removed back in 1.24) to start them, runs the probes, reports status back.
- **kube-proxy** — programs iptables/IPVS rules so Service virtual IPs route to pod IPs. (Cilium-based clusters replace it with eBPF.)
- **Container runtime** — containerd or CRI-O actually runs the containers.

The flow for `kubectl apply -f deploy.yaml`: API server validates and persists to etcd → Deployment controller creates a ReplicaSet → ReplicaSet controller creates Pod objects → scheduler assigns each pod a node → that node's kubelet pulls the image and starts containers → kubelet reports status. Every arrow is a watch on the API server; components never talk to each other directly.

## Pods, Deployments, Services, Ingress

### Pods

The **Pod** is the smallest deployable unit: one or more containers that share a network namespace (same IP, talk over localhost) and can share volumes. Multi-container pods exist for tight coupling only — a log shipper or proxy alongside the app (the sidecar pattern; since 1.33 sidecars are natively supported as restartable init containers with `restartPolicy: Always`, fixing the old "job never exits because the sidecar is still running" problem). Pods are mortal and disposable: you almost never create bare pods, because nothing reschedules a bare pod when its node dies.

### Deployments

A **Deployment** manages ReplicaSets, which manage identical pods. You declare `replicas: 3` and an image; the controller keeps 3 running and orchestrates rollouts when the pod template changes (see rolling updates below). This is the workload type for stateless backend services — which is most of them.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: registry.example.com/api:abc123
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits: { memory: 256Mi }
          readinessProbe:
            httpGet: { path: /healthz/ready, port: 8080 }
          livenessProbe:
            httpGet: { path: /healthz/live, port: 8080 }
            periodSeconds: 10
            failureThreshold: 3
```

### Services

Pod IPs churn constantly, so a **Service** provides a stable virtual IP and DNS name (`api.prod.svc.cluster.local`) that load-balances across the pods matching its label selector. Types:

| Type | What you get | Use |
| --- | --- | --- |
| `ClusterIP` | Stable in-cluster VIP | Default; service-to-service traffic |
| `NodePort` | ClusterIP + a port (30000–32767) on every node | Rarely used directly; building block for LBs |
| `LoadBalancer` | NodePort + a cloud load balancer (ELB/NLB) | Exposing one service externally |
| `ExternalName` | DNS CNAME, no proxying | Aliasing an external dependency |
| Headless (`clusterIP: None`) | DNS returns individual pod IPs | StatefulSets, client-side LB (gRPC) |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }        # label selector — this is the entire coupling
  ports:
    - port: 80                  # the Service's port
      targetPort: 8080          # the container's port
```

The label selector is the whole mechanism — Services, Deployments, PDBs, and NetworkPolicies all find their pods by labels, never by name. An EndpointSlice controller watches for pods matching the selector that are Ready and keeps the backend list current. This is also the first thing to check when "the Service doesn't work": a label typo yields an empty endpoint list with no error anywhere.

Worth knowing for gRPC-heavy shops: a ClusterIP Service load-balances at *connection* level, so long-lived HTTP/2 connections pin to one pod — the fixes are a headless Service with client-side balancing, or a mesh/proxy that balances per-request.

### Ingress

A Service of type LoadBalancer per microservice is expensive and unstructured. An **Ingress** is L7 routing config — host- and path-based rules mapping to Services, plus TLS termination — executed by an ingress controller (nginx, ALB controller, Traefik) that you must install; the Ingress object does nothing alone.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt   # real-world Ingress = annotations everywhere
spec:
  ingressClassName: nginx
  tls: [{ hosts: [api.example.com], secretName: api-tls }]
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
```

The annotation sprawl is the tell: anything beyond host/path routing (timeouts, rate limits, canary weights) is vendor-specific annotations, which is a major reason the Ingress API is frozen and its successor, the **Gateway API**, exists (covered in the advanced file). Ingress remains what you'll meet in most existing clusters.

## ConfigMaps and Secrets

Both inject configuration, decoupling config from image (the same image runs in staging and prod):

- **ConfigMap** — non-sensitive key-values or whole config files. Consumed as env vars (`envFrom`) or mounted as files.
- **Secret** — same mechanics, but for credentials. Base64-encoded, **not encrypted** by default — base64 is encoding, not encryption, and interviewers love this distinction. Real hardening: enable encryption at rest for etcd (KMS provider), RBAC-restrict who can read Secrets, and prefer pulling from an external manager (AWS Secrets Manager / Vault) via External Secrets Operator or CSI driver, keeping the source of truth out of the cluster.

Env-var consumption requires a pod restart to pick up changes; mounted files update in place (with kubelet sync delay ~1 min), but the app must re-read them. A common pattern is a checksum-of-config annotation on the pod template so a config change triggers a rollout.

## Requests, limits, and QoS classes

Per-container `resources` drive both scheduling and runtime enforcement:

- **`requests`** — what the scheduler reserves. A node is eligible only if unallocated capacity covers the pod's requests. Requests also set the CPU weight (`cpu.weight`) under contention.
- **`limits`** — the runtime cap. CPU over limit → **throttled** (CFS quota; latency spikes, no kill). Memory over limit → **OOMKilled** (exit code 137) by the kernel, because memory can't be throttled.

QoS class is derived from how you set them:

| QoS | Condition | Eviction priority |
| --- | --- | --- |
| `Guaranteed` | requests == limits for CPU and memory, all containers | Evicted last |
| `Burstable` | At least one request set, not Guaranteed | Middle |
| `BestEffort` | No requests or limits at all | Evicted first under node pressure |

The pragmatic production stance (be ready to defend it): always set memory requests = memory limits (memory is incompressible; a burstable memory pod is an OOM lottery), set CPU requests honestly from observed usage, and consider *omitting* CPU limits — CPU throttling hurts tail latency and the request already guarantees fair share. Since **1.35, in-place pod resize is GA**: requests/limits can be changed on a running pod without restart, which removes the old "VPA must evict to resize" pain (1.36 extends this to pod-level resources in beta).

## Probes

| Probe | Question it answers | On failure |
| --- | --- | --- |
| **Startup** | Has the app finished booting? | Kubelet keeps waiting (up to `failureThreshold × periodSeconds`); other probes are disabled until it passes |
| **Readiness** | Can this pod serve traffic *right now*? | Pod is removed from Service endpoints — no restart |
| **Liveness** | Is the process irrecoverably wedged? | Container is restarted |

Classic mistakes to be able to name: (1) pointing the liveness probe at a handler that checks the database — a DB blip then restarts every pod in the fleet simultaneously, turning a degradation into an outage; liveness should check only in-process health, dependencies belong in readiness. (2) No startup probe on a slow-booting app plus a tight liveness probe — kubelet kills the app mid-boot forever: a self-inflicted CrashLoopBackOff. (3) No readiness probe at all — rolling updates route traffic to pods that aren't listening yet.

## Autoscaling: HPA and VPA

**HPA** (Horizontal Pod Autoscaler) adjusts `replicas` on a Deployment to hold a target metric — classically `averageUtilization: 70` of CPU *requests* (metrics-server required), or custom/external metrics via an adapter (KEDA is the de facto standard for event-driven scaling: queue depth, Kafka lag, even scale-to-zero). Scale-up is fast, scale-down is deliberately damped (stabilization window, default 5 min) to avoid flapping. `behavior:` tunes both directions.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

Utilization is measured against *requests*, so dishonest requests break autoscaling silently: requests set far above real usage mean the HPA sees "10% utilization" and never scales up under real load. CPU is a decent default signal for CPU-bound services; queue-consumer services should scale on lag/depth via KEDA instead, since CPU lags the actual backlog.

**VPA** (Vertical Pod Autoscaler) adjusts requests/limits instead, based on observed usage. Historically it had to evict pods to apply changes, which limited adoption to recommendation mode ("what should my requests be?"); with in-place resize GA since 1.35, in-place VPA updates are becoming the norm. Don't run HPA and VPA on the same metric (both on CPU) — they fight.

HPA scales pods; something must scale *nodes* when pods no longer fit — Cluster Autoscaler or Karpenter, covered in the advanced file.

## Rolling updates and rollbacks

Changing a Deployment's pod template triggers a rollout. Default strategy `RollingUpdate` with `maxSurge: 25%` / `maxUnavailable: 25%`: the controller creates a new ReplicaSet, scales it up and the old one down in steps, gated by readiness — a new pod must be Ready before the next old pod is taken down. The old ReplicaSets are kept (at zero replicas, `revisionHistoryLimit` deep), which is what makes rollback instant: it's just re-scaling a previous ReplicaSet.

```bash
kubectl set image deploy/api api=registry.example.com/api:def456
kubectl rollout status deploy/api          # blocks until complete or deadline
kubectl rollout history deploy/api
kubectl rollout undo deploy/api            # back to previous revision
kubectl rollout undo deploy/api --to-revision=3
```

Rolling-update correctness is not free — it requires the app to cooperate:

1. **Readiness probe** that only passes when the pod can actually serve.
2. **Graceful shutdown**: on SIGTERM, stop accepting new work, drain in-flight requests, exit before `terminationGracePeriodSeconds` (default 30s) or receive SIGKILL. Endpoint removal and SIGTERM race, so a `preStop` sleep of a few seconds is the standard trick to let kube-proxy/LB rules converge before the server stops listening.
3. **N and N+1 run concurrently** during the rollout, so schema changes must be backward-compatible (expand/contract migrations) and API changes versioned.

`strategy: Recreate` (kill all, then start all) exists for workloads that can't run two versions at once, at the cost of downtime. Canary and blue/green are not native Deployment features — you approximate canary with a second Deployment sharing a Service selector, or do it properly with Argo Rollouts / Flagger / mesh traffic-splitting.

!!! note
    "Design a zero-downtime deploy" is a standard interview scenario, and the expected answer is exactly the list above: rolling strategy + readiness gating + SIGTERM draining + preStop delay + PodDisruptionBudget + backward-compatible migrations. The K8s mechanics are necessary but not sufficient — the application-level contract is what candidates forget.

## Namespaces and the kubectl you actually use

Namespaces partition a cluster: names are scoped per namespace, RBAC and ResourceQuotas/LimitRanges attach to them, and cross-namespace DNS is `svc.namespace.svc.cluster.local`. Typical layout is one namespace per team or per app per environment.

```bash
kubectl get pods -n prod -o wide                 # -o yaml / -o jsonpath for detail
kubectl describe pod api-7d9f8-x2j4k -n prod     # events at the bottom — read them first
kubectl logs -f deploy/api -n prod --previous    # --previous = the crashed container's logs
kubectl exec -it api-7d9f8-x2j4k -- sh
kubectl port-forward svc/api 8080:80
kubectl top pods -n prod                         # live usage vs requests
kubectl apply -f k8s/ --dry-run=server           # validate without persisting
```
