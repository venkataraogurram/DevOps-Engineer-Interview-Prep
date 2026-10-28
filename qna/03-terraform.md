# 3. Terraform

[← Back to index](README.md)

**Q17. Difference between `terraform plan` and `terraform apply`?**
Plan refreshes state against real infrastructure, compares with the configuration, and shows what would be created, updated, replaced or destroyed; it changes nothing. Apply executes those changes. In CI I run `plan -out=tfplan`, have it reviewed, then `apply tfplan`, so what runs is exactly what was approved and not a fresh plan that might differ if something changed in between.

---

**Q18. What is Terraform state and why is it needed?**
State is Terraform's record mapping resources in config to real resource IDs plus their last-known attributes. It's needed because many resource attributes are only known after creation (IDs, ARNs), to detect drift, to compute dependencies for destroy ordering, and for performance so Terraform doesn't query everything every time. It can contain secrets, so it must be encrypted and access-controlled.

---

**Q19. How do you manage state in a team?**
Remote backend on S3 with encryption, versioning and Block Public Access; state locking via a DynamoDB table with a `LockID` hash key, or S3 native locking with `use_lockfile = true` on Terraform 1.10 and later. One state file per component per environment, so network, EKS and each app are separate and the blast radius is small. Applies run only from the CI role; humans get a read-only plan role. We never hand-edit state; we use `state mv`, `state rm`, `import`, and `moved` blocks.

---

**Q20. What happens if two engineers run `apply` simultaneously without locking?**
Both read the same state, both make API calls, and the second one to finish writes its state over the first. The state now disagrees with reality; the next plan shows phantom creates or destroys, and you may end up with orphaned resources. Locking makes the second run fail fast with "Error acquiring the state lock". That's why locking plus CI-only apply is non-negotiable.

---

**Q21. What are workspaces and when wouldn't you use them?**
Workspaces give you multiple state files for the same configuration under the same backend, selected with `terraform workspace select`. They're fine for short-lived variants like per-developer sandboxes. I wouldn't use them to separate prod from non-prod: same backend, same credentials, same provider configuration, and it's one wrong `workspace select` away from applying dev changes to prod. I prefer a directory and backend per environment, in separate AWS accounts.

---

**Q22. Explain modules and your best practices.**
A module is a reusable unit of Terraform with inputs, resources and outputs. Best practices: small and single-purpose (a VPC module, an EKS module, an app security group module), versioned by git tag or private registry and pinned by consumers, typed and validated input variables with descriptions, outputs for everything a consumer might need, no `provider` blocks inside reusable modules so callers pass providers, an `examples/` folder, `terraform-docs` for the README, and tests with `terraform test` or Terratest.

---

**Q23. `count` vs `for_each`?**
Both create multiple instances. `count` indexes by position, so removing an item in the middle shifts indexes and Terraform wants to destroy and recreate everything after it. `for_each` keys by a map key or set value, so adding or removing one item only affects that item. I use `count` only for the boolean on/off pattern (`count = var.enabled ? 1 : 0`) and `for_each` for collections.

---

**Q24. What is a `dynamic` block?**
It generates repeated nested blocks from a collection, for example multiple `ingress` blocks in a security group from a list of rule objects. It's useful, but I prefer standalone rule resources like `aws_vpc_security_group_ingress_rule` with `for_each` where they exist, because each rule becomes its own resource with a clean diff instead of the whole SG churning.

---

**Q25. What do `lifecycle` arguments do?**
`create_before_destroy` builds the replacement before destroying the old resource, for zero-downtime on things like launch templates and security groups. `prevent_destroy` makes plan fail if the resource would be destroyed, for databases and state buckets. `ignore_changes` stops Terraform fighting something managed elsewhere, like an ASG desired capacity controlled by autoscaling. `replace_triggered_by` forces replacement when another resource changes. Preconditions and postconditions assert invariants.

---

**Q26. How do you handle secrets in Terraform?**
Never in committed `.tfvars`. Read at apply time from Secrets Manager or SSM via data sources, or inject through `TF_VAR_` from the CI secret store. Mark variables and outputs `sensitive`, and on Terraform 1.10+ use ephemeral values so they never land in state. Remember state still holds most attributes, so the backend is encrypted with KMS and access is restricted. For things like RDS, let AWS manage the password (`manage_master_user_password`) so Terraform never sees it.

---

**Q27. A resource was created manually in the console. How do you bring it under Terraform?**
Write the resource block, then `terraform import ADDRESS ID`, or on 1.5+ an `import` block with `plan -generate-config-out` to draft the config. Iterate on the config until `plan` shows no changes. The risk is importing with a config that doesn't match: the next apply "fixes" the resource to match your code, so I always confirm a clean plan before merging.

---

**Q28. Terraform wants to replace a production resource. What do you do?**
Read the plan for the attribute marked "forces replacement". Decide if the change is needed. If it is, use `create_before_destroy`, or for stateful resources plan a migration: RDS snapshot and restore or Blue/Green deployment, ASG instance refresh via launch template version. If it's a refactor or rename, use `moved` blocks so the address changes without replacement. If the attribute is managed elsewhere, `ignore_changes`. Never apply a surprise replacement in prod.

---

**Q29. How do you detect and handle drift?**
Scheduled `terraform plan` in CI against each environment; alert on a non-empty plan. Config rules and CloudTrail show who changed what. Policy: no console changes in prod; break-glass changes must be followed by a PR within the day. To accept a drift, `apply -refresh-only` updates state without touching infra; to revert it, a normal apply.

---

**Q30. How do you structure a Terraform repository for multiple environments?**
`modules/` for reusable modules, `envs/dev`, `envs/staging`, `envs/prod` each with its own backend config and tfvars and composing the modules, pipelines per environment with path filters. Versions pinned: Terraform via `.terraform-version`, providers via `required_providers` and the committed lock file, modules by tag. Prod changes require a reviewed plan and a manual approval environment.

---

**Q31. What is `terraform taint` / `-replace`?**
`taint` marked a resource for recreation on the next apply; it's deprecated in favour of `terraform apply -replace=ADDRESS`, which does the same thing in a single planned step. Used when a resource is unhealthy but its config is fine, such as a corrupted instance.

---

**Q32. Explain `depends_on` and when it's a smell.**
Terraform infers dependencies from references. `depends_on` adds an explicit one for cases with no attribute reference, like an IAM policy attachment that must exist before an EKS node group joins. It's a smell when overused because it hides the real relationship and can cause unnecessary replans; prefer referencing an attribute.

---

**Q33. What are provisioners and why avoid them?**
`local-exec`, `remote-exec` and `file` run scripts during create or destroy. They're not idempotent, not visible in plan, taint the resource on failure, and require network access. I use cloud-init user data, SSM documents, or Packer-baked images instead, and `terraform_data` with triggers if I really need a hook.

---

**Q34. How do you test Terraform?**
Static: `fmt -check`, `validate`, `tflint`, `checkov` or `tfsec`, and OPA/Conftest on the plan JSON for policies like "no 0.0.0.0/0 on port 22" or "all buckets encrypted". Unit-ish: `terraform test` with `.tftest.hcl` files asserting on plan values. Integration: Terratest or `terraform test` with `command = apply` in an ephemeral account, then destroy. Plus the PR plan comment for human review.

---

[← Back to index](README.md)
