# Core Concepts

## What IaC is and why

Infrastructure-as-Code means defining your infrastructure — VMs, networking, databases, queues, IAM — in version-controlled text files rather than clicking through a console or hand-running scripts. The payoff is repeatability (the same config produces the same infra every time), reviewability (changes go through pull requests), auditability (the git history shows who changed what and when), and velocity (whole environments can be torn down and rebuilt).

Two paradigms:

- **Declarative** (Terraform, CloudFormation, Pulumi YAML): you describe the desired end state and the tool computes the diff and the order of operations. The tool is responsible for reaching the state.
- **Imperative** (Ansible for config management, shell scripts, AWS CLI): you describe the steps to take. You must handle order, error recovery, and "is it already done?" yourself.

The defining property that makes declarative IaC usable is **idempotency**: running the same config twice produces the same result as running it once. Terraform achieves this by reading the current state of each resource from the provider, comparing to the desired config, and only issuing the mutating API calls needed to reconcile the two.

## Terraform architecture

Terraform has three moving parts:

1. **Terraform Core** (the binary) parses HCL, builds a resource graph, plans, and drives the apply loop.
2. **Providers** (separate binaries, fetched from the registry) implement the CRUD API calls against a given cloud or service. Each resource type like `aws_instance` lives in a provider (`hashicorp/aws`).
3. **State** (`terraform.tfstate`) records which real-world object each config resource maps to plus its last-known attributes. Core reads state to know what exists and writes state after each apply.

## HCL syntax

HCL is block-structured. The general form is:

```hcl
<block_type> "<label>" "<label>" {
  <argument> = <value>
  <argument> = <value>
}
```

Labels are strings; arguments are `key = value` pairs where values can be literals, expressions, or references to other resources. Comments are `#` and `//` (line) and `/* */` (block).

### The block types

```hcl
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
  backend "s3" {
    bucket       = "my-tfstate"
    key          = "prod/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true   # S3-native locking (1.10+, GA in 1.11) — no DynamoDB table needed
    encrypt      = true
  }
}

provider "aws" {
  region = "us-east-1"
  default_tags { tags = { managed_by = "terraform" } }
}

variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "Size of the app server"
  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Must be a t3 family instance."
  }
  sensitive = false
}

locals {
  azs    = ["us-east-1a", "us-east-1b", "us-east-1c"]
  fqdn   = "${var.app_name}.${var.domain}"
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  tags          = { Name = local.fqdn }

  lifecycle {
    create_before_destroy = true
    ignore_changes         = [tags["LastRotated"]]
  }
}

data "aws_ami" "ubuntu" {
  most_recent = true
  filter { name = "name"; values = ["ubuntu/images/hvm-ssd/ubuntu-noble-24.04-amd64-server-*"] }
  filter { name = "virtualization-type"; values = ["hvm"] }
  owners = ["099720109477"]
}

output "instance_id" {
  value       = aws_instance.app.id
  description = "ID of the app server"
  sensitive   = false
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  cidr    = "10.0.0.0/16"
}
```

- **`terraform`** — top-level config: required provider/version, backend (state storage), experiments, provider-side defaults.
- **`provider`** — configures a provider; multiple instances allowed with `alias`.
- **`variable`** — inputs to the configuration; typed, with defaults, validation, `sensitive`, `nullable`.
- **`resource`** — the central primitive: declares a real thing to manage. Referenceable as `<TYPE>.<NAME>`.
- **`data`** — reads an existing thing managed elsewhere; queried at plan/apply time.
- **`locals`** — named expressions scoped to the current module, useful to avoid repetition.
- **`output`** — values exposed to callers (parent module or shell). Can be marked `sensitive`.
- **`module`** — instantiate a child module by `source` and `version`.

## The workflow: init -> plan -> apply -> destroy

```bash
terraform init      # resolve providers and modules, init the backend
terraform fmt       # rewrite files to canonical formatting (run -check in CI)
terraform validate  # static typechecks against schemas
terraform plan      # compute and display the diff
terraform apply     # apply; prompts unless -auto-approve
terraform destroy   # remove every resource tracked in state
```

`init` is the only step that touches the network for providers/modules (and state). `plan` is read-only: it refreshes state from the provider, diffs against config, and prints the proposed change set. `apply` re-runs that diff internally and executes it.

### Providers and the registry

