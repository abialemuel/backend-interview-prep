# GitHub Actions Interview Questions

## Easy

### Q1: What triggers a workflow and how do you filter it?
**Answer:** The `on:` key lists one or more events: `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual), `repository_dispatch` (external API call), `workflow_call` (reusable), `issues`, etc. Each event accepts filters: `branches`/`tags` to scope by ref, `paths`/`paths-ignore` to skip when no matching file changed, and `types` for activity types (e.g. PR opened vs closed). `schedule` uses standard cron in UTC. Multiple events combine under one `on:`. A common pattern is `push` to `main` plus `pull_request` scoped to `src/**`.

### Q2: Jobs vs steps?
**Answer:** A **job** runs on a single runner and contains a list of steps; jobs declare `runs-on`, can depend on each other via `needs`, and form a DAG. A **step** is either a `run` shell command or a `uses` action invocation, executed in order within a job. Steps share the same runner filesystem and env, so a file written in one step is readable in the next. Jobs are the unit of parallelism and fan-out (matrix); steps are the unit of sequential work.

### Q3: What does `needs` do?
**Answer:** `needs: [lint, build]` declares that a job depends on those jobs; the dependent job only starts when all `needs` complete successfully, forming a DAG. Combined with `if: always()` or `if: failure()` you can run a job regardless of upstream status (e.g. a cleanup job that runs even if tests failed). `needs` also lets a downstream job reference upstream outputs via `${{ needs.build.outputs.image }}`. Without `needs`, jobs run in parallel.

### Q4: What is the matrix strategy?
**Answer:** `strategy.matrix` fans a single job out across combinations of values — `os: [ubuntu-latest, macos-latest]` and `go: ["1.21", "1.22"]` produce four jobs, each with `matrix.os` and `matrix.go` in scope. `exclude` removes specific combinations; `include` adds extra combinations or extra vars to existing ones. `fail-fast: false` lets all legs complete instead of canceling on the first failure; `max-parallel` caps concurrent legs. Use matrix to test across language/runtime/OS combinations in one declaration.

### Q5: GitHub-hosted vs self-hosted runners?
**Answer:** GitHub-hosted runners are VMs GitHub maintains (`ubuntu-latest`, `windows-latest`, `macos-latest`), patched and ephemeral — each job gets a fresh VM, billed per minute. Good default; no access to private networks without tunneling. Self-hosted runners are machines you own, registered with arbitrary labels and grouped into runner groups scoped to org/enterprise; you control hardware, network, and cost but must patch, scale, and secure them. Never run untrusted public-fork PRs on a self-hosted runner with prod access — use ephemeral self-hosted runners (ARC on Kubernetes) destroyed after each job.

### Q6: How are secrets handled and masked?
**Answer:** Secrets are encrypted values scoped to repo, environment, or organization. They are masked in logs by exact-string match and not passed to fork-PR workflows unless approved. Reference them as `${{ secrets.NAME }}` in `env:` or action `with:` inputs. Never `echo` a secret — masking is by exact string, and formatting or truncating can leak it. Environment secrets require the job to declare `environment: <name>` and pass its protection rules, so prod credentials can be gated behind required reviewers. For cloud auth, prefer OIDC over stored keys.

## Medium

### Q7: Environment secrets vs repo secrets?
**Answer:** Repo secrets are available to every workflow in the repo; environment secrets are available only to jobs that declare `environment: <name>` and pass that environment's protection rules (required reviewers, wait timer, branch restriction). Use environment secrets for prod credentials so they are gated behind an approval step, and repo secrets for non-sensitive shared values like a Slack webhook. Environments also give you deployment history and a `url` shown in the GitHub UI. The gate is real: a job with `environment: production` pauses until a reviewer approves.

### Q8: OIDC for cloud auth — why is it better than stored keys?
**Answer:** With OIDC, GitHub Actions mints a short-lived signed token; the cloud trusts it via `AssumeRoleWithWebIdentity` (AWS) or Workload Identity Federation (GCP), returning credentials valid for one hour scoped to the role. No long-lived access key is stored in a secret, so there is nothing to leak or rotate manually; blast radius is the role's policy; rotation is automatic. The job declares `permissions: id-token: write` and uses `aws-actions/configure-aws-credentials` or the GCP equivalent. The cloud-side trust policy condition matches your org/repo/branch so only your workflows can assume the role.

### Q9: Caching strategies and cache keys?
**Answer:** `actions/cache` restores by `key` and saves on miss. The key should be specific enough to invalidate when inputs change (`hashFiles('**/go.sum')`, `hashFiles('**/package-lock.json')`) but broad enough to hit on repeat runs; `restore-keys` provides fallback prefix matches so a miss on the exact key can still restore a near-miss cache and then save the new one. Scope keys by OS and workspace path in monorepos to avoid collisions. Many setup actions (`setup-node`, `setup-go`, `setup-python`) have a built-in `cache: <pm>` input that wraps `actions/cache` for the common case — prefer that over hand-rolling.

