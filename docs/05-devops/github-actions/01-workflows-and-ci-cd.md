# Workflows and CI/CD

## The .github/workflows model

A workflow is a YAML file under `.github/workflows/*.yml`. The file defines triggers (`on:`), one or more **jobs**, and each job defines a runner (`runs-on:`) and a list of **steps**. Steps either `run` shell commands or `uses` an action. GitHub parses the file and dispatches jobs to runners when a trigger fires.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    paths:
      - "src/**"
      - "tests/**"
      - ".github/workflows/ci.yml"
  schedule:
    - cron: "0 2 * * *"
  workflow_dispatch:
    inputs:
      debug:
        description: "Enable debug logging"
        type: boolean
        default: false

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25"
      - run: go test ./... -race -coverprofile=cover.out
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: cover.out
```

## Workflow syntax — `on:`

`on:` accepts one or more events, each optionally filtered:

```yaml
on:
  push:
    branches: [main, "release/*"]
    tags: ["v*"]
    paths: ["src/**", "tests/**"]
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]
  schedule:
    - cron: "0 2 * * *"
  workflow_dispatch:           # manual trigger from the Actions UI
    inputs:
      environment:
        type: choice
        options: [staging, prod]
  repository_dispatch:          # triggered by an external POST to the dispatch API
  workflow_call:                # makes the workflow reusable from other workflows
  issues:
    types: [opened]
```

`paths` and `paths-ignore` skip the workflow when no matching file changed. `branches`/`tags` filter by ref. `types` selects the activity types for an event (e.g. PR opened vs closed). `schedule` uses standard cron in UTC. `workflow_dispatch` adds a "Run workflow" button and typed inputs.

## Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    needs: [lint]                       # runs after lint
    if: github.event_name == 'push'
    environment: staging                 # gated by environment protection rules
    timeout-minutes: 30
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        go: ["1.24", "1.25"]
        exclude:
          - os: macos-latest
            go: "1.24"
      fail-fast: false
      max-parallel: 4
    steps: [...]
```

- `runs-on` — a runner label (`ubuntu-latest`, `macos-latest`, `windows-latest`) or a self-hosted label.
- `needs` — declares dependencies; jobs form a DAG and a job only starts when its `needs` complete (or `always()`/`failure()` conditionals override).
- `if` — a boolean expression; common values are `github.event_name == 'pull_request'`, `always()`, `failure()`, `success()`.
- `environment` — places the job in a named environment, triggering required reviewers, branch protection, and environment secrets.
- `timeout-minutes` — kills the job after N minutes; set it to avoid hung runners burning minutes.
- `strategy.matrix` — fans the job out across combinations of values; each combination is a separate job with its own `matrix.os`/`matrix.go` context.
- `fail-fast` — when true (the default), the first failing matrix leg cancels the others; set false if you want all legs to complete.
- `max-parallel` — caps concurrent matrix legs.
- `concurrency` — at the workflow or job level, groups runs and cancels obsolete ones (see below).

## Steps

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4
    with:
      fetch-depth: 0          # full history for tag/commit-range tools
      persist-credentials: false

  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: "22"
      cache: "npm"            # built-in cache for npm

  - name: Install
    run: npm ci
    working-directory: packages/api
    env:
      NODE_OPTIONS: "--max-old-space-size=4096"

  - name: Test
    run: npm test -- --reporter=dot
    continue-on-error: true   # let the workflow continue; check status in a later step
