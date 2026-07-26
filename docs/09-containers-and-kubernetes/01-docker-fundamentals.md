# Docker Fundamentals

## Containers vs VMs, images vs containers

A container is a normal Linux process isolated by kernel primitives: **namespaces** (what the process can *see* — its own PID tree, network stack, mounts, hostname, users) and **cgroups** (what it can *use* — CPU, memory, IO caps). There is no guest OS and no hypervisor; every container on a host shares the host kernel. That is why containers start in milliseconds and pack densely where VMs take seconds-to-minutes and carry a full OS each — and also why container isolation is weaker than VM isolation (a kernel exploit escapes all containers on the host).

The image/container split is the class/instance split:

- An **image** is an immutable, layered filesystem snapshot plus metadata (entrypoint, env, exposed ports). It is content-addressed — every layer and the manifest are identified by SHA-256 digest — and stored in a registry (Docker Hub, ECR, GHCR).
- A **container** is a running (or stopped) instance of an image: the image's read-only layers plus one thin writable layer on top, plus a live process.

You can run fifty containers from one image; they share the read-only layers on disk and each get their own writable layer. Anything written to the writable layer dies with the container — persistent data belongs in volumes.

## Tags, digests, and the OCI ecosystem

An image reference is `registry/repository:tag` — `registry.example.com/payments/api:1.4.2`. Two details interviewers use to separate people who have shipped from people who have read tutorials:

