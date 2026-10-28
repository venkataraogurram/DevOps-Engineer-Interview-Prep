# IDFC FIRST Bank – Platform/DevOps Engineer – Interview Prep Guide

## 1. What they are actually hiring for

Read the JD past the keywords:

| JD line | What it means in practice | What you must be able to show |
|---|---|---|
| "primary focus on AWS cloud services" | Bank workloads on AWS: VPC design with strict segmentation, IAM least privilege, encryption everywhere, audit trail | Talk VPC/IAM/KMS/CloudTrail fluently; know what a regulated environment adds (no public subnets for app tier, SCPs, Config rules) |
| "automated provisioning … Terraform and Packer" | Golden AMI pipeline (Packer) + infra pipeline (Terraform). Packer is named twice; expect at least one question | A Packer HCL2 template from memory and how its AMI ID flows into Terraform |
| "As a DevOps Lead" | They want someone who owns standards: module structure, state layout, review process | Opinions on repo layout, module versioning, state per environment, PR review for infra |
| "CI/CD pipelines … reliable and efficient deployment" | Jenkins is the common default in Indian banks; GitLab CI also common. Rollback matters more than speed | Pipeline stages, approval gates, rollback strategies, artifact immutability |
| "Performance Monitoring and Optimization" | CloudWatch first, ELK/Prometheus second. Cost optimization is explicitly listed | Alarms → runbooks; right-sizing; Savings Plans; S3 lifecycle |
| Secondary: Docker/K8s, microservices, serverless | Nice-to-have, but the recruiter's prep sheet makes K8s a coding-round topic, so treat it as mandatory | Deployment/Service/Secret/probe manifests without reference |

