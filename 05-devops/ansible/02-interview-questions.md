# Ansible Interview Questions

## Easy

### Q1: Why is Ansible agentless and what does that mean?
**Answer:** Agentless means Ansible installs nothing on managed hosts — no daemon, no agent process, no extra port to expose. The control node connects over SSH (Linux) or WinRM (Windows), ships a small Python module to the target, executes it, and removes it. This removes the need to maintain agent upgrades, agent security, and a central server, and it works against hosts you can reach over SSH but cannot easily install software on (appliances, hardened boxes). The trade-off is that the control node must be able to reach every target, and SSH overhead limits throughput on very large inventories.

### Q2: How does Ansible achieve idempotency?
**Answer:** Modules check current state before acting and only make changes when needed. `yum name=htop state=present` queries the package manager and only invokes `yum install` if `htop` is missing; on a converged host it reports `ok` not `changed`. `template` compares the rendered content to the destination file and only writes when they differ. `systemd state=started enabled=true` checks the unit's running and enabled state and only acts if needed. The `command` and `shell` modules are not idempotent by default — they run every time — which is why you reach for the declarative module first or add `creates`/`removes`/`when` guards.

### Q3: Modules vs `command`/`shell`?
**Answer:** Declarative modules (`yum`, `file`, `template`, `systemd`) understand the resource they manage and report idempotent `changed`/`ok` state. `command` runs a command without shell processing (no pipes, env, redirects) and `shell` runs through `/bin/sh`. Both execute every time they are reached, so they are not idempotent by default; you make them safe with `creates:`/`removes:` arguments or `when` conditions. Use a module whenever one exists for the task; fall back to `command`/`shell` only for one-off scripts or things no module covers.

### Q4: Static vs dynamic inventory?
**Answer:** Static inventory is an INI or YAML file you maintain by hand, fine for a fixed set of on-prem hosts. Dynamic inventory is generated at run time by a plugin or script that queries a cloud API (AWS, GCP, Azure) or CMDB, so hosts that appear or disappear are reflected automatically. Use dynamic inventory whenever your fleet is cloud-based or autoscaled; the `amazon.aws.aws_ec2` plugin, for example, returns groups like `tag_env_prod` from instance tags. You can combine static and dynamic inventory sources in a directory passed with `-i`.

### Q5: What is `become`?
**Answer:** `become: true` enables privilege escalation for a play or task; `become_user` selects the target user (root by default) and `become_method` selects the mechanism (sudo, su, pbrun, doas). It is the Ansible equivalent of `sudo` and is required for tasks that need root: package installs, service management, editing files owned by root. You can scope it per-play, per-task, or per-host in inventory (`ansible_become: true`).

### Q6: What are facts and the `setup` module?
**Answer:** Facts are variables Ansible gathers about each host at the start of a play via the `setup` module — OS family, distribution, architecture, network interfaces, mounted filesystems, CPU, memory, etc. They are namespaced under `ansible_*` (e.g. `ansible_os_family`, `ansible_eth0.ipv4.address`) and let you write OS-agnostic playbooks (`when: ansible_os_family == "Debian"`). Disable gathering with `gather_facts: false` when you do not need them, for a speedup. Custom facts live in `/etc/ansible/facts.d/*.fact` on the target and appear under `ansible_local.*`.

## Medium

### Q7: Where do `group_vars` and `host_vars` fit in precedence?
**Answer:** `group_vars/<group>.yml` and `host_vars/<host>.yml` are auto-loaded files that apply variables to a group or host respectively; `host_vars` wins over `group_vars` for the same host. In the full precedence order, CLI `--extra-vars` wins, then role vars, then play `vars`, then inventory vars (host_vars over group_vars), then role `defaults` at the bottom. The practical rule: put sensible overridable defaults in `roles/*/defaults/`, put fixed role behavior in `roles/*/vars/`, put environment-specific values in `group_vars/<env>.yml`, and use `--extra-vars` only for one-off overrides.

### Q8: How do Jinja2 templates work in Ansible?
**Answer:** The `ansible.builtin.template` module renders a `.j2` file through Jinja2 and copies it to the target, comparing the rendered content to the destination and reporting `changed` only when they differ. Jinja2 gives you filters (`default`, `to_json`, `regex_replace`, `map`, `selectattr`), tests (`defined`, `success`, `string`), conditionals (`{% if %}`), and loops (`{% for host in groups['web'] %}`). Templates can reference facts, hostvars, and inventory vars, so you can build an nginx upstream block that lists every host in the `web` group from `hostvars`. Re-running the play is a no-op if the rendered output is unchanged.

### Q9: What are handlers and when do they run?
**Answer:** Handlers are tasks notified by other tasks and run only when the notifier reports `changed`. They are flushed at the end of the play (or at `meta: flush_handlers`) and deduplicated — notifying `reload nginx` three times runs it once. The canonical use is restarting a service only when its config actually changed, so a converged playbook does not bounce nginx on every run. Handlers run in the order they are defined, not the order they were notified, and they do not run if the play fails before the flush point unless `force_handlers: true` is set.

