# 17. Terraform – advanced and scenario questions

[← Back to index](README.md)

**Q155. How do you refactor a large Terraform root module into modules without destroying anything?**
Write the module, then use `moved` blocks to map every old address to its new address (`moved { from = aws_subnet.private[0]  to = module.vpc.aws_subnet.private["a"] }`), run `plan` and confirm it shows only moves and no destroy/create, apply, then delete the `moved` blocks after everyone has applied. Before `moved` existed this was `terraform state mv` by hand, which is riskier because it's not reviewed in a PR. Do it one component at a time with a state backup (S3 versioning) ready.

---

**Q156. How do you split one state file into two?**
Create the new root with its own backend, then move resources: `terraform state pull` to back up, in the new root `terraform import` or use `terraform state mv -state=old.tfstate -state-out=new.tfstate ADDR ADDR` locally and push, or on Terraform 1.7+ use `removed` blocks in the old root (with `destroy = false`) and `import` blocks in the new root so the whole operation is declarative and reviewable. Verify both roots plan clean. Replace cross-references with `terraform_remote_state` or SSM parameters.

---

**Q157. `for_each` on a set derived from another resource fails at plan time. Why and how do you fix it?**
Keys of `for_each` must be known at plan time; attributes computed during apply (like IDs of resources not yet created) aren't. Fix by keying on values you control: names, CIDRs, or the input variable keys, and look up the computed attribute inside the body (`vpc_id = aws_vpc.main[each.key].id`). Last resort, two-phase apply with `-target`, which should never be the standard workflow.

---

**Q158. How would you implement policy-as-code for Terraform in a bank?**
Three layers. Pre-plan static: `tflint` rules and `checkov`/`tfsec` for known bad patterns. Plan-time: export `terraform show -json tfplan`, evaluate with OPA/Conftest (or Sentinel in HCP Terraform) for organisation policies such as no public ingress on 22/3389, all S3 buckets encrypted with a CMK, mandatory tags, approved instance families, no `0.0.0.0/0` egress from DB tiers; fail the pipeline on deny. Runtime: AWS Config and SCPs catch anything that bypassed the pipeline. Policies live in their own repo, versioned, with unit tests, owned by the platform team with security review.

---

**Q159. Terraform provider upgrade from AWS v4 to v5 broke the plan. Approach?**
Read the provider upgrade guide for breaking changes (v5 removed deprecated arguments and changed S3 bucket sub-resources, default tags behaviour, and some attribute types). Pin the old version, upgrade in a branch, run `plan` per root and categorise: syntax changes you fix in code, in-place updates that are harmless (tag normalisation), and anything showing replacement, which you investigate and often fix with `ignore_changes` or by adopting the new resource shape via `moved`/`import`. Upgrade non-prod first, commit the lock file, roll through environments.

---

**Q160. How do you manage Terraform for 30 teams at once?**
Platform team owns a private module registry with versioned, tested modules and a pipeline template that includes plan review, policy checks and OIDC roles. Each team owns their own roots and state within their accounts; they consume modules by version. Guardrails are in policy-as-code and SCPs, not in code review of every PR. A monorepo for modules, polyrepo for application infra, or HCP Terraform/Spacelift/Atlantis for workspace management if scale justifies it. Standards doc plus `terraform-docs` so every module is self-describing.

---

**Q161. How do you handle Terraform when a resource type isn't supported by the provider?**
Options in order: check for a newer provider version or the `awscc` (Cloud Control) provider; use `aws_cloudformation_stack` to wrap a CloudFormation resource; the `aws_cloudcontrolapi_resource` resource; as a last resort `terraform_data` with a `local-exec` calling the CLI, accepting that it's not idempotent or planned. Document the debt and revisit when the provider catches up.

---

**Q162. Someone ran `terraform destroy` on prod by mistake. How did it happen and how do you make it impossible?**
It happened because a human had apply-level credentials and no guardrails. Prevent: humans only get plan permissions, apply runs from CI with a saved plan; `prevent_destroy` on stateful resources; a plan step that fails when destroys exceed zero for prod unless an explicit variable is set; S3 bucket and RDS deletion protection at the AWS level; MFA delete and Object Lock on the state bucket; separate accounts so a mistake in non-prod can't touch prod. Recovery: restore RDS from snapshot or PITR, S3 versions, re-apply infra from code; this is why backups and IaC matter.

---

**Q163. Explain how you'd manage Kubernetes resources with Terraform, and the pitfalls.**
`kubernetes` and `helm` providers can create namespaces, RBAC, add-ons and base charts right after the EKS cluster in the same or a downstream root. Pitfalls: the provider needs cluster credentials at plan time, so the cluster must exist before the root that configures it (separate root, or `depends_on` tricks that break on destroy); CRDs installed by a chart aren't visible to the provider in the same apply; drift when teams `kubectl apply` over Terraform-managed objects. Common split: Terraform for cluster, node groups, add-ons, IAM and namespaces; Argo CD or Flux for application workloads.

---

**Q164. What's the difference between `terraform refresh`, `plan -refresh-only`, and `apply -refresh-only`?**
`refresh` (deprecated) updates state from real infrastructure immediately with no review. `plan -refresh-only` shows what would change in state if you accepted reality, without proposing infra changes. `apply -refresh-only` writes those state updates after confirmation. Use the refresh-only pair to reconcile drift you've decided to accept, for example a tag added by a Config remediation.

---

**Q165. How do you use Terraform with multiple AWS accounts in one run?**
Aliased providers each with `assume_role` into a target account, and pass them to modules via `providers = { aws = aws.workload }`. Typical hub-and-spoke use: the network account creates a Transit Gateway share, the workload account's provider accepts the RAM share and attaches its VPC, in one apply. Keep the pipeline role in the shared-services account with trust relationships into each target account, scoped with external IDs and session tags.

---

**Q166. How do you keep Terraform runs fast on a large estate?**
Split state by component so each plan touches hundreds, not thousands, of resources; `-parallelism` tuned per API throttling; provider plugin cache in CI; `-refresh=false` for quick syntax iterations in dev (never for the real plan); `-target` only for emergencies; move slow data sources (huge `aws_ami` searches) to SSM parameters; use `terraform plan -detailed-exitcode` to skip apply when nothing changed.

---

**Q167. Explain Terraform Cloud/HCP Terraform or Atlantis versus plain CI.**
Plain CI (GitHub Actions/Jenkins) is flexible and you already have it, but you build plan-comment, locking, and RBAC yourself. Atlantis is a self-hosted PR-driven Terraform runner: `atlantis plan`/`apply` comments, per-directory locking, policy checks; good middle ground. HCP Terraform/Spacelift/env0 add workspaces, remote state, run queues, Sentinel/OPA, drift detection, cost estimation, private registry, SSO, and audit, at a per-resource or per-user price. For a bank I'd evaluate a managed platform for the governance features and keep plain CI as the fallback.

---

**Q168. A module output changed type and broke consumers. How do you version modules safely?**
Semantic versioning on module tags: patch for fixes, minor for backward-compatible additions, major for breaking changes like renamed variables or output type changes. Consumers pin `version = "~> 2.1"` so they get patches automatically but never a major. Deprecate before removing: keep the old output alongside the new one for one minor release with a `description` noting deprecation. Changelog per module, `terraform test` covering outputs, and a CI matrix that plans the examples against the module on every PR.

[← Back to index](README.md)
