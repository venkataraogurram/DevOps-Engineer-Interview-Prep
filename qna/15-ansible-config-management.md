# 15. Ansible and configuration management

[← Back to index](README.md)

**Q128. Ansible vs Terraform: what is each for, and how do they work together?**
Terraform provisions infrastructure declaratively and keeps state: VPCs, instances, clusters, IAM. Ansible configures what runs on that infrastructure: packages, files, services, application deployment; it's agentless over SSH and effectively stateless. Together: Terraform creates the instances and emits an inventory (or Ansible uses the `aws_ec2` dynamic inventory on tags), Ansible configures them. The better pattern for immutability is Packer running Ansible at image-build time, so Terraform only deploys finished AMIs.

---

**Q129. How does Ansible achieve idempotency, and where does it break?**
Modules check current state before changing it and report `changed` only when they did something: `package state=present` won't reinstall, `file state=directory` won't recreate, `lineinfile` only edits when the line differs. It breaks with `command` and `shell`, which always run; guard them with `creates`/`removes`, `when` on a registered fact, or `changed_when`. It also breaks when a template includes a timestamp or random value, which makes every run "changed".

---

**Q130. Explain handlers.**
A handler is a task that runs only when notified by a task that changed something, and it runs once at the end of the play even if notified multiple times. Classic use: restart or reload nginx after any config template changes. `meta: flush_handlers` forces them to run mid-play. Handlers are skipped if a later task fails, unless `force_handlers: true`.

---

**Q131. Variable precedence in Ansible?**
From lowest to highest: role defaults, inventory group vars (all, then parent groups, then child groups), host vars, play vars and vars_files, role vars, block and task vars, `set_fact` and registered variables, and finally extra vars from `-e`, which always win. The practical rule: put tunables in role defaults, environment differences in group_vars, and use `-e` only for one-off overrides from the pipeline.

---

**Q132. How do you do a rolling deployment with Ansible without downtime?**
`serial: 25%` or a fixed number so only a slice of hosts is touched at a time. In `pre_tasks`, deregister the host from the load balancer target group (`amazon.aws.elb_target` with `delegate_to: localhost`) and wait for connection draining; deploy and restart; run a local health check with `uri` and `until`; in `post_tasks` re-register and wait for the target to be healthy. `max_fail_percentage` aborts the play if too many hosts fail, so a bad build doesn't take out the fleet.

---

**Q133. How do you handle secrets in Ansible?**
Ansible Vault for anything in git: encrypted vars files or `encrypt_string` for individual values, with the vault password from a CI secret or `--vault-id`. Better for AWS: look up at runtime with `lookup('amazon.aws.aws_ssm', ...)` or the Secrets Manager lookup so nothing sensitive is in the repo. Mark tasks `no_log: true` so values don't print. Never put secrets in inventory or `-e` on the command line where they land in shell history.

---

**Q134. How does Ansible connect to EC2 hosts that have no SSH open?**
Use the `community.aws.aws_ssm` connection plugin: it runs commands through SSM Session Manager and uses an S3 bucket for file transfer, so no port 22, no key pairs, and all access is logged in CloudTrail. The host needs the SSM agent and an instance profile with `AmazonSSMManagedInstanceCore`; the controller needs `ssm:StartSession` and S3 access. Dynamic inventory from the `aws_ec2` plugin filtered by tags gives the host list.

---

**Q135. Roles vs playbooks vs collections?**
A playbook orchestrates: which hosts, in what order, with which roles and tasks. A role is a reusable, self-contained unit with a standard layout: tasks, handlers, defaults, vars, templates, files, meta. A collection packages roles, modules, and plugins for distribution via Galaxy or a private hub, like `amazon.aws` and `kubernetes.core`. Structure: thin playbooks, fat roles, versioned collections pinned in `requirements.yml`.

---

**Q136. How do you test Ansible content?**
`ansible-lint` for style and known anti-patterns; `--syntax-check`; `--check --diff` as a dry run against a staging inventory; Molecule to spin up a container or EC2 instance, apply the role, run verification tasks or Testinfra assertions, and tear down; all of this in CI on every PR. Idempotence test: run twice, second run must report zero changes.

---

**Q137. Ansible vs Chef/Puppet/Salt: when would you pick something else?**
Ansible is agentless, push-based, YAML, low barrier; great for orchestration and moderate fleets. Puppet and Chef are agent-based and pull-based with continuous enforcement every 30 minutes, which suits very large long-lived mutable fleets with strict drift control. Salt is fast with an event bus for reactive automation. In an immutable-AMI, container-first environment the question matters less: configuration happens at image build time, and Ansible is the common choice there because Packer integrates with it directly.

---

**Q138. What's the difference between `include_tasks` and `import_tasks`?**
`import_*` is static: processed at playbook parse time, so tags and `when` apply to every imported task, and you can't loop over it. `include_*` is dynamic: evaluated at runtime, supports loops and conditional inclusion, but tags on the include don't propagate to the included tasks. Rule of thumb: import for structure, include when you need a loop or runtime decision.

---

**Q139. How would you manage configuration for 500 EC2 instances across accounts without Ansible?**
SSM: State Manager associations apply SSM documents on a schedule per tag, Run Command for ad hoc fleet operations, Patch Manager for OS patching, Parameter Store for configuration, Distributor for agents, and Inventory plus Compliance for reporting, all through cross-account delegated admin. Combined with immutable AMIs from Packer, the in-place configuration surface stays small.

[← Back to index](README.md)
