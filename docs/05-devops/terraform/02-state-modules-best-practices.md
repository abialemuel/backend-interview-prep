# State, Modules, and Best Practices

## State management

### Why state matters

Terraform's state file is what lets the tool be declarative and fast. Three roles:

1. **Mapping** — each config resource (`aws_instance.app`) is bound to a real cloud object (`i-0abc123`). This binding is the only thing that lets Terraform know "this is mine, don't recreate it."
2. **Performance** — because state caches each resource's last sampled attributes, a plan can refresh only the resources in the config rather than listing an entire subscription/account.
3. **Dependency resolution** — outputs of one resource feed inputs of the next; the state stores those outputs between runs.

### Drift

"Drift" is when the real world no longer matches what state says. Causes: a teammate edits through the console, an autoscaler changes a tag, a cloud provider retires a resource, a CloudFormation stack wanders out of sync. Terraform detects drift on every refresh (`plan` and `apply` both refresh by default). Once detected you either re-apply (Terraform changes reality back to match config) or accept the drifted value (e.g. `terraform state rm` plus a new resource, or `lifecycle.ignore_changes`).

### Local vs remote state

Local state (default `terraform.tfstate` on disk) is acceptable only for throwaway learning. In any team you need remote state because:

- Multiple operators would otherwise clobber each other's edits.
- There is no locking, so two concurrent applies produce torn state.
- State committed to git (the obvious "team" workaround for a single state file) leaks secrets and pollutes diffs with huge JSON.

Remote state lives in a backend — S3, GCS, Azure Blob, TFC/TFE workspaces, Consul, HTTP, etc. The backend is configured in the `terraform` block and initialized by `terraform init`.

### S3 with native locking — the modern pattern

Since Terraform 1.10 (GA in 1.11), the S3 backend locks natively via a conditional `PutObject` of a `.tflock` file next to the state object — **no DynamoDB table needed**:

```hcl
terraform {
  required_version = ">= 1.11"
  required_providers { aws = { source = "hashicorp/aws", version = "~> 6.0" } }
  backend "s3" {
    bucket       = "company-tfstate-prod"
    key          = "network/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
    kms_key_id   = "alias/tfstate"
  }
}
```

- The `key` is the object key inside the bucket; different stacks use different keys, optionally in different folders (`prod/network`, `prod/app`).
- `workspaces` with this backend are stored under `env:/<ws>/<key>`, which is good for branching test stacks while sharing infrastructure config.
- **Locking**: with `use_lockfile = true`, Terraform creates `<key>.tflock` using S3 conditional writes (if-none-match); a second concurrent run fails to acquire it and errors out. After apply the lockfile is deleted. This relies on S3's strong consistency and conditional-write support (2024+), which is why the old pattern existed at all.
- **The legacy pattern**: before 1.10 the lock was a DynamoDB row keyed by the state path (`dynamodb_table = "company-tf-locks"`). It still works but is **deprecated** and slated for removal; expect interviewers to accept either, but the current answer is the S3 lockfile — one less piece of infrastructure to provision, pay for, and IAM-scope. During migration you can set both `use_lockfile` and `dynamodb_table` (Terraform acquires both locks) before dropping the table.
- `encrypt = true` plus a KMS key ensures the state object (which may contain secrets) is encrypted at rest with a key your team controls, not an AWS-managed default key.

State locking and concurrency: a missing or broken lock means concurrent applies can produce torn state — e.g. both plans see the resource absent, both create it, you now have two instances but state records one. That silent corruption is hard to recover from.

### State file sensitivity

State is sensitive by construction. Even with `sensitive = true` on variables and outputs, the underlying provider attributes (DB passwords if the cloud API returns them, IAM secret access keys, private keys) are stored in cleartext in state. Defensive measures:

- Keep state remote, locked, and encrypted (S3 + KMS + native lockfile).
- Never commit `terraform.tfstate` or `*.tfstate.backup`. `.gitignore` it; use a pre-commit hook.
- Restrict the bucket to least-privilege IAM (read/write only for the runner role and break-glass humans).
- Split sensitive resources into their own state with restricted access.
- Use **ephemeral values** (1.10+) and **write-only arguments** (1.11+, e.g. `password_wo`) so secrets pass through to the provider API without ever being written to plan or state — this is the current best practice for things like DB master passwords.
- Long term: draw secrets from a real secret manager (Vault, AWS Secrets Manager, Parameter Store) — ideally via an ephemeral resource — instead of reading them through persisted Terraform attributes.

(OpenTofu additionally offers client-side state encryption, so even a leaked state object is ciphertext; Terraform relies on backend-level encryption.)