Providers are distributed from the **Terraform Registry** (`registry.terraform.io`), the same domain OpenTofu also reads by default. A provider address is `namespace/name`, e.g. `hashicorp/aws`. HashiCorp-maintained providers are "official"; partner providers are "verified". Community providers exist but should be audited. Local or in-house providers build against the plugin SDK or `terraform-plugin-framework`.

### Resources vs data sources

A `resource` is something Terraform manages end-to-end (create, read, update, delete). A `data` source is a read-only lookup of something that already exists — an AMI, the caller's current AWS account ID, a subnet by tag. Use data sources to decouple modules from upstream concerns: a module that needs "the latest Ubuntu AMI" should `data "aws_ami"` rather than accept an AMI ID as a variable.

## The dependency graph

Terraform builds a **DAG** of resources from two sources:

1. Explicit references — `aws_instance.app.ami_id = data.aws_ami.ubuntu.id` creates an edge data -> resource.
2. `depends_on` declarations for dependencies the engine cannot see (e.g. ordering side effects across resources).

The graph drives parallelism: resources with no edges between them are created concurrently (`-parallelism=N` caps workers). It also explains the plan: if a provider-read for `data.aws_subnet.example` depends on `aws_vpc.example`, the VPC is applied first.

## State

`terraform.tfstate` is a JSON file mapping each config resource to a real object ID plus its last-sampled attributes (`{"type.name": {"schema_version":0,"attributes":{"id":"i-abc",...}}}`). State is the source of truth that makes plan work: the engine refreshes each resource's attributes from the provider, compares to the desired config, and emits the diff. Without state Terraform would have to inspect the entire account on every run and could not distinguish things it owns from things it doesn't.

Implications:

- **Performance**: state carries the last-known attributes, so most plans only refresh listed resources instead of listing the whole cloud.
- **Drift**: somebody editing a resource out-of-band is detected on the next refresh and surfaced as a diff.
- **Secrets**: state can contain secrets (`password`, `access_key` attributes, `sensitive = true` values). Treat state as a secret itself: remote + locked + encrypted, never committed.

## The core loop

```
config.tf  ──► plan: refresh state from providers ──► diff ──► apply: issue API calls ──► write state
   ▲                                                                            │
   └────────────────────────────────────────────────────────────────────────────┘
```

A real `plan`:

1. Load config and state.
2. For each resource in state, call the provider's `Read` to refresh attributes (this is the network-touching part; `plan -refresh=false` skips it).
3. Diff refreshed attributes against the desired config.
4. Emit proposed actions: `+ create`, `~ update in-place`, `- destroy`, `+/- destroy-then-recreate` (the `forces-replacement` case).
5. Print and exit.

`apply` does the same and then performs the actions, persisting state after each one so partial failures leave a consistent record.

## Variables, outputs, locals

Variable types: string, number, bool, list, set, map, tuple, object, `any`. Type constraints matter — they surface errors at plan time, not apply time. Validation rules can express arbitrary `condition`/`error_message` pairs. `sensitive = true` masks the value in plan output but does NOT prevent it being written to state — splitting resources or using a secret manager is the real protection.

```hcl
variable "common_tags" {
  type = map(string)
  default = { env = "prod" }
}

output "endpoint" {
  value     = aws_lb.this.dns_name
  sensitive = false
}

locals {
  short_id = substr(uuid(), 0, 8)
}
```

Locals are evaluated per module and only where actually referenced.

## Ephemeral values (1.10+) and write-only arguments (1.11+)

The historical weakness of Terraform secrets handling was that everything flowing through config landed in plan files and state. Two features close that gap:

- **Ephemeral values** (1.10): `variable` and `output` blocks can be marked `ephemeral = true`, and **ephemeral resources** (`ephemeral "aws_secretsmanager_secret_version" "db" { ... }`) fetch a value at run time — none of it is persisted to plan or state. Use them to fetch a short-lived credential (a Vault token, a Secrets Manager value) that only needs to exist during the run.
- **Write-only arguments** (1.11): provider-declared resource arguments (e.g. `password_wo` on `aws_db_instance`) that accept an ephemeral value, are sent to the API, and are *never* stored in state. A companion version argument (`password_wo_version`) triggers rotation. This is the modern answer to "how do I set a DB password without it living in state in plaintext."

