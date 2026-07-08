# Terraform Interview Questions

## Easy

### Q1: What is Infrastructure as Code and why use it?
**Answer:** IaC is the practice of defining infrastructure — servers, networking, databases — in version-controlled text files rather than manual console clicks or shell scripts. It gives you repeatability (same config produces the same infra), reviewability (changes flow through PRs), auditability (git history shows who changed what), and velocity (whole environments can be torn down and rebuilt). Terraform is declarative IaC: you describe the desired end state and the tool computes and applies the diff to reach it. The other payoff is idempotency — running the same config twice yields the same result as running it once.

### Q2: Declarative vs imperative IaC?
**Answer:** Declarative (Terraform, CloudFormation) describes the desired end state and lets the tool figure out the steps; imperative (Ansible for config management, shell scripts) describes the steps to take. Declarative tools manage idempotency for you by refreshing current state and computing a diff each run, so you never describe "how to undo my last run." Imperative tools are more flexible but the operator is responsible for order, error recovery, and detecting "already done." Terraform is declarative at the provisioning layer; Ansible is commonly used alongside it for the imperative configuration step.

### Q3: What is Terraform state for?
**Answer:** State (`terraform.tfstate`) is a JSON file that maps each config resource to a real cloud object plus its last-sampled attributes. It is the source of truth that makes planning possible: Terraform refreshes each resource from the provider, compares the refreshed attributes to the desired config, and emits only the diff. State also caches attributes so plans do not have to list entire cloud accounts, and it carries outputs that feed inputs of dependent resources. Without state, Terraform could not tell which real objects it owns and which it does not.

### Q4: Walk through the Terraform workflow.
**Answer:** `terraform init` resolves providers and modules and initializes the backend; `terraform fmt` and `validate` lint locally; `terraform plan` refreshes state, computes the diff, and prints it; `terraform apply` performs the planned changes and writes state after each resource; `terraform destroy` removes every resource tracked in state. `init` is the only step that touches the network for providers/modules; plan and apply refresh resources through the provider APIs. In CI you typically run fmt -check, validate, plan, and gate apply behind merge to main.

### Q5: What is a provider and where does it come from?
**Answer:** A provider is a separate binary that implements CRUD API calls against a given cloud or service (AWS, GCP, Azure, GitHub, Datadog). Each resource type like `aws_instance` lives in a provider. Providers are resolved from the Terraform Registry at `registry.terraform.io` by an address like `hashicorp/aws`, and pinned with a `version = "~> 5.0"` constraint. HashiCorp-maintained providers are "official," partner providers are "verified," and community providers should be audited. `terraform init` downloads providers into `.terraform/providers`.

### Q6: What is the difference between a resource and a data source?
**Answer:** A `resource` is something Terraform owns end to end — it creates, reads, updates, and deletes it. A `data` source is a read-only lookup of something that already exists, queried at plan/apply time. Use data sources to decouple modules from upstream concerns: a module that needs the latest Ubuntu AMI should `data "aws_ami"` rather than take an AMI ID as a variable, so the caller does not have to know AMI IDs. Both end up as attributes in state, but only resources can be modified by Terraform.

## Medium

### Q7: Why use remote state? Describe the S3 + DynamoDB pattern.
**Answer:** Remote state lives in a backend — S3, GCS, TFC, Consul — instead of a local file, because team use requires a shared, locked, encrypted store. The classic AWS pattern stores state in an S3 bucket and uses a DynamoDB table for locking: Terraform writes a row keyed by the S3 key before plan/apply and deletes it after; if a row exists the operation blocks or fails. S3 alone does not lock — DynamoDB is the lock. Encryption at rest via KMS and least-privilege IAM on the bucket round out the setup. Different stacks use different `key` paths in the same bucket; workspaces under this backend are stored at `env:/<ws>/<key>`.

### Q8: What happens if you run Terraform without state locking?
**Answer:** Two concurrent applies can both read the same pre-state, both plan "create," and both create the real resource; state then records only one of them, leaving the other orphaned (or vice versa, leaving the recorded one destroyed). The corruption is silent and hard to recover from — you discover it on the next plan when reality and state disagree. This is why every remote backend used by a team must have a locking mechanism (DynamoDB for S3, native locking for TFC/Consul), and why local state is never acceptable for shared infra.

### Q9: Why must you never commit terraform.tfstate to git?
**Answer:** State is sensitive by construction: even with `sensitive = true` on variables and outputs, provider attributes like DB passwords and IAM secret keys land in state in plaintext. Committing it to git leaks those secrets in history forever, even after a later `rm`. It also produces giant JSON diffs and lets two operators run with stale copies. The correct setup is remote + locked + encrypted state in a backend with least-privilege IAM, plus a `.gitignore` rule for `*.tfstate` and `*.tfstate.backup`, enforced by a pre-commit hook.