### `terraform state` subcommands

| Command | Use |
| --- | --- |
| `terraform state list` | show every resource recorded in state |
| `terraform state show <addr>` | print attributes for one resource as known to state |
| `terraform state mv <a> <b>` | move a resource's address (rename, move into a module) without recreating it |
| `terraform state rm <addr>` | detach a resource from state without destroying the real object |
| `terraform state pull` | output state to stdout (for backup/inspection) |
| `terraform state push <file>` | replace remote state with a local copy (use with extreme care) |
| `terraform state replace-provider <old> <new>` | swap the provider address recorded against all resources |

`state mv` and `rm` modify state directly; they don't change config. The safer alternatives are `moved` blocks (declarative, reviewed in a PR) and `removed` blocks respectively.

### Import

`terraform import <addr> <id>` attaches an existing real-world object to a resource address in state. Before 1.5 you then hand-wrote the matching `resource` block. From **1.5+** the `import` block lets you declare imports in config and generate the HCL:

```hcl
import {
  to = aws_instance.app
  id = "i-0abc123"
}
```

Running `terraform plan -generate-config-out=generated.tf` produces candidate HCL for the imported resources, which you then review and merge. This is the modern pattern for adopting unmanaged infra. Since 1.12, `import` blocks can match by provider-defined `identity` instead of a raw `id`, and **1.14's list resources + `terraform query`** take it further: a `.tfquery.hcl` file can enumerate all matching real-world resources (e.g. every untagged instance) and bulk-generate import blocks — the answer to "how would you adopt 500 existing resources."

### Refreshing

- `terraform refresh` — refreshes state from providers and writes it back. Rarely run directly; **deprecated** as a standalone command in favor of:
- `terraform apply -refresh-only` (or `terraform plan -refresh-only`) — refresh, show the diff that drift caused, then optionally apply it back to state without changing the real world or pursuing config changes. Use this to absorb drift safely or to inspect it.
- `-refresh=false` skips refresh entirely for a faster plan, but it means you trust the cached state — only safe with freshly written state or read-only data-only configs.

### moved blocks (1.1+)

When refactoring (renaming a resource, moving it into a module) you want state to follow without a destroy/recreate cycle. Historically you did `terraform state mv`. Since 1.1 you declare the move:

```hcl
moved {
  from = aws_instance.app
  to   = aws_instance.web
}
```

On the next `plan` Terraform detects the `moved` block, rebinds state from `from` to `to`, and removes the block from the plan (it is consumed as a one-shot migration). This is strictly better than `state mv`: it is reviewable in a PR, repeated across team members automatically, and recorded in git history.

## Modules

### Why modules

Modules are the unit of reuse. They:

- Encapsulate a self-contained piece (VPC, database, load balancer + DNS + ACM cert) behind a typed interface.
- Standardize teams — one blessed VPC module means a security fix lands in one place.
- Keep top-level configs small and declarative: "give me a production VPC with these characteristics."

### Module structure

The conventional layout:

```
modules/vpc/
  main.tf       # resources and data sources
  variables.tf  # input declarations
  outputs.tf    # output declarations
  versions.tf   # required Terraform + provider versions
  README.md      # usage docs
```

### Module sources

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # Registry
  version = "5.0.0"
}

module "db" {
  source = "git@github.com:company/tf-modules//postgres?ref=v1.4.0"   # git ref pin
}

module "cdn" {
  source = "./modules/cdn"   # local path
}
```

- **Registry modules**: versioned, downloaded by `terraform init`, cached in `.terraform/modules`. Use `version = "x.y.z"` not floating tags.
- **Git sources**: pin to a `ref=` (tag or SHA). Without a pin you get whatever is at HEAD.
- **Path sources**: relative paths; useful during development of a module before promoting to a registry.

`terraform init -upgrade` re-resolves module versions to the latest allowed by constraints.

### Module composition — count vs for_each

`count` and `for_each` work on both resources and modules.

```hcl
# count: integer index, each.value is awkward (use count.index)
module "replica" {
  count  = var.enable_replica ? 1 : 0
  source = "./modules/db"
  role   = "replica"
}