### Q10: Artifacts vs cache?
**Answer:** Artifacts are outputs produced by a workflow and stored for humans or downstream jobs to download — build binaries, coverage reports, logs. They expire (default 90 days, configurable via `retention-days`) and are not meant as long-term storage. Cache is an internal accelerator for dependencies, keyed by you, restored at the start of future runs to skip re-installation. Don't use cache as persistent storage — entries can be evicted by size limits. Use `upload-artifact`/`download-artifact` for outputs and `actions/cache` (or setup-action built-ins) for dependency reuse.

### Q11: Reusable workflows vs composite actions?
**Answer:** A **reusable workflow** is a whole workflow file with `on: workflow_call` that another workflow invokes via `uses: org/repo/.github/workflows/x.yml@ref`, passing typed inputs and secrets; it can have its own jobs, matrix, needs, environments, and outputs. Use it to standardize an entire CI pipeline across many repos. A **composite action** packages multiple steps into a single `action.yml` with `runs: composite`; it runs on the caller's job, no separate runner. Use it for repeated step sequences within a repo or org. Reusable workflows are more powerful; composite actions are simpler and lighter-weight.

### Q12: `$GITHUB_OUTPUT`, `$GITHUB_ENV`, `$GITHUB_PATH`?
**Answer:** `$GITHUB_OUTPUT` is a file you append `key=value` lines to; the step's outputs are then readable as `${{ steps.<id>.outputs.<key> }}` in later steps. `$GITHUB_ENV` does the same for environment variables — set in one step, available in subsequent steps of the same job. `$GITHUB_PATH` prepends entries to PATH for subsequent steps. These replaced the deprecated `::set-output` command. They are the standard way to pass data between steps within a job; for cross-job data use job outputs or artifacts.

### Q13: Least-privilege permissions — why and how?
**Answer:** Every workflow gets an auto-generated `GITHUB_TOKEN` whose default permissions historically were permissive (`contents: write`, etc.). A workflow that only needs to read code and write a PR comment should not have `contents: write`, so default-deny at the org/repo level and grant per workflow with an explicit `permissions:` block: `contents: read`, `pull-requests: write`, `id-token: write` for OIDC, `packages: write` if pushing to GHCR. The token expires at the end of the job. Least-privilege limits blast radius if a workflow or a third-party action it runs is compromised.

### Q14: Why pin actions by SHA?
**Answer:** Action tags (`@v4`) are mutable — the maintainer can move the tag to a different commit, and a compromised maintainer or stolen token can swap malicious code under a tag you already trust. Pinning to a commit SHA (`actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11`) makes the reference immutable. Keep Dependabot enabled for the `github-actions` ecosystem so you still get PRs that bump the SHA after you've reviewed the diff. The cost is a noisier workflow file; the benefit is supply-chain integrity.

## Hard

### Q15: Branch protection and required checks?
**Answer:** Branch protection rules on `main` can require status checks to pass before merge, require PR review, require signed commits, and require linear history. Required checks are workflow job names — list them explicitly so a renamed job does not silently become optional. Combine with `environments` for prod deploys and with `concurrency` groups so only one deploy runs at a time. Note that required-status-check names must match the job name exactly; a job that is skipped due to `if:` can block merge if it is listed as required, so design `if:` conditions carefully or use `jobs.<id>.outputs` to communicate status.

### Q16: How would you deploy via environments and required reviewers?
**Answer:** Define a deploy job with `environment: production`; the environment has a required-reviewer rule, so the workflow pauses when it reaches that job until a reviewer clicks approve in the GitHub UI. Add `concurrency: deploy-prod` so only one prod deploy runs at a time, and `timeout-minutes` so a stuck approval does not hold a runner forever. Use OIDC for cloud auth, environment secrets for prod credentials, and a `url:` for deployment history. Split the deploy from a smoke-test job via `needs: deploy-prod` so a failed smoke test surfaces separately.

### Q17: Handling long-running or deploy jobs?
**Answer:** Set `timeout-minutes` so a hung job is killed rather than burning runner minutes indefinitely. For deploys, use `concurrency` with `cancel-in-progress: false` so deploys queue rather than cancel each other. Split a long deploy into build -> deploy -> smoke-test jobs via `needs` so a smoke-test failure is visible as its own step. For genuinely long jobs (data migrations), consider a self-hosted runner or a scheduled `workflow_dispatch` with manual approval. Persist logs as artifacts (`if: always()`) so you can inspect a failed deploy after the runner is gone.

### Q18: Design a CI/CD pipeline for a backend service.
**Answer:** On PR: lint, typecheck, unit tests, `terraform plan` posted as a comment, security scans (trivy, tfsec). On merge to main: build a container, push to a registry with a SHA tag, deploy to a `staging` environment via OIDC, run smoke tests. Prod deploy is a separate `workflow_dispatch` or an environment-gated job on a release tag: `environment: production` with required reviewers, `concurrency: deploy-prod`, OIDC cloud auth, and a post-deploy smoke-test job via `needs`. Pin actions by SHA, default-deny permissions, cache dependencies by lockfile hash, and upload build/test artifacts with a short retention window.