**Bank-specific flavour to weave in:** change management (CAB approvals, maintenance windows), segregation of duties (pipeline deploys, humans don't), audit evidence (CloudTrail, Config, Terraform plan output attached to the change ticket), data residency (ap-south-1 / ap-south-2), encryption at rest with CMKs, private connectivity (VPC endpoints, no NAT for sensitive tiers, Direct Connect to DC).

## 2. How to position your background

You are a Cloud Support Engineer at AWS (Developer Tools queue: Elastic Beanstalk, EKS, CloudFormation, Control Tower, CodePipeline/CodeBuild, Kiro, DevOps Agent) with ~4+ years total, Infosys before that, and hands-on projects (ShopStack on EKS with Terraform + GitHub Actions OIDC; Expense Tracker with SSO on AWS; Three-Tier architecture).

Lead with what the JD wants and translate your support work into platform-engineer language:

- **"I debug other people's infrastructure for a living."** Every day you read someone else's CloudFormation/Terraform/K8s manifests, find the defect, reproduce it, and prove the fix. That is exactly the coding-round skill ("read half-baked code, complete it, debug it").
- **Breadth of failure modes.** You have seen hundreds of ways EKS, ASG, ALB, IAM and CloudFormation break. Use concrete ones: IAM trust-policy mistakes, SG rules that don't help because there's no listener, CFN custom-resource failures, rolling-update ordering on instance replacement, health-check path mismatches.
- **IaC ownership.** ShopStack: Terraform-built EKS, ALB ingress, Pod Identity, probes, HPA 2→6 pods in under 40 s, GitHub Actions with OIDC (no long-lived keys), CloudWatch tracing. That is a real end-to-end pipeline; walk through it in 3 minutes (see §6).
- **Gap to own honestly: Packer.** Say: "I've used Packer for golden AMIs in my own lab, not at bank scale. Here's the template, here's how I'd wire it into a pipeline, and here's how the AMI ID reaches Terraform." Then say it. Being specific beats pretending.
- **Jenkins.** If asked and you've mainly used GitHub Actions/CodePipeline: map concepts (Jenkinsfile stages ≈ workflow jobs, agents ≈ runners, shared libraries ≈ reusable workflows). Know a declarative Jenkinsfile skeleton (in `06`).

Say "4+ years" consistently; don't round up.

## 3. Likely round structure

Based on the recruiter's sheet:

1. **Technical screen (45–60 min)** – conceptual Q&A across AWS, Terraform, K8s, CI/CD, scripting. Expect "difference between…" and "how would you…" questions.
2. **Coding / hands-on round (60–90 min)** – shared screen or take-home. Partially written Terraform (provider, VPC, subnets, SGs, module calls, remote state) and/or Kubernetes (Deployment + Secret via `secretKeyRef`, Service, ConfigMap, probes, limits, rollout verification). See `09`.
3. **Managerial / lead round** – project walkthroughs, incident stories, how you drive standards, stakeholder communication, on-call, change management.
4. **HR** – notice period, CTC, relocation (IDFC tech is largely Mumbai/Chennai/Bengaluru; confirm location).

## 4. Question bank with answer outlines

Answer outlines are deliberately short. Expand with one real example each.

### 4.1 DevOps fundamentals

**Q: What does DevOps mean to you beyond tooling?**
Shared ownership of the delivery pipeline and production outcomes; measure with DORA metrics (deployment frequency, lead time, change failure rate, MTTR); automate the toil; make change small and reversible. In a bank, add: automation is also the audit trail.

**Q: Why IaC?**
Repeatable, reviewable (PRs), versioned, diff-able (`plan`), testable, self-documenting, enables ephemeral environments, drift detection. Cost: learning curve, state management, you can destroy things fast too, so guardrails (plan review, `prevent_destroy`, least-privilege pipeline roles).

**Q: CloudFormation vs Terraform?**
CFN: native, no state to manage (AWS holds it), drift detection built in, StackSets for org-wide rollout, rollback on failure by default. Terraform: multi-cloud/multi-provider (Datadog, GitHub, K8s), faster provider coverage, modules ecosystem, plan is clearer, but you own the state and its locking. You've supported CFN deeply; say which you'd pick for the bank and why (Terraform for app teams because of the ecosystem and the JD; CFN/StackSets for org guardrails from Control Tower).

### 4.2 AWS

**Q: Design a VPC for a 3-tier banking app.**
Multi-AZ (3 AZs in ap-south-1). Public subnets only for ALB/NAT. Private app subnets, isolated DB subnets (no route to NAT). NACLs coarse, SGs fine-grained with SG-to-SG references. VPC endpoints for S3, DynamoDB (gateway), and interface endpoints for SSM, ECR, CloudWatch Logs, STS, KMS so app tier needs no internet. Flow Logs on. Transit Gateway or Direct Connect to on-prem. Separate accounts per environment via Organizations/Control Tower.

**Q: IAM role vs user vs policy; how do you avoid long-lived keys?**
Roles are assumed via STS for temporary creds; users have long-lived keys (avoid). Instance profiles for EC2, IRSA/Pod Identity for EKS, OIDC federation for GitHub Actions/GitLab, Identity Center for humans. Permission boundaries + SCPs for guardrails. Access Analyzer to find unused access.

**Q: ALB vs NLB?**
ALB L7: path/host routing, WAF, OIDC auth, HTTP health checks, target groups per service. NLB L4: static IPs/EIPs, TLS passthrough, extreme throughput, PrivateLink. Bank APIs behind ALB+WAF; PrivateLink exposure via NLB.

**Q: How do you reduce AWS cost?**
Right-size from CloudWatch/Compute Optimizer, Graviton, Savings Plans for the stable baseline, Spot for stateless/batch, S3 lifecycle to IA/Glacier, delete unattached EBS and old snapshots, gp2→gp3, schedule non-prod off-hours (Instance Scheduler), tag everything and use Cost Explorer/Budgets with anomaly detection.

**Q: How do you secure S3?**
Block Public Access at account level, bucket policies with `aws:SecureTransport` deny, SSE-KMS with CMK, versioning + MFA delete for critical buckets, Object Lock for audit logs, access logging, VPC endpoint policy restricting to known buckets.

**Q: RDS Multi-AZ vs read replica?**
Multi-AZ = synchronous standby for HA/failover (DNS flips, ~60–120 s), not for reads. Read replica = async, scales reads, can be cross-region, promotable for DR. Aurora adds shared storage, fast failover, up to 15 replicas.

**Q: Lambda cold start, concurrency, when not to use it?**
Cold start = init of new execution environment; mitigate with provisioned concurrency, smaller packages, SnapStart (Java). Reserved concurrency caps a function; account limit shared. Avoid for long-running (>15 min), stateful, or latency-critical constant-load workloads where containers are cheaper.

### 4.3 Terraform (highest weight – see `03` for depth)

**Q: `plan` vs `apply`?**
`plan` refreshes state, computes diff (create/update/replace/destroy), writes optional plan file; no changes. `apply` executes a plan (or plans+applies). Best practice: `plan -out=tfplan` in CI, review, `apply tfplan` so what was approved is exactly what runs.

**Q: How do you manage state in a team?**
Remote backend (S3) with encryption and versioning, locking (DynamoDB table with `LockID` hash key, or S3 native locking with `use_lockfile = true` on Terraform ≥1.10), one state per environment/component (blast radius), never edit state by hand, `terraform state mv/rm/import` for surgery, restrict who can `apply` (CI role only), state contains secrets so treat the bucket as sensitive.

**Q: What happens if two people apply at once without locking?**
Both read the same state, both make changes, the second write overwrites the first; state no longer matches reality. Next plan shows phantom creates/destroys. Fix: locking + CI-only applies.

**Q: Module best practices?**
Small, single-purpose, versioned via git tags or registry; inputs with validation and descriptions; outputs for everything consumers need; no provider blocks inside reusable modules (pass via `providers`); pin provider versions in root; examples/ and tests.

**Q: Workspaces – when yes, when no?**
Fine for lightweight variants of the same config (feature branches). Not for prod vs non-prod in a bank: same backend, same credentials, easy to apply to the wrong one. Prefer separate directories/backends/accounts per environment.

**Q: How do you handle a resource created manually?**
`terraform import` (or `import {}` block ≥1.5 with `-generate-config-out`), then `plan` until no diff. Explain the risk of importing without matching config (next apply "fixes" it).

**Q: Secrets in Terraform?**
Never in `.tfvars` committed to git. Read from Secrets Manager/SSM at apply time (`data "aws_secretsmanager_secret_version"`) or pass via `TF_VAR_` from the CI vault. Mark `sensitive = true`. Remember state still holds them → encrypt and restrict the backend.

**Q: Terraform says a resource must be replaced; how do you avoid downtime?**
Read the plan for the forcing attribute; `lifecycle { create_before_destroy = true }`; for ASGs use launch template versions + instance refresh; for RDS use blue/green or snapshots; `ignore_changes` for attributes managed elsewhere; `moved {}` blocks for refactors so renames don't destroy.

### 4.4 Kubernetes (see `05`)

**Q: Zero-downtime deployment?**
Deployment with `RollingUpdate` (`maxSurge: 25%`, `maxUnavailable: 0`), readiness probe gating Service endpoints, `preStop` sleep + graceful shutdown handling SIGTERM, `terminationGracePeriodSeconds`, PodDisruptionBudget, multiple replicas spread across nodes (`topologySpreadConstraints`), and ALB deregistration delay aligned. Verify with `kubectl rollout status`; revert with `kubectl rollout undo`.

**Q: Secure a cluster?**
Private API endpoint, IAM auth (EKS access entries), RBAC least privilege with namespaces, Pod Security Standards (`restricted`), no root containers, read-only root FS, NetworkPolicies default-deny, IRSA/Pod Identity instead of node roles, image scanning (ECR), signed images, Secrets encrypted with KMS envelope encryption, audit logging on, limit `kubectl exec`.

**Q: Liveness vs readiness vs startup probe?**
Liveness restarts the container; readiness removes the pod from Service endpoints without restarting; startup probe delays the others for slow boots. Common mistake: liveness probe hitting a dependency (DB) → cascading restarts.

**Q: Requests vs limits; what happens on exceeding each?**
Requests drive scheduling and QoS class; CPU limit → throttling; memory limit → OOMKilled. Set requests realistically, consider no CPU limit for latency-sensitive apps, always set memory limit.

**Q: How does a Service find pods? What is Ingress?**
Label selector → Endpoints/EndpointSlices; kube-proxy (iptables/IPVS) programs ClusterIP → pod IPs. Ingress is L7 routing implemented by a controller (AWS Load Balancer Controller creates an ALB; nginx-ingress). Gateway API is the successor.

### 4.5 CI/CD (see `06`)

**Q: Walk through your ideal pipeline for an app on EKS.**
PR: lint, unit tests, SAST, `terraform plan` comment. Merge to main: build image once, tag with git SHA, scan (Trivy/ECR), push to ECR, sign. Deploy dev → integration tests → staging → manual approval (CAB evidence) → prod with rolling/canary → smoke tests → auto-rollback on failed health. Everything via OIDC-assumed roles; no static keys. Artifacts immutable; deploy promotes the same image.

**Q: Rollback strategies?**
Redeploy previous immutable artifact (fastest, if DB compatible); blue/green switch back; canary abort; feature flags for app-level rollback; DB: expand/contract migrations so schema is backwards-compatible for one release. `kubectl rollout undo`; CodeDeploy automatic rollback on alarm.

**Q: GitFlow vs trunk-based?**
GitFlow: long-lived develop/release/hotfix branches; suits scheduled releases and regulated change windows but merge pain. Trunk-based: short-lived branches, feature flags, CI on every commit; higher deploy frequency. Banks often GitFlow-ish for release trains; argue for trunk-based + release branches cut for compliance.

### 4.6 Scripting (see `07`)

Expect: "parse this log and count 5xx per endpoint", "find EC2 instances without a tag and stop them", "rotate old snapshots", "check disk usage and alert". Practice writing them in both Bash and Python.

### 4.7 Monitoring

**Q: How do you monitor a service?**
Golden signals: latency, traffic, errors, saturation. CloudWatch metrics + alarms (composite alarms to reduce noise), structured JSON logs → Logs Insights, X-Ray/OTel traces, dashboards per service, alarms → SNS → PagerDuty/Opsgenie with a runbook link. SLOs and error budgets. For EKS: Container Insights or Prometheus + Grafana, kube-state-metrics.

**Q: CloudWatch vs ELK?**
CloudWatch: native, zero-ops, Logs Insights, metric filters, cheaper to run but query cost per GB scanned. ELK/OpenSearch: rich full-text search, Kibana dashboards, long retention economics, needs operating (or use OpenSearch Service). Many banks ship CloudWatch → Kinesis Firehose → OpenSearch/Splunk for central SIEM.

## 5. STAR story bank (write yours in first person, 60–90 seconds each)

Prepare at least these six; map each to a JD bullet.

1. **Diagnosed a production outage from other people's IaC** (CloudFormation/Terraform). Example from support: Custom::CDKBucketDeployment failure that looked like KMS but was a Lambda runtime version drift; or an Elastic Beanstalk rolling update terminating instances before launching. Point: read the actual log before the obvious theory.
2. **Built end-to-end IaC + CI/CD** – ShopStack: Terraform EKS, ALB controller, Pod Identity, probes, HPA, GitHub Actions OIDC. Include the three bugs you diagnosed while building it.
3. **Security hardening** – replaced static keys with OIDC role assumption; least-privilege IAM for pipeline; SG that opened 3389/22 to 0.0.0.0/0 and how you closed it.
4. **Cost or performance optimization** – a metric. (Resume: 30% fewer deployment failures; Infosys numbers.) Explain how you measured it.
5. **Cross-team communication under pressure** – an SLA-bound case where you had to tell a customer an uncomfortable truth (platform bug, no ETA) and still kept trust. Maps to "Collaboration and Communication".
6. **Driving a standard** – a time you wrote a runbook/template that others adopted; or how you'd introduce Terraform module standards at a bank (repo layout, versioning, review).

Structure: Situation (1 line), Task (1 line), Action (3–4 lines, your decisions), Result (metric or concrete outcome), Learning (1 line).

## 6. The 3-minute ShopStack walkthrough (rehearse)

1. **Goal**: production-style microservice on EKS, fully from code.
2. **Terraform**: root module per environment, S3 backend + locking, VPC module (3 AZ, private nodes), EKS module (managed node groups, access entries), ALB controller via Helm provider, Pod Identity association for the app's S3/DynamoDB access.
3. **Kubernetes**: Deployment with readiness/liveness, requests/limits, HPA (CPU target), Ingress → ALB, Secrets from AWS Secrets Manager (CSI driver or external-secrets), ConfigMap for non-secret config.
4. **Pipeline**: GitHub Actions; OIDC → IAM role; build → Trivy scan → push ECR tagged by SHA → `kubectl set image`/Helm upgrade → `rollout status` gate → smoke test.
5. **Observability**: CloudWatch Container Insights + X-Ray; HPA scaled 2→6 pods in under 40 s under load.
6. **Three things that broke and what they taught you** (pick from: health-check path mismatch, Pod Identity trust, image pull from private ECR).

## 7. Questions to ask them

- What does the current AWS landing zone look like: Control Tower/Organizations, number of accounts, how are guardrails enforced?
- How is Terraform state managed today, and who can apply to prod?
- Is Packer already producing golden AMIs, or is that greenfield? What's the patching cadence requirement from InfoSec?
- Which CI tool is the standard, and how does CAB approval integrate with the pipeline?
- What's the on-call model and the current MTTR pain point?
- How much of the estate is EKS vs EC2 vs serverless, and where is it heading?

## 8. Red flags to avoid

- Saying "I'd just use the console" for anything repeatable.
- Workspaces for prod/non-prod separation.
- Committing `.tfvars` with secrets or `terraform.tfstate` to git.
- Liveness probe that depends on a downstream.
- `latest` image tags in prod.
- Pretending to know Packer or Jenkins depth you don't have. Bridge honestly.