### Q10: What is drift and how do you detect and remediate it?
**Answer:** Drift is any difference between what state says exists and what the cloud actually has, caused by out-of-band edits, autoscalers, provider retirement, etc. Terraform detects it by refreshing each resource from the provider at plan/apply time. Remediation is either re-apply (Terraform reconciles reality back to config) or accept the drift via `terraform state rm` plus a new resource, or `lifecycle.ignore_changes` for attributes that legitimately change outside Terraform. `terraform plan -refresh-only` shows the drift without applying config changes, which is the safest way to inspect it before deciding.

### Q11: How does `terraform import` work, and what changed in 1.5+?
**Answer:** `terraform import <addr> <id>` attaches an existing real-world object to a resource address in state without creating or destroying anything. Before 1.5 you then hand-wrote the matching `resource` block to match the imported attributes. From 1.5 you can declare an `import` block in config and run `terraform plan -generate-migration` to emit the candidate HCL for you, which you review in a PR. This makes adoption of unmanaged infra declarative and reviewable rather than a one-shot CLI command. Import never modifies the real resource — it only reads it into state.

### Q12: `moved` blocks vs `terraform state mv`?
**Answer:** Both rebind state from one address to another without recreating the real resource. `state mv` is an imperative, one-off CLI command run by one operator; the rest of the team has no record of it. A `moved` block is declarative HCL committed to git: on the next plan Terraform performs the rebind automatically for everyone and consumes the block as a one-shot migration. `moved` blocks are strictly better for refactors because they are reviewable in a PR, recorded in git history, and replayed across team members. Use `state mv` only for emergencies or local scratch work.

### Q13: `terraform refresh` vs `-refresh-only` plan?
**Answer:** `terraform refresh` refreshes state from providers and writes it back as a standalone operation — it is now deprecated in favor of the safer `-refresh-only` mode. `terraform plan -refresh-only` (or `apply -refresh-only`) refreshes, shows you the diff that drift caused, and lets you choose to apply it back to state without changing the real world or pursuing config changes. The standalone `refresh` could silently mutate state; `-refresh-only` makes the change visible and reviewable. For trusted-just-written state, `-refresh=false` skips refresh entirely for a faster plan.

### Q14: Why use modules?
**Answer:** Modules are the unit of reuse: they encapsulate a self-contained piece (a VPC, a database plus DNS plus cert) behind a typed interface, standardize teams so a security fix lands in one place, and keep top-level configs small and declarative. The conventional structure is `main.tf` / `variables.tf` / `outputs.tf` / `versions.tf` / `README.md`. Sources can be the Registry (`terraform-aws-modules/vpc/aws` with a `version` pin), git with a `ref=` pin, or a local path. Always pin module versions to a tag or SHA, never to a floating branch.

### Q15: `count` vs `for_each`?
**Answer:** Both create multiple instances of a resource or module. `count` takes an integer and gives you `count.index`; removing an item shifts every higher index, which produces noisy diffs and can risk accidental recreate edge cases. `for_each` takes a set or map and gives you `each.key`/`each.value`; adding or removing a key leaves the others untouched. `for_each` is generally preferred for that reason. Use `count` for simple boolean conditionals (`count = var.enabled ? 1 : 0`) and `for_each` for collections.

### Q16: Workspaces and when to avoid them?
**Answer:** Workspaces are named slices of a single state in the same backend, convenient for ephemeral stacks or identical-shape multi-region deploys against shared config. They are a poor fit for genuine environments: prod and staging usually differ in accounts, regions, provider configs, or variable sets, and sharing one config across workspaces forces `terraform.workspace` conditionals that bloat the code. The common advice is to use separate directories with separate state files per environment, and reserve workspaces for short-lived dev/test branches. Terragrunt makes the directory approach DRY by sharing backend and provider config.

### Q17: Why separate state per environment?
**Answer:** Separate state files limit blast radius — a bad apply in staging cannot destroy prod resources, and a corrupted state recovery only affects one environment. They also allow different IAM and backend configs per environment (prod state bucket has stricter access than dev). The trade-off is some duplication of backend config, which Terragrunt or a shared root module addresses. The alternative — one state file with workspaces or environment-scoped resources — couples every environment to a single point of failure and a single lock.

## Hard

### Q18: Terragrunt — what problem does it solve?
**Answer:** Terragrunt is a thin wrapper from Gruntwork that keeps Terraform code DRY across many stacks. It lets you define backend config, provider config, and module inputs once and inherit them across environments via `terragrunt.hcl` files, instead of repeating `backend "s3"` blocks and module source paths in every env directory. It also orchestrates `init/plan/apply` across multiple modules in dependency order, manages remote state setup, and supports `--terragrunt-parallelism` to apply many stacks at once. The main reason it appears in interviews is the DRY backend story for the one-state-per-env directory pattern.