```

Step fields: `uses` (an action reference, with optional `with` inputs), `run` (shell command(s), with optional `shell` and `working-directory`), `env` (per-step env), `name`, `if`, `continue-on-error`, `timeout-minutes`. `actions/checkout` is the universal first step; `fetch-depth: 0` is needed for tools that read history (semantic-release, changesets, coverage diff). `persist-credentials: false` prevents the workflow's `GITHUB_TOKEN` from being persisted in the local git config, which is a hygiene win for supply-chain safety.

## Runners

**GitHub-hosted runners** are VMs GitHub maintains. Labels: `ubuntu-latest` (Ubuntu 24.04 since early 2025), `windows-latest`, `macos-latest` (Apple Silicon). Standard Linux runners are 4 vCPU/16 GB; **larger runners** (paid plans) scale to 96 vCPUs, add GPU options, and — notably — **arm64** Linux runners, which matter for building multi-arch container images without QEMU emulation (arm64 runners are even free for public repos). They start clean, are patched by GitHub, and are ephemeral — each job gets a fresh VM. Good default; the trade-off is cost (minutes are billed) and no access to private networks without extra tunneling.

**Self-hosted runners** are machines you own and register with GitHub. Labels are arbitrary; you can group them into **runner groups** scoped to an org or enterprise and restrict which repos can use a group. Trade-offs: you control hardware, network, and cost (no per-minute billing); you must patch, scale, and secure them. A self-hosted runner that runs untrusted PRs is a serious security boundary — never run public-fork PRs on a self-hosted runner that has access to prod secrets; use ephemeral runners (auto-scaler, ARC — Actions Runner Controller on Kubernetes) that are destroyed after each job.

## Secrets

Secrets are encrypted values scoped to a repo, an environment, or an organization. They are masked in logs (`::add-mask::`), never echoed in `env:` to subprocesses in plaintext beyond what the runner needs, and are not passed to workflows triggered by pull requests from forks unless explicitly approved.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production          # required-reviewer gate + environment secrets
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.PROD_API_KEY }}
```

Repo secrets are available to every workflow in the repo. Environment secrets are available only to jobs that declare `environment: <name>` and pass the environment's protection rules (required reviewers, wait timer, branch restriction). Use environment secrets for prod credentials and repo secrets for non-sensitive shared values. Never `echo` a secret — masking is by exact-string match, and formatting or truncating can leak it.

### OIDC for cloud auth — the default, not the alternative

Long-lived cloud keys in repo secrets are a legacy pattern; in 2026 an interviewer expects OIDC as your first answer. GitHub Actions issues an OIDC token signed by GitHub's provider; the cloud trusts it via `AssumeRoleWithWebIdentity` (AWS) or Workload Identity Federation (GCP/Azure). The job gets credentials valid for about an hour, scoped to the role, with no key stored in a secret — nothing to leak, nothing to rotate.

```yaml
permissions:
  id-token: write        # required to mint an OIDC token
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-tf
          aws-region: us-east-1
      - run: terraform apply -auto-approve
```

On the AWS side, the role's trust policy allows `sts:AssumeRoleWithWebIdentity` from GitHub's OIDC provider with a condition matching the token's `sub` claim (org/repo/branch or environment). No `AWS_ACCESS_KEY_ID` in a secret; rotation is automatic; blast radius is the role's policy. Scope the `sub` condition tightly — `repo:acme/api:environment:production` is much stronger than `repo:acme/*` — and use this for every cloud-authenticated job.

## Environments and protection rules

An environment is a named deployment target (e.g. `staging`, `production`) with optional protection rules: required reviewers (a human must approve before the job starts), wait timer (e.g. 5 minutes to allow rollback), branch restriction (only `main` can deploy to `production`), and environment secrets. Jobs that declare `environment: production` are gated until the rules pass.

```yaml
jobs:
  deploy-prod:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - run: ./deploy.sh prod
```

The `url` surfaces a deployment link in the GitHub UI and the deployment history. Required reviewers turn the job into an approval gate — the workflow pauses until a reviewer clicks approve.

## Artifacts and caching

**Artifacts** are files produced by a workflow and stored for download later (logs, coverage reports, build binaries). Use `actions/upload-artifact` and `actions/download-artifact` — **v4 only**: v3 and earlier were shut down in early 2025. v4 is up to 10x faster, makes each artifact immediately downloadable as soon as its job uploads it (no waiting for the workflow to finish), and scopes artifacts to the job, so names must be unique per run (use a matrix suffix like `build-${{ matrix.os }}` and merge with the `merge-multiple` download input). Artifacts expire (default 90 days, configurable).

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-${{ github.sha }}
    path: dist/
    retention-days: 7