### Q10: Describe the standard role structure.
**Answer:** A role is a directory with conventional subdirectories: `tasks/main.yml` (the task list), `handlers/main.yml`, `vars/main.yml`, `defaults/main.yml` (lowest-precedence overridable defaults), `files/` (static files for `copy`), `templates/` (Jinja2 templates for `template`), and `meta/main.yml` (role metadata, dependencies, supported platforms). Ansible auto-loads each file in its conventional location when the role is applied via `roles:` or `include_role`. `ansible-galaxy init roles/<name>` scaffolds the structure. Roles are the standard unit of reuse and are published to Ansible Galaxy via `requirements.yml`.

### Q11: What are collections and why did Ansible introduce them?
**Answer:** Collections are the distribution model since Ansible 2.10: versioned, namespaced bundles of modules, roles, and plugins (`community.aws`, `community.docker`, `ansible.posix`, `google.cloud`). The split from a monolithic Ansible into `ansible-core` (engine + builtins under `ansible.builtin`) plus collections lets each collection release independently instead of gating every module on the core release cycle. Install with `ansible-galaxy collection install <name>:<version>` and pin via `requirements.yml`. Reference modules by their fully qualified name (`community.docker.docker_container`) to make the source explicit and avoid collisions.

### Q12: How does ansible-vault protect secrets?
**Answer:** `ansible-vault encrypt` encrypts a YAML file or string with AES-256 using a vault password; the encrypted blob is safe to commit to git. Run playbooks with `--ask-vault-pass` or `--vault-password-file`, and for team use store the vault password in a shared secret store (Bitwarden, 1Password, AWS Secrets Manager) via a vault password client script. Vault is convenient but the secrets still live in your repo (just encrypted). For serious deployments, integrate with an external secret store at runtime via lookup plugins like `amazon.aws.aws_secret`, so secrets never touch disk in plaintext and rotation is centralized.

### Q13: What is check mode and why use it?
**Answer:** `ansible-playbook --check` is Ansible's dry run: modules simulate the change and report `changed` but do not act. `--diff` adds line-level before/after diffs for `template`, `lineinfile`, `file` mode changes, etc. It is the Ansible analog of `terraform plan` — use it in CI and before prod runs to preview the effect of a playbook. Some modules (notably `command`/`shell`) cannot check and are skipped under `--check`, so a fully check-mode-safe playbook is a sign of good module choice.

### Q14: Conditionals and loops — show the common patterns.
**Answer:** `when: ansible_os_family == "RedHat"` gates a task on a fact; `when` accepts booleans, Jinja comparisons, and lists (`when: x and y`). `loop:` iterates a list with `item`; `with_dict:` iterates a dict with `item.key`/`item.value`; `loop` plus filters (`{{ users | map(attribute='name') | list }}`) covers most older `with_*` helpers. Combine them: install a package list per OS with `loop: "{{ pkgs[ansible_os_family] }}"` gated by `when: ansible_os_family in pkgs`. `loop_control` lets you set a label and pause, useful for readable output on long loops.

## Hard

### Q15: Ansible vs Terraform — mutable vs immutable?
**Answer:** Ansible is configuration management: it mutates existing hosts in place — install a package, edit a file, restart a service — and the host accumulates state over time. Terraform is provisioning: it declares cloud resources and replaces them when the config changes, leaning toward immutable infrastructure where change means redeploy. Ansible is stateless (no central source of truth) and idempotency is per-module opt-in; Terraform is stateful (state maps config to reality) and idempotency is built-in. In practice they compose: Terraform provisions the VM/network/DB, Ansible configures the OS and deploys the app. In a fully immutable pipeline (Packer AMIs deployed by Terraform), Ansible's role shrinks to the Packer build step.

### Q16: Ansible vs Chef/Puppet — push vs pull?
**Answer:** Ansible pushes from a control node over SSH with no agents on targets; Chef and Puppet use a pull model where an agent daemon on each host periodically checks in with a central server to fetch its desired state. Agentless push means no agents to install, upgrade, or secure, and no central server to maintain, but it requires the control node to reach every target and SSH overhead limits throughput. Pull models scale better for very large fleets and work behind NAT, but they require agent infrastructure and a central server. The agentless push model is the most-cited Ansible advantage in interviews.

### Q17: Best practices for large inventories?
**Answer:** Use dynamic inventory plugins so hosts reflect the cloud in real time; split variables into `group_vars/` and `host_vars/` directories rather than inline; use roles and collections to keep playbooks small and reusable; bump `forks` (default 5) in `ansible.cfg` to parallelize across hosts; consider the `free` strategy or Mitogen for faster execution; enable pipelining in `ansible.cfg` to reduce SSH round trips; disable `host_key_checking` only in controlled CI, not prod; tag tasks so you can run partial plays during incidents; run `--syntax-check` and `--check --diff` in CI before prod; and store secrets in an external secret store rather than vault files for serious fleets.

### Q18: How would you structure Ansible code for a multi-tier app across dev/staging/prod?
**Answer:** One inventory directory per environment (`inventory/dev`, `inventory/staging`, `inventory/prod`) each with dynamic inventory plugins and `group_vars/<env>.yml` for environment-specific values; shared roles in `roles/` applied via a thin `site.yml` playbook that targets host groups (`web`, `app`, `db`); secrets pulled at runtime from a secret store keyed by environment; a `requirements.yml` pinning role and collection versions. CI runs `ansible-playbook --syntax-check` and `--check --diff` on PRs against each environment's inventory, and prod applies are gated behind a protected environment. This keeps the playbook logic DRY while letting each environment differ in inventory, variables, and secret scope.