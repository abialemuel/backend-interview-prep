# Ansible

Ansible is a declarative, agentless configuration management and orchestration tool. You describe the desired state of hosts in YAML "playbooks," and Ansible pushes changes over SSH (Linux) or WinRM (Windows) from a control node — there are no agents installed on targets, and no daemon to keep running. Tasks are idempotent by design: a module either makes the change or reports "already in that state," so re-running a playbook converges rather than accumulates.

This directory covers the Ansible fundamentals and interview-relevant knowledge for backend engineers.

Version note: this material targets **ansible-core 2.19** (the engine plus `ansible.builtin` modules), shipped in the **Ansible community package 12** (current as of mid-2026). Since 2.10 Ansible has been split into `ansible-core` (the engine + builtins) and the **Ansible community distribution** (a curated bundle of collections). Install one or the other depending on whether you want minimal core or the batteries-included distribution; collection-based modules live under namespaces like `community.aws`, `community.docker`, `ansible.posix`. The commercial layer is **Red Hat Ansible Automation Platform (AAP) 2.6** — controller (the former Tower/AWX), Event-Driven Ansible, execution environments, and Ansible Lightspeed — which interviews at enterprise shops increasingly touch on.

Sub-files:

| File | Topic |
| --- | --- |
| `01-core-concepts.md` | Inventory, modules, playbooks, roles, collections, Vault, Jinja2, Ansible vs Terraform |
| `02-interview-questions.md` | Graded interview questions with model answers |

Suggested reading order: read the core concepts first (they are tightly coupled — inventory feeds playbooks, playbooks use modules, roles package playbooks), then attempt the questions and revisit sections as gaps surface.