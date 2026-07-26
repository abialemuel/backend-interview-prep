# Terraform

Terraform is HashiCorp's declarative Infrastructure-as-Code (IaC) tool for provisioning and managing cloud and on-prem resources across hundreds of providers. You write configuration in HCL describing the desired end state, and Terraform figures out a plan to reach it, applies the changes, and records the result in a state file that maps your config to real-world resources.

This directory covers Terraform fundamentals and the state/modules/best-practices knowledge most likely to come up in backend interviews.

Version note: this material targets the **Terraform 1.10+ era** (current stable is **1.15** as of mid-2026). The headline changes of that era: S3-native state locking (no more DynamoDB lock tables), ephemeral values and write-only arguments for keeping secrets out of state, the built-in `terraform test` framework with mock providers, Stacks (GA on HCP Terraform) for multi-environment orchestration, and 1.14/1.15's list resources and dynamic module sources. In 2023 HashiCorp re-licensed Terraform from MPL-2.0 to the Business Source License (BSL), which restricts competitive use. In response the Linux Foundation hosted the **OpenTofu** fork (1.12.x as of mid-2026), which is MPL-2.0, compatible with the fundamentals covered here, and has since added its own features (state encryption, OCI registries). Many shops moving off BSL now run OpenTofu; unless license posture or fork-specific features matter, the CLI surface and HCL are essentially the same.

Sub-files:

| File | Topic |
| --- | --- |
| `01-core-concepts.md` | HCL, blocks, dependency graph, state basics, expressions, version constraints, OpenTofu |
| `02-state-modules-best-practices.md` | Remote state and locking, `terraform state`, import, moved blocks, modules, CI, security, testing |
| `03-interview-questions.md` | Graded interview questions with model answers |

Suggested reading order: read through the core concepts first, then state/modules (which build on state and config), then attempt the questions without peeking, and revisit sections as gaps surface.