### Q19: Plan in CI — what does a good pipeline look like?
**Answer:** On every PR: `terraform fmt -check`, `terraform validate`, `tflint`, `tfsec`/`trivy` for security, then `terraform plan` against the target environment's backend with credentials from OIDC. The plan output is posted as a PR comment so reviewers can see the diff. Apply is gated: a separate job on merge to `main` (or via an environment with required reviewers) runs `terraform apply -auto-approve`, but only after a human has approved the plan. Never auto-apply from an arbitrary feature branch, and never run apply without a reviewed plan.

### Q20: Sensitive values in state — what is and isn't solved by `sensitive = true`?
**Answer:** `sensitive = true` on a variable or output masks the value in plan output, but it does NOT prevent the value from being written to state in plaintext. State can still contain DB passwords, IAM secret keys, private keys returned by provider APIs. Real protections are remote + locked + encrypted state, least-privilege IAM on the state bucket, splitting sensitive resources into a separate state with restricted access, and pulling secrets at runtime from a secret manager (Vault, AWS Secrets Manager) instead of reading them through Terraform attributes. Sensitive marking is a UX guard, not a security control.

### Q21: `prevent_destroy`, `ignore_changes`, `create_before_destroy` — when do you use each?
**Answer:** `prevent_destroy` refuses to destroy or replace a resource while the block is present; use it on durable resources like production databases or S3 buckets with irreplaceable data, so a refactor typo cannot delete them. `ignore_changes` tells Terraform to stop tracking drift on listed attributes; use it when an external process (autoscaler, secrets rotator, console tagging) legitimately mutates them and you do not want Terraform to fight it. `create_before_destroy` builds the replacement before tearing down the old; use it on resources referenced by name (security groups, IAM policies) that must overlap in time to avoid dangling references.

### Q22: Why avoid provisioners?
**Answer:** Provisioners (`remote-exec`, `local-exec`, `file`) run commands at create/destroy time but are non-hermetic: they are not idempotent (re-running requires destroy+recreate), their result is invisible to state (the engine does not know whether they succeeded), and they couple Terraform to the network path (SSH reachability, instance user-data). They are also hard to test. Prefer user-data scripts for first-boot, configuration management (Ansible) for ongoing config, and `terraform_data` (1.4+) or `null_resource` only as a last-resort escape hatch for ordering side effects. Treat any provisioner in a code review as a smell.

### Q23: Provider version pinning — what does `~> 5.0` mean and why use it?
**Answer:** `~> 5.0` is the pessimistic constraint: at least 5.0 but below 6.0, so you get minor and patch releases automatically but never a major that may break. `~> 5.20.0` is stricter — it allows 5.20.x patches but not 5.21. Always pin providers so `terraform init` on a fresh machine resolves the same version your state was written against; unpinned providers drift across runs and a new major can change resource schemas in ways that force recreation. Combine with `required_version` on the `terraform` block to pin the CLI itself.

### Q24: tflint vs tfsec?
**Answer:** `tflint` is a linter focused on provider-specific correctness — invalid `instance_type` values, deprecated AMIs, unused declarations, cross-resource consistency. It catches things `terraform validate` (which only checks schema and types) cannot. `tfsec` (now part of `trivy`) is a security scanner that flags risky patterns: unencrypted S3 buckets, security groups open to 0.0.0.0/0, IAM `Action: "*"` policies, plaintext secrets. Both run in CI and as pre-commit hooks. `checkov` is a similar policy-as-code alternative to tfsec. Run `fmt -check`, `validate`, `tflint`, and `tfsec`/`trivy` in that order on every PR.

### Q25: OpenTofu vs Terraform — what is the practical difference?
**Answer:** In 2023 HashiCorp re-licensed Terraform from MPL-2.0 to the Business Source License (BSL), which restricts competitive use (e.g. building a hosted Terraform competitor). The Linux Foundation forked OpenTofu (first GA 1.6), which remains MPL-2.0. As of 1.6+ the HCL, CLI, and provider/plugin protocol are the same as Terraform 1.x — you can run `tofu init/plan/apply` on existing `.tf` files and state written by Terraform. The practical difference is licensing posture: organizations with BSL compliance concerns run OpenTofu; the technical content (state, modules, providers) is essentially identical.

### Q26: `terraform test` (1.6+) — what is it and when use it?
**Answer:** Native `terraform test` runs `.tftest.hcl` files that stand up a small configuration in isolated state, make assertions against outputs and resource attributes, and tear down. It is integration-style testing inside the Terraform toolchain, without needing a separate framework. For heavier validation (cross-cloud SDK assertions, HTTP checks after apply), Terratest (Go) or kitchen-terraform (Ruby) are more powerful. Use `terraform test` for module unit tests and quick verifications, and Terratest for full-stack smoke tests against a real cloud.