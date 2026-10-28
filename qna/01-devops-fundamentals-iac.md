# 1. DevOps fundamentals and IaC

[← Back to index](README.md)

**Q1. What is DevOps, in your words?**
A culture and set of practices where the people who build software also own how it runs. In practice that means shared responsibility for the pipeline and production, small frequent changes, automation of everything repeatable, and feedback loops from monitoring back into development. I measure it with the DORA metrics: deployment frequency, lead time for change, change failure rate, and time to restore. In a bank, automation also doubles as the audit trail, because every change goes through a reviewed, logged pipeline instead of a person on a console.

---

**Q2. Why Infrastructure as Code? What are the downsides?**
Benefits: infrastructure becomes reviewable in pull requests, versioned in git, reproducible across environments, diff-able before change (`terraform plan`), and self-documenting. It enables ephemeral test environments and drift detection. Downsides: a learning curve, state management complexity, and the same automation that creates a VPC in two minutes can destroy it in two minutes, so you need guardrails: plan review, `prevent_destroy`, least-privilege pipeline roles, and separate state per environment.

---

**Q3. Mutable vs immutable infrastructure?**
Mutable: you patch and reconfigure running servers (Ansible, SSM Patch Manager). Immutable: you never change a running instance; you build a new image (Packer), roll it out, and terminate the old ones. Immutable removes drift, makes rollback a matter of pointing back at the previous AMI, and is easier to audit. Cost is a build pipeline and image lifecycle management. For a bank I'd default to immutable for compute and use configuration management only for the image build step.

---

**Q4. CloudFormation vs Terraform, which would you choose and why?**
CloudFormation: native, AWS holds the state, built-in rollback and drift detection, StackSets for org-wide deployment, hooks for policy. Terraform: multi-provider (AWS plus GitHub, Datadog, Kubernetes, Vault), richer module ecosystem, clearer plan output, faster coverage of new services, but you own state and locking. I've supported CloudFormation deeply, and for this JD I'd standardise application teams on Terraform because of the module ecosystem and multi-provider reach, while keeping CloudFormation/StackSets for organisation guardrails coming out of Control Tower.

---

**Q5. Declarative vs imperative?**
Declarative: describe the end state (Terraform, Kubernetes manifests); the tool computes the steps. Imperative: specify the steps (shell scripts, `aws` CLI). Declarative tools are idempotent by construction and can show a diff before acting; imperative scripts need you to build idempotency in yourself. I use declarative for infrastructure and imperative only for glue.

---

**Q6. What is idempotency and why does it matter in automation?**
Running the same operation N times gives the same result as running it once. Terraform apply on unchanged code does nothing; a Bash script that runs `aws ec2 run-instances` unconditionally creates a new instance each time. It matters because pipelines retry, people re-run jobs, and cron overlaps. I design scripts to check current state before acting and to tag what they create so a re-run finds and skips it.

---

[← Back to index](README.md)
