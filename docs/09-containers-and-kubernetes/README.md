# Containers & Kubernetes

Containers package an application with its dependencies into a single immutable artifact that runs the same everywhere; Kubernetes schedules those containers across a fleet of machines, restarts them when they die, scales them under load, and routes traffic to them. Together they are the default deployment substrate for backend services — if the company you are interviewing at runs microservices, it almost certainly runs them on Kubernetes (or a managed flavor of it: EKS, GKE, AKS).

That is why containers/K8s questions now show up in almost every backend loop, not just SRE/platform interviews. A backend engineer is expected to write a production-grade Dockerfile, understand why their pod got OOMKilled, know what a readiness probe actually gates, and reason about zero-downtime deploys. You are rarely expected to administer a cluster — but you are expected to be a competent *tenant* of one, and senior/staff loops go further into failure modes, autoscaling, and multi-tenancy trade-offs.

Version note: this material targets **Kubernetes 1.36** (released April 2026; 1.37 lands August 2026) and **Docker Engine 28 / Compose v2**. Kubernetes ships three minor releases a year and supports each for ~14 months, so real clusters you meet will span roughly 1.32–1.36. Where a feature is version-gated (in-place pod resize GA in 1.35, native sidecar containers GA in 1.33, user namespaces GA in 1.36), it is called out inline.

Sub-files:

| File | Topic |
| --- | --- |
| `01-docker-fundamentals.md` | Images vs containers, layers and build cache, multi-stage builds, Dockerfile best practices, networking, volumes, security, Compose |
| `02-kubernetes-core.md` | Architecture, Pods/Deployments/Services/Ingress, ConfigMaps/Secrets, requests/limits and QoS, probes, HPA/VPA, rolling updates |
| `03-kubernetes-advanced.md` | StatefulSets, DaemonSets, Jobs, operators/CRDs, service mesh, NetworkPolicies, RBAC, PDBs, Karpenter, Gateway API, debugging |
| `04-interview-questions.md` | Graded interview questions (junior → senior → staff) with model answers |

Suggested reading order: Docker first — every Kubernetes concept assumes you know what an image and a container are. Then the Kubernetes core file, which covers ~80% of what backend loops ask. The advanced file matters most for senior/staff loops and for the debugging scenarios interviewers love. Finish with the questions, attempted cold before checking the answers.

!!! note
    If you are short on time: requests/limits + QoS, the three probe types, and the "pod keeps restarting" debugging workflow are the three highest-frequency topics in backend interviews. Know those cold.