```

**Cache** is for reusing dependencies across runs to speed them up. `actions/cache` restores by key and saves on miss; `restore-keys` provides fallback prefix matches.

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

Cache keys should be specific enough to invalidate when inputs change (`hashFiles('**/go.sum')`) but broad enough to hit on repeat runs. `restore-keys` lets a miss fall back to a partial key (same OS, older lockfile) and still save the new one. Many setup actions (`setup-node`, `setup-go`, `setup-python`) have a built-in `cache: <pm>` input that wraps `actions/cache` for the common case — prefer that over hand-rolling.

Use `actions/cache@v4` — the v1–v3 cache actions were shut down in early 2025 along with the legacy cache backend. v4 handles large caches better and supports cross-OS restoration. For monorepos, scope cache keys by workspace path to avoid collisions.

Artifacts vs cache: artifacts are outputs you keep for humans or downstream jobs; cache is an internal accelerator for dependencies. Don't use cache as persistent storage — entries can be evicted.

## Reusable workflows

A workflow that defines `on: workflow_call` can be called by other workflows, with typed inputs and secrets:

```yaml
# .github/workflows/ci.yml in repo acme/ci-library
on:
  workflow_call:
    inputs:
      go-version:
        type: string
        required: true
    secrets:
      CODECOV_TOKEN:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "${{ inputs.go-version }}" }
      - run: go test ./...
```

```yaml
# .github/workflows/test.yml in repo acme/api
jobs:
  ci:
    uses: acme/ci-library/.github/workflows/ci.yml@v1
    with: { go-version: "1.25" }
    secrets: { CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN } } }
```

Reusable workflows let an org standardize CI across many repos and patch it in one place. They support matrix, outputs, and secret inheritance.

## Composite actions

A composite action packages multiple steps into a single action defined in `action.yml`:

```yaml
# action.yml
name: "Setup and Test"
description: "Install deps and run tests"
inputs:
  go-version:
    description: "Go version"
    required: true
    default: "1.25"
runs:
  using: "composite"
  steps:
    - uses: actions/setup-go@v5
      with: { go-version: "${{ inputs.go-version }}" }
    - shell: bash
      run: go install gotest.tools/gotestsum@latest
    - shell: bash
      run: gotestsum --format=short ./...
```

```yaml
- uses: acme/actions/setup-and-test@v1
  with: { go-version: "1.25" }
```

Use composite actions for steps repeated within one repo or org; use reusable workflows for whole job-level pipelines. Composite actions are simpler (no runner, no job lifecycle); reusable workflows are more powerful (matrix, needs, environment).

## Workflow commands

GitHub Actions recognizes special `::`-prefixed lines that interact with the runner:

```bash
echo "::add-mask::$SECRET_VALUE"
echo "result=abc123" >> "$GITHUB_OUTPUT"
echo "EXTRA_PATH=$HOME/.local/bin" >> "$GITHUB_ENV"
echo "$HOME/.local/bin" >> "$GITHUB_PATH"
echo "::group::Setup"
echo "::endgroup::"
echo "::set-output name=foo::bar"   # deprecated, use $GITHUB_OUTPUT
echo "::error file=app.js,line=10::Missing semicolon"
echo "::warning file=app.js,line=10::Unused variable"
echo "::debug::Detailed debug info"
echo "::stop-commands::pause-logging"   # disable command parsing until end token
```

- `$GITHUB_OUTPUT` — write `key=value` lines; the step's outputs are then readable as `${{ steps.<id>.outputs.<key> }}`.
- `$GITHUB_ENV` — write `key=value` lines to set env for subsequent steps in the same job.
- `$GITHUB_PATH` — prepend to PATH for subsequent steps.
- `::add-mask::` — mask a value in logs.
- `::error::` / `::warning::` / `::notice::` — annotate the run with file/line-attached messages.
- `::group::`/`::endgroup::` — collapsible log sections.

The legacy `::set-output` form is deprecated; use `$GITHUB_OUTPUT`.

## Permissions and the GITHUB_TOKEN

Every workflow gets an auto-generated `GITHUB_TOKEN` scoped to the repo. Default permissions historically were permissive (`contents: write`, `pull-requests: write`, etc.); best practice is to default-deny at the org/repo level and grant per-workflow:

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write          # for OIDC
  packages: write
```

Least-privilege matters: a workflow that only needs to read code and write a PR comment should not have `contents: write`. Set the org default to read-only and grant per workflow. The token expires at the end of the job.

