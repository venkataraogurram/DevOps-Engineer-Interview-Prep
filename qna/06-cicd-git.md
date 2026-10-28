# 6. CI/CD and Git

[← Back to index](README.md)

**Q55. Describe your ideal CI/CD pipeline for an application on EKS.**
On pull request: lint, unit tests, SAST, secret scan, Terraform plan posted as a comment. On merge to main: build the container once, tag with the git SHA, scan with Trivy, sign, push to ECR with tag immutability. Deploy to dev with Helm, run integration tests, promote the same image to staging, then a manual approval tied to a change ticket, deploy to prod with rolling or canary, smoke tests, and automatic rollback if alarms fire. All AWS access via OIDC-assumed roles with no stored keys. Every stage leaves evidence for audit.

---

**Q56. What rollback strategies do you know?**
Redeploy the previous immutable artifact; `kubectl rollout undo` or ASG instance refresh back to the previous launch template version; blue/green where you switch the load balancer back; canary where you shift traffic to zero; feature flags that turn a change off without a deploy. The database is the hard part: expand/contract migrations so the schema is compatible with the previous release, and snapshots before any migration.

---

**Q57. Blue/green vs canary vs rolling?**
Rolling replaces instances gradually within one environment; cheap, but two versions coexist and rollback is a new roll. Blue/green runs two complete environments and switches traffic at once; instant rollback, double cost during the switch. Canary sends a small percentage to the new version and promotes on metrics; the safest for user impact, needs good observability and traffic-splitting support like ALB weighted target groups or Argo Rollouts.

---

**Q58. GitFlow vs trunk-based development?**
GitFlow has long-lived develop, release and hotfix branches; it fits scheduled releases and formal QA sign-off but creates merge pain and slow feedback. Trunk-based has everyone merging to main in small, short-lived branches with feature flags hiding unfinished work; it enables continuous delivery. For a bank I'd recommend trunk-based with release branches cut for compliance evidence, rather than environment branches, which promote by merge instead of by artifact.

---

**Q59. How do you secure a CI/CD pipeline?**
No static cloud credentials: OIDC federation to roles scoped per repo, branch and environment. Branch protection with required reviews and status checks. Secrets in a vault, masked in logs, scanned for in commits. Pin third-party actions by SHA. Runners inside the VPC for private resources, ephemeral. Signed commits and signed images. Separate plan and apply roles. The pipeline is the only identity with prod write access.

---

**Q60. Jenkins: how do you structure pipelines for 50 teams?**
Jenkinsfile per repo using declarative syntax, with a Shared Library that provides standard stages (`standardBuild()`, `deployToEks()`) so teams get the same security scans and approval gates without copying code. Ephemeral agents on Kubernetes via the Kubernetes plugin. Controller configuration as code with JCasC, credentials in the Jenkins store backed by Secrets Manager, Role Strategy for RBAC, and backups of JENKINS_HOME.

---

**Q61. Merge vs rebase?**
Merge preserves history and creates a merge commit; rebase rewrites your commits on top of the target for linear history. Rebase only on your own unpushed branches; never rebase a shared branch because it changes SHAs others have. Teams commonly squash-merge PRs for a clean main.

---

**Q62. How do you handle hotfixes in production?**
Branch from the tag that's in prod, fix, test through the same pipeline with an expedited approval, deploy, tag, then merge the fix back to main so it isn't lost. Record the change ticket. If the fix is infra, the same flow with Terraform plan review.

---

[← Back to index](README.md)