- **Tags are mutable pointers.** `api:1.4.2` and especially `api:latest` can be re-pushed to point at different bytes tomorrow. A **digest** reference (`api@sha256:9f8e...`) is immutable — it names exact content. Production deploys should resolve to digests (most CD systems and `kubectl` support it; Kubernetes' `imagePullPolicy` interacts with this: `IfNotPresent` on a mutable tag means different nodes can run different bytes under the same tag).
- **"Docker image" is really an OCI image.** The Open Container Initiative standardized the image format, distribution protocol, and runtime spec — which is why images built by Docker, Buildah, or Bazel run identically under containerd, CRI-O, or Podman, and why Kubernetes dropped the Docker daemon (1.24) with zero image compatibility impact. The runtime stack is: orchestrator → containerd (lifecycle) → runc (actually creates the namespaced/cgrouped process).

## Layers and the build cache

Each Dockerfile instruction that changes the filesystem (`FROM`, `RUN`, `COPY`, `ADD`) produces a layer. Layers stack via a union filesystem (overlay2): the container sees the merged view. Two properties follow:

1. **Caching**: at build time, Docker (BuildKit, the default builder since Engine 23) reuses a cached layer if the instruction and its inputs are unchanged. For `COPY`, "inputs" means a checksum of the copied files. The first cache miss invalidates every subsequent layer.
2. **Deletion doesn't shrink**: a `RUN rm -rf /big-thing` in a later layer hides the files but the earlier layer still ships. Cleanup must happen in the *same* `RUN` as the thing that created the mess.

The cache rule dictates Dockerfile ordering: put the things that change least often first. The classic pattern for a Go/Node/Python service is to copy the dependency manifest alone, install dependencies, and only then copy the source — so editing application code doesn't re-download the world:

```dockerfile
COPY go.mod go.sum ./
RUN go mod download          # cached until go.mod/go.sum change
COPY . .                     # source changes only invalidate from here
RUN go build -o /app ./cmd/server
```

## Multi-stage builds

A multi-stage build uses several `FROM` blocks in one Dockerfile; only the final stage ships. The build stage carries compilers, SDKs, and dev dependencies; the runtime stage copies out just the artifact. This is the single biggest lever for image size and attack surface:

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /bin/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /bin/server /server
USER nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

A `golang:1.24` image is ~800 MB; the distroless result is ~15 MB. Smaller images pull faster (matters for autoscaling cold starts), have fewer CVEs to patch, and give an attacker who lands in the container no shell or package manager to work with. `COPY --from=` can also pull from a named external image (`COPY --from=busybox:latest /bin/busybox ...`).

Stages can also target different purposes from one Dockerfile — `docker build --target build` stops at the build stage, which is a common trick for a dev image (with toolchain and test deps) and a prod image (minimal) sharing all the dependency layers:

```dockerfile
FROM golang:1.24 AS build
# ... deps + source ...

FROM build AS test
RUN go vet ./... && go test ./...

FROM gcr.io/distroless/static-debian12:nonroot AS prod
COPY --from=build /bin/server /server
```

CI builds `--target test` for the gate and `--target prod` for the artifact; the shared layers are built once. For interpreted languages the same structure applies — build stage installs dev dependencies and compiles assets, runtime stage copies `node_modules --omit=dev` or a Python venv.

## Dockerfile best practices

| Practice | Why |
| --- | --- |
| Pin base images (`golang:1.24-bookworm`, better: `@sha256:...`) | `latest` makes builds non-reproducible; a base bump lands silently |
| Order instructions least- to most-frequently changing | Maximizes cache hits; dependency install shouldn't rerun on a code edit |
| One `RUN` for install + cleanup (`apt-get update && apt-get install -y --no-install-recommends x && rm -rf /var/lib/apt/lists/*`) | Deleting in a later layer doesn't reclaim space |
| Use `.dockerignore` (`.git`, `node_modules`, secrets, build output) | Shrinks build context, prevents cache busts and secret leaks via `COPY . .` |
| Prefer `COPY` over `ADD` | `ADD` auto-extracts archives and fetches URLs — surprising behavior; use `curl` explicitly if needed |
| `ENTRYPOINT ["exec","form"]` not shell form | Shell form wraps the process in `/bin/sh -c`, so PID 1 is the shell and your app never receives SIGTERM — graceful shutdown breaks |
| Never bake secrets via `ENV`/`ARG`/`COPY` | They persist in layers/metadata (`docker history` shows ARGs); use `RUN --mount=type=secret` for build-time and env/files injected at runtime |
| Add a `HEALTHCHECK` (for plain Docker; K8s uses probes instead) | Lets the daemon/orchestrator distinguish "running" from "working" |

`ENTRYPOINT` vs `CMD`: `ENTRYPOINT` is the fixed executable, `CMD` provides default arguments that `docker run <image> <args>` overrides. The conventional pairing is `ENTRYPOINT ["/server"]` + `CMD ["--config=/etc/app.yaml"]`.

BuildKit extras worth naming in an interview: `RUN --mount=type=cache,target=/root/.cache/go-build` persists a package/compiler cache across builds without baking it into a layer; `RUN --mount=type=secret,id=npmrc` exposes a secret during one instruction only; `docker buildx build --platform linux/amd64,linux/arm64` produces multi-arch images — relevant now that Graviton/ARM nodes are common cost optimizations.

## Networking modes

| Mode | What it does | When |
| --- | --- | --- |
| `bridge` (default) | Container gets a private IP on a virtual bridge; outbound traffic is NAT'd; inbound requires `-p host:container` port publishing | Default single-host case |
| user-defined bridge | Like bridge, plus built-in DNS: containers resolve each other by name | Multi-container apps on one host; what Compose creates per project |
| `host` | No network namespace — container shares the host's stack, no NAT, no port mapping | Max network performance, or tools that need raw host networking; ports can collide |
| `none` | Loopback only | Batch jobs, security-sensitive workloads |
| `overlay` | VXLAN network spanning multiple hosts | Docker Swarm; in Kubernetes a CNI plugin (Cilium, Calico) fills this role instead |

The interview-worthy detail: on the default bridge, containers must reach each other by IP; on a *user-defined* bridge Docker runs an embedded DNS server so `db:5432` just works — which is why Compose services address each other by service name:

```bash
docker network create appnet
docker run -d --name db --network appnet postgres:17
docker run -d --name api --network appnet -p 8080:8080 api:abc123
# inside api: postgres://db:5432 resolves via Docker's embedded DNS
```

Port publishing (`-p 8080:8080`) only matters for traffic entering from *outside* the Docker host; container-to-container traffic on the same network uses container ports directly. And `-p 127.0.0.1:5432:5432` vs `-p 5432:5432` is a real security distinction — the latter binds on all host interfaces and, on default Docker iptables handling, can bypass host firewall expectations.

## Volumes and persistence

Three ways to get data in/out of the writable-layer trap:

- **Named volumes** (`docker volume create pgdata; -v pgdata:/var/lib/postgresql/data`) — managed by Docker under `/var/lib/docker/volumes`, survive container removal, the right choice for databases and anything stateful.
- **Bind mounts** (`-v $(pwd)/src:/app/src`) — map a host path directly; the workhorse of local dev hot-reload; couples the container to host layout, so avoid in production.
- **tmpfs** (`--tmpfs /scratch`) — RAM-backed, gone on stop; scratch space and secrets you don't want on disk.

Kubernetes generalizes this into `volumes`/`PersistentVolumeClaims`, but the mental model — "container filesystem is disposable, state lives in a mount" — is identical.

## Resource limits and PID 1

Docker exposes the same cgroup controls Kubernetes builds on: `--memory 512m` (exceed it and the kernel OOM-kills the process — exit code 137), `--cpus 1.5` (CFS quota — exceed it and you're throttled, not killed), `--pids-limit` (fork-bomb protection). Two process-model facts that explain a lot of production weirdness:

- Your entrypoint runs as **PID 1**, which has special semantics: default signal dispositions don't apply (an unhandled SIGTERM does nothing rather than terminating), and it is responsible for reaping zombie children. If your app doesn't handle both, run it under a minimal init: `docker run --init`, or `tini` as the entrypoint. This is the root cause of half of all "my container takes exactly 10/30 seconds to stop" reports — the app never saw SIGTERM, and the runtime SIGKILLed it at the timeout.
- Memory-managed runtimes size themselves from what they can see. Older JVMs read host memory instead of the cgroup limit and got OOMKilled "mysteriously"; modern JVMs are container-aware, and Go needs `GOMEMLIMIT`/`GOMAXPROCS` (or automaxprocs) to align with limits. The same issue resurfaces in Kubernetes with requests/limits.

## Security

- **Run as non-root.** The default user in most images is root; combined with a container-escape or a mounted host path, that's root on the node. Create a user (`USER app`) or use distroless `:nonroot` variants. Kubernetes can enforce this (`runAsNonRoot: true` in the securityContext; the Pod Security "restricted" profile requires it).
- **Distroless / minimal bases.** `gcr.io/distroless/*` images contain your binary, CA certs, and tzdata — no shell, no package manager, single-digit CVE counts. Alpine is the middle ground (musl libc occasionally bites native dependencies). Chainguard/Wolfi images are the same idea with faster CVE patching.
- **Drop capabilities, read-only rootfs.** `--cap-drop ALL --cap-add NET_BIND_SERVICE`, `--read-only` with explicit tmpfs mounts. Never `--privileged` in production — it disables essentially all isolation.
- **Scan images in CI** (Trivy, Grype) and fail the build on critical CVEs; sign images (cosign) and verify at admission if the org requires provenance.
- **Don't mount the Docker socket** (`/var/run/docker.sock`) into a container unless it is explicitly a build/infra tool — the socket is unauthenticated root on the host.

A hardened `docker run` pulls these together — the same posture you later express in a Kubernetes `securityContext`:

```bash
docker run -d --name api \
  --read-only --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --user 10001:10001 \
  --memory 512m --cpus 1 --pids-limit 256 \
  api:abc123
```

!!! warning
    "Our containers are isolated" is not a security boundary claim you can make in an interview without caveats. Containers share the host kernel; treat container escape as possible and layer defenses: non-root, seccomp/AppArmor defaults, minimal images, and (since K8s 1.36 made user namespaces GA) UID remapping so root-in-container is unprivileged on the host.

## Docker Compose

Compose (v2, the `docker compose` plugin — the Python `docker-compose` v1 is long dead) declares a multi-container app in one YAML file. It is the standard for local dev environments and lightweight single-host deployments; it is *not* an orchestrator — no multi-node scheduling, no self-healing beyond `restart:`, no rolling deploys.

```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://app:app@db:5432/app
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10
volumes:
  pgdata:
```

Details interviewers probe: `depends_on` alone only orders *startup*, not readiness — the `condition: service_healthy` form plus a healthcheck is what actually waits for Postgres to accept connections. Services share a per-project network and resolve each other by service name. `docker compose watch` (or bind mounts) gives hot reload in dev.

Other Compose features worth knowing: **override files** (`compose.override.yaml` is merged automatically — commit the base, keep dev-only bind mounts and debug ports in the override; `-f` stacks arbitrary files for CI vs local), **profiles** (`profiles: ["debug"]` on a service keeps pgadmin/jaeger out of the default `up`), and `env_file` + variable interpolation (`${VAR:-default}`) for per-developer settings. The limits are the answer to "why not Compose in production": one host, no self-healing beyond `restart: unless-stopped`, no rolling deploys, no secret management beyond files. For a single-box internal tool it's fine; for anything with availability requirements you want an orchestrator.

## Commands you should be fluent in

```bash
docker build -t api:$(git rev-parse --short HEAD) .
docker run -d --name api -p 8080:8080 --memory 512m --cpus 1 api:abc123
docker logs -f api                 # stdout/stderr — log to stdout, not files
docker exec -it api sh             # shell into a running container (if it has one)
docker inspect api                 # full config/state JSON
docker stats                       # live cgroup usage
docker history api:abc123          # layer-by-layer size — find the bloat
docker system prune -af --volumes  # reclaim disk (careful with --volumes)
```

The habit worth stating out loud in interviews: containers log to stdout/stderr and the platform (Docker log driver, or in K8s the node agent) ships them — never write log files inside the container.