## Concurrency

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

A concurrency group ensures only one run per `group` value is active at a time. `cancel-in-progress: true` cancels older runs in the group — ideal for CI on a branch where only the latest commit matters. `false` queues runs — ideal for deploys where you want each commit to deploy in order. Note the historical gotcha: by default a group holds only one *pending* run, so intermediate queued runs are superseded; since 2026 you can add `queue: max` to keep a real queue (up to 100 runs) when every run must execute. Use a branch/ref-based group so concurrent branches do not cancel each other.

## Merge queues

Branch protection with required checks validates a PR against *its own* base — not against `main` as it will be after other PRs land, so two individually-green PRs can break `main` together. A **merge queue** fixes this: instead of merging directly, approved PRs enter a queue; GitHub creates temporary branches with each PR stacked on the previous queue entries, fires the `merge_group` trigger, runs CI against that speculative future state of `main`, and merges only if it passes. Your CI workflow must include `merge_group:` in `on:` or queued PRs will hang waiting for checks. This is how high-velocity repos keep `main` always green without serializing merges behind full CI latency; it replaces the "update branch, wait, retry" dance and third-party bots like Bors.

## Deployment workflows

A typical deploy pipeline:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [checkout, build, push image]

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    concurrency: deploy-staging
    steps: [deploy to staging]

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    concurrency: deploy-prod
    steps: [deploy to prod]
```

The `environment: production` step gates on required reviewers; `concurrency: deploy-prod` ensures one prod deploy at a time. Long-running deploys should set `timeout-minutes` and consider splitting the deploy from a smoke-test job via `needs`.

**Push vs pull CD (GitOps):** deploying *from* Actions is push-based — the pipeline holds cloud credentials and imperatively rolls out. The GitOps alternative (ArgoCD/Flux) is pull-based: Actions only builds, signs, and pushes the image and updates a manifest repo; a controller inside the cluster reconciles continuously. Pull-based wins for Kubernetes at scale (no cluster credentials in CI, automatic drift correction, easy multi-cluster); push-based from Actions stays simpler for VMs, serverless, and small setups. Knowing where you'd draw that line is a common senior interview probe.

## Security and the software supply chain

- **Pin actions by SHA, not tag.** `actions/checkout@v4` is mutable; a compromised tag can swap the code under you. This is not theoretical: the March 2025 **tj-actions/changed-files** compromise retro-pointed *every* version tag at malicious code that dumped CI secrets from runner memory, hitting thousands of repos — the canonical case study interviewers now cite. Pin to a SHA (`actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4`) and keep Dependabot on to bump. GitHub's longer-term fix is **immutable actions** — actions published as sealed OCI packages in GHCR whose version, once published, cannot be re-pointed.
- **Dependabot for actions.** Enable the `github-actions` ecosystem in dependabot.yml to get PRs that bump action versions, including SHA-pinned ones.
- **OIDC over stored credentials.** Use `id-token: write` + cloud workload identity instead of long-lived AWS/GCP keys in secrets.
- **Least-privilege permissions.** Default-deny at org level; grant per workflow.
- **Beware script injection via untrusted contexts.** Interpolating attacker-controlled values (`${{ github.event.pull_request.title }}`, issue bodies, branch names) directly into `run:` executes them in the shell. Pass them through `env:` and quote instead. `pull_request_target` is the sharpest edge: it runs with secrets in the *base* repo context against fork code — never check out and execute fork code under it. Static analyzers like **zizmor** and CodeQL's workflow queries catch these patterns.
- **Provenance and attestations.** `actions/attest-build-provenance` generates signed SLSA build provenance for artifacts and container images ("this binary was built by this workflow from this commit"), verifiable with `gh attestation verify`. Increasingly expected for released artifacts.
- **Don't run untrusted PRs on self-hosted runners** with prod access. For public repos, fork PRs do not get access to secrets unless a maintainer approves the run.
- **Secret scanning.** Enable GitHub secret scanning and push protection to catch leaked tokens before they land.
- **Treat the workflow `env:` like code** — a secret in `env:` is visible to every step, including ones that run third-party actions. Scope secrets to the step that needs them.