```hcl
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/master"
}

resource "aws_db_instance" "main" {
  # ...
  password_wo         = ephemeral.aws_secretsmanager_secret_version.db.secret_string
  password_wo_version = 1   # bump to rotate
}
```

Interviewers increasingly probe this because it changes the old canonical answer: `sensitive = true` only masks output; ephemeral values and write-only arguments actually keep the secret out of state.

## Expressions

```hcl
# for / for_each
resource "aws_subnet" "public" {
  for_each = toset(local.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr, 8, index(local.azs, each.value))
  availability_zone = each.value
  tags              = { Name = "public-${each.value}" }
}

# splat
output "subnet_ids" { value = [for s in aws_subnet.public : s.id] }
# or the projection shorthand:
output "subnet_ids_short" { value = aws_subnet.public[*].id }

# conditional
size = var.high_memory ? "r6i.xlarge" : "m6i.large"

# dynamic block
resource "aws_security_group" "this" {
  name = "app"
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

Terraform functions are pure (`join`, `split`, `concat`, `cidrsubnet`, `regex`, `format`, `md5`, `jsonencode`, `base64encode`, etc). There is no user-defined function except local values; for non-trivial logic extract a module.

## Workspaces

Workspaces are named slices of a single state in the same backend. They are convenient for short-lived dev/test splits against the same backend config — `terraform workspace select staging`. They share config, which makes them a poor fit for environments that legitimately differ (e.g. prod has different accounts, regions, or provider configs). The common advice is: use **separate directories and separate state files** for environments, and treat workspaces only as a temporary-scratch tool or for identical-shape multi-region deploys. Tools like Terragrunt make the directory approach DRY.

## Version constraints

| Syntax | Meaning |
| --- | --- |
| `>= 5.0` | at least 5.0 |
| `~> 5.0` | at least 5.0 but below 6.0 (a.k.a. pessimistic / compatible-with) |
| `~> 5.20` | at least 5.20 but below 6.0 — note minor pinning tracks patch releases only with `~> 5.20.0` |
| `= 5.20.1` | exactly 5.20.1 |
| `>= 5.0, < 6.0` | explicit range |

The `~>` operator is the interview favorite: it lets you get patch (and minor, depending on operands) updates automatically while avoiding majors that may break. Always pin providers; unpinned (`version = "5.20.1"` is fine too) drifts across `init` runs. The `required_version` on the `terraform` block works the same way for Terraform itself.

## What changed since 1.10 — the release cheat sheet

Interviewers like "what's new in Terraform" as a currency check. The recent line:

| Version | Highlights |
| --- | --- |
| **1.10** (late 2024) | Ephemeral values and ephemeral resources; S3-native state locking (`use_lockfile`, experimental) |
| **1.11** (2025) | Write-only arguments; S3-native locking GA — DynamoDB lock tables now legacy |
| **1.12** (2025) | OCI Object Storage backend; `import` blocks accept resource `identity` |
| **1.13** (2025) | `terraform stacks` CLI commands; RPC API GA |
| **1.14** (late 2025) | List resources + `terraform query` for bulk discovery/import; `actions` block for imperative lifecycle hooks |
| **1.15** (2026, current stable) | Variables allowed in module `source`/`version` (dynamic sources); formal `deprecated` mechanism for variables/outputs; output type constraints |

**Terraform Stacks** (HCP Terraform, GA) deserve a sentence: a stack composes multiple components (`*.tfcomponent.hcl`) with shared inputs and deploys them across many environments/regions from one definition — HashiCorp's answer to the Terragrunt-style multi-stack orchestration problem, but tied to HCP Terraform.

## OpenTofu note

After the 2023 BSL re-license, the Linux Foundation's **OpenTofu** is the open-source continuation. The HCL, CLI, and provider/plugin protocol remain compatible with Terraform 1.x — you can use OpenTofu on existing `.tf` files with `tofu init/plan/apply`, against the same registry. The licensing posture is the practical difference: BSL restricts competitive offerings; OpenTofu is MPL-2.0. Since the fork the projects have also diverged on features — OpenTofu (1.12.x as of mid-2026) shipped native client-side **state encryption**, `for_each` on provider blocks, early variable evaluation in backend/module blocks, an `-exclude` flag, and OCI-registry distribution for providers/modules, several of which Terraform later matched (e.g. 1.15's dynamic module sources). The core technical content here applies to either.