# for_each: set or map keys, each.key/each.value
module "bucket" {
  for_each = toset(["logs", "assets", "backups"])
  source   = "./modules/bucket"
  name     = "${each.key}-${var.env}"
}
```

`for_each` is generally preferred: adding or removing a key does not disturb the others (with `count`, removing item N shifts every higher index, which Terraform handles but creates noisy diffs and risks accidental recreate-edge cases). Use `depends_on` at the module level when the engine can not infer ordering, e.g. when a module relies on side effects of another.

### Verified modules and the Registry

The Terraform Registry badges modules as **Verified** when the partner organization controls them. Verified AWS modules (`terraform-aws-modules/*`) are the de-facto standard and rarely hand-rolled. Review the module source the same way you'd review any code dependency.

## Best practices

### Repository structure

Recommended: **one directory per environment, each with its own state file**, optionally wrapped with Terragrunt for DRY backend/provider/source config.

```
envs/
  prod/
    backend.tf   # state backend pointing at prod/...
    network.tf   # module "vpc" { source = "..." }
    app.tf
  staging/
    backend.tf
    network.tf
    app.tf
modules/
  vpc/
  app/
```

Each environment passes different variables to the same modules. Workspaces can collapse identical-shape stacks but make multi-account or cross-region setups awkward; prefer directories for genuine environment differences and use workspaces only for ephemeral stacks.

Terragrunt (Gruntwork's DRY wrapper) is worth knowing for interviews: it initializes remote state config, downloads the same root module across all stacks, passes in `inputs = { ... }`, and orchestrates `init/plan/apply` across many stacks. Know the 2026 alternatives too: **Terraform Stacks** (HCP Terraform, GA — components in `*.tfcomponent.hcl` deployed across many environments from one definition) attack the same problem natively, and Terraform 1.15's dynamic module sources remove one of Terragrunt's original reasons to exist. Third-party orchestration platforms (Spacelift, env0, Scalr, Atlantis for PR-driven plan/apply) fill the same niche for teams not on HCP Terraform.

### State isolation

- One state file per environment at minimum. Many teams go further — one per team per stack (`network/`, `data/`, `app/`) to limit blast radius.
- Locking via the S3 lockfile (`use_lockfile = true`) or the backend-native mechanism (GCS and Azure Blob lock natively; HCP Terraform locks per workspace). DynamoDB lock tables are the legacy AWS answer — still encountered everywhere, but deprecated.
- Access: the runner role can read/write state; humans get read access for break-glass, but apply is gated through CI.

### IAM for the runner

The CI account running Terraform needs broad cloud privileges by definition (it can provision almost anything). Harden it:

- Dedicated AWS account or GCP project for the runner; no production data plane.
- Role with only the managedPolicy-level permissions needed (avoid `AdministratorAccess`).
- Short-lived credentials via OIDC for GitHub Actions → AWS STS AssumeRoleWithWebIdentity.
- State bucket and lock table lockable to specific roles.
- CloudTrail/DataDog alerting on unusual API calls from the runner.

### Sensitive values

- Mark secrets as `sensitive = true` on variables and outputs so plan output masks them.
- Remember state still stores them in plaintext — split sensitive resources into a separate state with restricted backend access, or pull them at runtime from a secret manager (Vault, AWS SM) instead of reading them through Terraform.
- Never `echo` declared-sensitive values in CI; mark secrets as repo/env secrets and pass them through.

### Plan in CI, gated apply

- CI should run `terraform fmt -check`, `terraform validate`, `terraform plan`, and comment the plan on the PR. The plan is the reviewer's evidence.
- Apply is gated: a separate job on `main` (or an environment with required reviewers) runs `terraform apply -auto-approve` only after the plan approval.
- Never allow `apply` from an arbitrary PR or feature branch.

### Pre-commit hooks and linting

- `terraform fmt -recursive` — formatting. The CI `-check` form must pass.
- `terraform validate` — schema and type checks. Cheap, run it everywhere.
- `tflint` — catches provider-specific smells (e.g. an invalid `instance_type`, a deprecated `ami`).
- `trivy` (which absorbed `tfsec`) — security scanning (e.g. S3 bucket not encrypted, security group 0.0.0.0/0, IAM `*` action). Run in CI and optionally as a pre-commit hook.
- `checkov` — similar, policy-as-code.

### Policy as code

Security scanners answer "is this pattern known-bad"; **policy as code** answers "does this change comply with *our* rules" — and blocks the pipeline when it doesn't. Two main engines:

- **OPA (Open Policy Agent)** — vendor-neutral CNCF standard; policies in Rego evaluate the JSON plan (`terraform show -json plan.out | conftest test -`). Rules like "every resource must carry a `cost-center` tag," "no public S3 buckets," "RDS must be multi-AZ in prod." Free, runs in any CI.
- **Sentinel** — HashiCorp's proprietary engine, embedded in HCP Terraform/Enterprise between plan and apply, with soft-mandatory (overridable) and hard-mandatory enforcement levels.

The staff-level framing: linting catches mistakes, policy-as-code encodes organizational guardrails, and both run against the *plan* so violations are blocked before they exist in the cloud.

### The dangers of `terraform apply` without review

- Auto-applying a typo or refactor in `main` can destroy resources in seconds. The same DAG that creates in parallel destroys in parallel.
- `prevent_destroy` lifecycle locks critical resources from being deleted even by an otherwise-valid apply.
- Always read the plan; never trust CI to apply on green without a human having seen the plan.

### Resource targeting `-target`

`terraform apply -target=module.app.aws_instance.app` restricts the operation to the targeted resource and its dependencies. It is an escape hatch for emergencies ("patch this one resource now") and local development. Pinning `-target` in regular CI is a smell — it leaves drift elsewhere and hides dependencies. After any targeted apply, run a full plan to confirm the rest reconciles.

### Lifecycle blocks

```hcl
resource "aws_security_group" "this" {
  name = "app"

  lifecycle {
    create_before_destroy = true           # build the replacement before destroying the old
    prevent_destroy        = true           # guard against accidental destroy, must remove block to delete
    ignore_changes         = [tags["Owner"]] # ignore drift on these attributes
  }
}
```

- `create_before_destroy` — required for things like security groups that are referenced by name and must overlap in time. The whole resource is replaced, not patched.
- `prevent_destroy` — refuses `destroy`/`replace` of the resource while the block is present. To actually destroy, remove the lifecycle block and apply first.
- `ignore_changes` — ignores drift on listed attributes; useful when an external process (autoscaler, secrets rotator) legitimately mutates them.

### Provisioners — and why to avoid them

Provisioners (`remote-exec`, `local-exec`, `file`) run commands at create/destroy time on the resource or locally. They are non-hermetic:

- They are not idempotent — re-running them on update requires destroy+recreate.
- Their result is invisible to state; the engine does not know whether they succeeded.
- They couple Terraform to the network path SSH, instance user-data, etc.

Prefer: user-data scripts, configuration management (Ansible) for ongoing config, cloud-init, or `userdata`/`ignore_changes` to trigger a fresh boot only when needed. `null_resource` and `terraform_data` (1.4+) are escape hatches for arranging side-effect ordering; treat them as last resort.

### Testing

- **Native `terraform test`** (1.6+) — the current default answer. `.tftest.hcl` files contain `run` blocks that plan or apply a config with given variables and `assert` on outputs/attributes. `command = plan` tests are fast and free; `command = apply` tests stand up real infra in isolated state and tear it down. Since 1.7, `mock_provider` blocks let unit tests run with synthetic provider data — no cloud credentials at all.
- **Terratest** (Gruntwork) — Go tests that run `terraform init/plan/apply`, then assert against the cloud with SDK calls. Most established for full-stack integration tests; heavier to maintain.
- The rule of thumb: `terraform test` with mocks for module unit tests in every PR; a small set of apply-mode or Terratest smoke tests on a schedule or pre-release.

### GitOps and Terraform — the comparison interviewers ask

"GitOps" in the strict sense (ArgoCD, Flux) is a *pull-based* model for Kubernetes: a controller in the cluster continuously reconciles live state against a git repo. Terraform CI/CD is *push-based*: nothing reconciles between runs, so drift persists until the next plan. Atlantis / HCP Terraform / Spacelift bring the PR-driven workflow ("plan on PR, apply on merge") to Terraform, and HCP Terraform's drift detection adds scheduled reconciliation, but there is no always-on controller. The clean answer: use Terraform (or OpenTofu) for cloud resources with PR-gated applies, and ArgoCD/Flux for what runs *inside* Kubernetes — and be able to articulate that continuous reconciliation vs point-in-time apply is the fundamental difference.

### Summary checklist

- One state file per environment; remote + locked (`use_lockfile = true` on S3) + encrypted.
- Pin providers with `~>`; pin module versions to tags.
- `fmt -check`, `validate`, `tflint`, `trivy` in CI; comment the plan on PRs; OPA/Sentinel policy checks against the plan JSON.
- Apply gated behind merge to `main` or an environment with required reviewers; CI credentials via OIDC, not stored keys.
- `moved`/`removed`/`import` blocks for refactor and adoption; list resources + `terraform query` (1.14+) for bulk adoption.
- Keep secrets out of state with ephemeral values and write-only arguments; pull them from a secret manager.
- Avoid provisioners; use user-data + Ansible.
- `terraform test` with mock providers for module unit tests.
- Use OpenTofu if BSL is a compliance/licensing concern.