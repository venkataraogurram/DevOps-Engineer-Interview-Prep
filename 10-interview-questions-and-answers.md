# Interview Questions and Answers – DevOps / Platform Engineer (AWS, Terraform, Packer, Kubernetes, CI/CD)

Full spoken-style answers, 30–90 seconds each. Read them aloud; then close the file and answer from memory. Where an answer says *[your example]*, drop in a real one.

Sections: 1 DevOps & IaC · 2 AWS · 3 Terraform · 4 Packer · 5 Kubernetes · 6 CI/CD & Git · 7 Scripting · 8 Monitoring · 9 Docker, Microservices, Serverless · 10 Security & Compliance (bank) · 11 Scenario / troubleshooting · 12 Behavioral & lead

---

## 1. DevOps fundamentals and IaC

**Q1. What is DevOps, in your words?**
A culture and set of practices where the people who build software also own how it runs. In practice that means shared responsibility for the pipeline and production, small frequent changes, automation of everything repeatable, and feedback loops from monitoring back into development. I measure it with the DORA metrics: deployment frequency, lead time for change, change failure rate, and time to restore. In a bank, automation also doubles as the audit trail, because every change goes through a reviewed, logged pipeline instead of a person on a console.

**Q2. Why Infrastructure as Code? What are the downsides?**
Benefits: infrastructure becomes reviewable in pull requests, versioned in git, reproducible across environments, diff-able before change (`terraform plan`), and self-documenting. It enables ephemeral test environments and drift detection. Downsides: a learning curve, state management complexity, and the same automation that creates a VPC in two minutes can destroy it in two minutes, so you need guardrails: plan review, `prevent_destroy`, least-privilege pipeline roles, and separate state per environment.

**Q3. Mutable vs immutable infrastructure?**
Mutable: you patch and reconfigure running servers (Ansible, SSM Patch Manager). Immutable: you never change a running instance; you build a new image (Packer), roll it out, and terminate the old ones. Immutable removes drift, makes rollback a matter of pointing back at the previous AMI, and is easier to audit. Cost is a build pipeline and image lifecycle management. For a bank I'd default to immutable for compute and use configuration management only for the image build step.

**Q4. CloudFormation vs Terraform, which would you choose and why?**
CloudFormation: native, AWS holds the state, built-in rollback and drift detection, StackSets for org-wide deployment, hooks for policy. Terraform: multi-provider (AWS plus GitHub, Datadog, Kubernetes, Vault), richer module ecosystem, clearer plan output, faster coverage of new services, but you own state and locking. I've supported CloudFormation deeply, and for this JD I'd standardise application teams on Terraform because of the module ecosystem and multi-provider reach, while keeping CloudFormation/StackSets for organisation guardrails coming out of Control Tower.

**Q5. Declarative vs imperative?**
Declarative: describe the end state (Terraform, Kubernetes manifests); the tool computes the steps. Imperative: specify the steps (shell scripts, `aws` CLI). Declarative tools are idempotent by construction and can show a diff before acting; imperative scripts need you to build idempotency in yourself. I use declarative for infrastructure and imperative only for glue.

**Q6. What is idempotency and why does it matter in automation?**
Running the same operation N times gives the same result as running it once. Terraform apply on unchanged code does nothing; a Bash script that runs `aws ec2 run-instances` unconditionally creates a new instance each time. It matters because pipelines retry, people re-run jobs, and cron overlaps. I design scripts to check current state before acting and to tag what they create so a re-run finds and skips it.

---

## 2. AWS

**Q7. Design the network for a three-tier banking application on AWS.**
One VPC per environment per account, /16, across three AZs in ap-south-1. Public subnets hold only the ALB and NAT gateways. Private subnets hold the app tier with routes to NAT per AZ. Isolated subnets hold RDS with no 0.0.0.0/0 route. Security groups reference each other: ALB SG allows 443 from corporate CIDRs or the WAF-fronted CloudFront; app SG allows 8080 only from the ALB SG; DB SG allows 5432 only from the app SG. VPC endpoints for S3, DynamoDB, SSM, ECR, CloudWatch Logs, STS and KMS so the app tier works without internet. Flow Logs on, NACLs coarse. Transit Gateway or Direct Connect back to the data centre. In Control Tower, this sits in a workload OU with SCPs denying region use outside India and denying public S3.

**Q8. Security group vs NACL?**
Security groups are stateful, attach to ENIs, allow-only, and can reference other security groups; return traffic is automatically allowed. NACLs are stateless, apply at the subnet, support explicit deny, evaluate numbered rules in order, and need ephemeral ports opened for return traffic. I use SGs for the real access model and NACLs only for coarse blocks like denying a known-bad CIDR.

**Q9. How does IAM policy evaluation work?**
Default deny. An explicit deny anywhere wins. Otherwise the request needs an allow from an identity policy, and if there's a permission boundary, SCP, or session policy, each of those must also allow it. Resource-based policies can grant directly within the same account; cross-account needs an allow on both sides. When debugging AccessDenied I read the CloudTrail event, check the action and resource ARN, then walk SCP → boundary → identity policy → resource policy, and use the policy simulator or Access Analyzer.

**Q10. How do you avoid long-lived access keys?**
Roles everywhere. EC2 gets an instance profile, ECS a task role, EKS pods use IRSA or EKS Pod Identity, CI uses OIDC federation to assume a role scoped to repo and branch, humans sign in through IAM Identity Center with short sessions and MFA. Anything that still needs a key, like a third-party SaaS, gets a dedicated user with a tight policy, rotation, and a CloudWatch alarm on usage from unexpected IPs. Config rule `iam-user-unused-credentials-check` and Access Analyzer unused-access findings catch stragglers.

**Q11. ALB vs NLB?**
ALB is layer 7: host and path routing, HTTP health checks, WAF, OIDC authentication, target groups per microservice, Lambda targets. NLB is layer 4: static IPs per AZ, TLS passthrough, source IP preservation, millions of connections, and it's what PrivateLink needs. API tier behind ALB plus WAF; exposing a service to other accounts privately goes through NLB and PrivateLink.

**Q12. RDS Multi-AZ vs read replica?**
Multi-AZ is a synchronous standby for availability; failover flips DNS in about a minute, and you can't read from the standby (except Multi-AZ DB cluster, which has two readable standbys). Read replicas are asynchronous, used to scale reads or seed cross-region DR, and can be promoted. Aurora changes the picture: shared storage, replicas double as failover targets in under 30 seconds, and Global Database for DR.

**Q13. How do you reduce an AWS bill by 30%?**
Measure first: Cost Explorer by service and tag, Compute Optimizer and Trusted Advisor. Then: right-size over-provisioned EC2/RDS, move to Graviton, buy a Compute Savings Plan for the steady baseline, use Spot for stateless and batch, schedule non-prod off out of hours, gp2 to gp3, delete unattached EBS, old snapshots and idle load balancers, S3 lifecycle to Intelligent-Tiering or Glacier, set CloudWatch Logs retention, and reserve RDS. Put Budgets with anomaly detection on so it doesn't creep back.

**Q14. Explain Lambda cold starts and how to mitigate them.**
A cold start is the time to create a new execution environment and run the initialisation code before the handler. Mitigation: smaller packages and fewer imports, initialise clients outside the handler so warm invocations reuse them, provisioned concurrency for latency-critical paths, SnapStart for Java, and avoid VPC attachment unless needed. If constant low latency at steady load is the requirement, a container on Fargate may be the better fit.

**Q15. How do you secure S3 for a bank?**
Account-level Block Public Access, SSE-KMS with a customer-managed key and a key policy that limits who can decrypt, bucket policy denying `aws:SecureTransport = false` and non-VPC-endpoint access, versioning with MFA delete on critical buckets, Object Lock in compliance mode for audit logs, access logging to a separate account, replication to the DR region, and Macie for PII discovery. IAM scoped by prefix, never `s3:*` on `*`.

**Q16. Control Tower and Organizations: how do they relate?**
Organizations gives you the account tree, consolidated billing and SCPs. Control Tower is an opinionated landing zone built on top: it creates the Log Archive and Audit accounts, enables CloudTrail and Config organisation-wide, applies preventive (SCP), detective (Config) and proactive (CloudFormation hooks) controls, and provides Account Factory for vending accounts. For a bank I'd use Control Tower for the baseline and add custom SCPs for region restriction, denying root usage, and protecting the security tooling.

---

## 3. Terraform

**Q17. Difference between `terraform plan` and `terraform apply`?**
Plan refreshes state against real infrastructure, compares with the configuration, and shows what would be created, updated, replaced or destroyed; it changes nothing. Apply executes those changes. In CI I run `plan -out=tfplan`, have it reviewed, then `apply tfplan`, so what runs is exactly what was approved and not a fresh plan that might differ if something changed in between.

**Q18. What is Terraform state and why is it needed?**
State is Terraform's record mapping resources in config to real resource IDs plus their last-known attributes. It's needed because many resource attributes are only known after creation (IDs, ARNs), to detect drift, to compute dependencies for destroy ordering, and for performance so Terraform doesn't query everything every time. It can contain secrets, so it must be encrypted and access-controlled.

**Q19. How do you manage state in a team?**
Remote backend on S3 with encryption, versioning and Block Public Access; state locking via a DynamoDB table with a `LockID` hash key, or S3 native locking with `use_lockfile = true` on Terraform 1.10 and later. One state file per component per environment, so network, EKS and each app are separate and the blast radius is small. Applies run only from the CI role; humans get a read-only plan role. We never hand-edit state; we use `state mv`, `state rm`, `import`, and `moved` blocks.

**Q20. What happens if two engineers run `apply` simultaneously without locking?**
Both read the same state, both make API calls, and the second one to finish writes its state over the first. The state now disagrees with reality; the next plan shows phantom creates or destroys, and you may end up with orphaned resources. Locking makes the second run fail fast with "Error acquiring the state lock". That's why locking plus CI-only apply is non-negotiable.

**Q21. What are workspaces and when wouldn't you use them?**
Workspaces give you multiple state files for the same configuration under the same backend, selected with `terraform workspace select`. They're fine for short-lived variants like per-developer sandboxes. I wouldn't use them to separate prod from non-prod: same backend, same credentials, same provider configuration, and it's one wrong `workspace select` away from applying dev changes to prod. I prefer a directory and backend per environment, in separate AWS accounts.

**Q22. Explain modules and your best practices.**
A module is a reusable unit of Terraform with inputs, resources and outputs. Best practices: small and single-purpose (a VPC module, an EKS module, an app security group module), versioned by git tag or private registry and pinned by consumers, typed and validated input variables with descriptions, outputs for everything a consumer might need, no `provider` blocks inside reusable modules so callers pass providers, an `examples/` folder, `terraform-docs` for the README, and tests with `terraform test` or Terratest.

**Q23. `count` vs `for_each`?**
Both create multiple instances. `count` indexes by position, so removing an item in the middle shifts indexes and Terraform wants to destroy and recreate everything after it. `for_each` keys by a map key or set value, so adding or removing one item only affects that item. I use `count` only for the boolean on/off pattern (`count = var.enabled ? 1 : 0`) and `for_each` for collections.

**Q24. What is a `dynamic` block?**
It generates repeated nested blocks from a collection, for example multiple `ingress` blocks in a security group from a list of rule objects. It's useful, but I prefer standalone rule resources like `aws_vpc_security_group_ingress_rule` with `for_each` where they exist, because each rule becomes its own resource with a clean diff instead of the whole SG churning.

**Q25. What do `lifecycle` arguments do?**
`create_before_destroy` builds the replacement before destroying the old resource, for zero-downtime on things like launch templates and security groups. `prevent_destroy` makes plan fail if the resource would be destroyed, for databases and state buckets. `ignore_changes` stops Terraform fighting something managed elsewhere, like an ASG desired capacity controlled by autoscaling. `replace_triggered_by` forces replacement when another resource changes. Preconditions and postconditions assert invariants.

**Q26. How do you handle secrets in Terraform?**
Never in committed `.tfvars`. Read at apply time from Secrets Manager or SSM via data sources, or inject through `TF_VAR_` from the CI secret store. Mark variables and outputs `sensitive`, and on Terraform 1.10+ use ephemeral values so they never land in state. Remember state still holds most attributes, so the backend is encrypted with KMS and access is restricted. For things like RDS, let AWS manage the password (`manage_master_user_password`) so Terraform never sees it.

**Q27. A resource was created manually in the console. How do you bring it under Terraform?**
Write the resource block, then `terraform import ADDRESS ID`, or on 1.5+ an `import` block with `plan -generate-config-out` to draft the config. Iterate on the config until `plan` shows no changes. The risk is importing with a config that doesn't match: the next apply "fixes" the resource to match your code, so I always confirm a clean plan before merging.

**Q28. Terraform wants to replace a production resource. What do you do?**
Read the plan for the attribute marked "forces replacement". Decide if the change is needed. If it is, use `create_before_destroy`, or for stateful resources plan a migration: RDS snapshot and restore or Blue/Green deployment, ASG instance refresh via launch template version. If it's a refactor or rename, use `moved` blocks so the address changes without replacement. If the attribute is managed elsewhere, `ignore_changes`. Never apply a surprise replacement in prod.

**Q29. How do you detect and handle drift?**
Scheduled `terraform plan` in CI against each environment; alert on a non-empty plan. Config rules and CloudTrail show who changed what. Policy: no console changes in prod; break-glass changes must be followed by a PR within the day. To accept a drift, `apply -refresh-only` updates state without touching infra; to revert it, a normal apply.

**Q30. How do you structure a Terraform repository for multiple environments?**
`modules/` for reusable modules, `envs/dev`, `envs/staging`, `envs/prod` each with its own backend config and tfvars and composing the modules, pipelines per environment with path filters. Versions pinned: Terraform via `.terraform-version`, providers via `required_providers` and the committed lock file, modules by tag. Prod changes require a reviewed plan and a manual approval environment.

**Q31. What is `terraform taint` / `-replace`?**
`taint` marked a resource for recreation on the next apply; it's deprecated in favour of `terraform apply -replace=ADDRESS`, which does the same thing in a single planned step. Used when a resource is unhealthy but its config is fine, such as a corrupted instance.

**Q32. Explain `depends_on` and when it's a smell.**
Terraform infers dependencies from references. `depends_on` adds an explicit one for cases with no attribute reference, like an IAM policy attachment that must exist before an EKS node group joins. It's a smell when overused because it hides the real relationship and can cause unnecessary replans; prefer referencing an attribute.

**Q33. What are provisioners and why avoid them?**
`local-exec`, `remote-exec` and `file` run scripts during create or destroy. They're not idempotent, not visible in plan, taint the resource on failure, and require network access. I use cloud-init user data, SSM documents, or Packer-baked images instead, and `terraform_data` with triggers if I really need a hook.

**Q34. How do you test Terraform?**
Static: `fmt -check`, `validate`, `tflint`, `checkov` or `tfsec`, and OPA/Conftest on the plan JSON for policies like "no 0.0.0.0/0 on port 22" or "all buckets encrypted". Unit-ish: `terraform test` with `.tftest.hcl` files asserting on plan values. Integration: Terratest or `terraform test` with `command = apply` in an ephemeral account, then destroy. Plus the PR plan comment for human review.

---

## 4. Packer

**Q35. What is Packer and why would a bank use it?**
Packer builds machine images from code: it launches a temporary instance from a base AMI, runs provisioners (shell, Ansible), creates an AMI, and terminates the builder. Banks use it to produce golden AMIs with CIS hardening, patches, monitoring and security agents already installed and scanned, so every instance starts compliant, boots fast, and there's no configuration drift. Packer builds; Terraform deploys the AMI.

**Q36. Walk me through a Packer HCL template.**
`packer { required_plugins }` for the Amazon plugin; variables; a `data "amazon-ami"` block to find the latest base image; a `source "amazon-ebs"` block with region, source AMI, instance type, VPC and private subnet, SSM as the communicator so no port 22, encrypted boot with a CMK, `ami_regions` to copy to the DR region, `ami_users` or org ARNs to share, and tags like SourceAMI, BuildDate, GitCommit; then a `build` block with shell provisioners for updates, hardening and agent install, a cleanup script that removes keys and logs, and a `manifest` post-processor that writes the AMI IDs to JSON for the pipeline.

**Q37. How does the Packer AMI reach Terraform?**
Two patterns. Explicit promotion: the pipeline writes the approved AMI ID to an SSM parameter like `/ami/al2023/base/latest` and Terraform reads `data "aws_ssm_parameter"`. Discovery: Terraform uses `data "aws_ami"` with `most_recent = true`, `owners = ["self"]` and tag filters like `Role=base, Approved=true`. For prod I pin the AMI ID in tfvars and bump it through a PR so a new image never rolls to prod implicitly; the ASG instance refresh then replaces nodes.

**Q38. Packer vs EC2 Image Builder?**
Image Builder is the managed AWS equivalent: recipes, components including AWS-managed CIS hardening, scheduled pipelines, built-in tests and distribution. No infrastructure to run, but AWS-only and less flexible. Packer is portable and fits any CI. I'd pick Packer if the team already has CI and wants multi-cloud or Docker images too; Image Builder if they want fully managed and AWS-only.

**Q39. Common Packer failures?**
Timeout waiting for SSH or SSM: wrong subnet routing, security group, missing SSM instance profile, or wrong `ssh_username`. "AMI name already exists": include a timestamp. Encrypted AMI can't be shared: the default EBS key isn't shareable; use a CMK with a key policy for the target accounts. Slow boots after a "successful" build: cleanup missing or install steps left in user data instead of the image.

---

## 5. Kubernetes

**Q40. Explain Kubernetes architecture.**
Control plane: API server as the single entry point handling authentication, authorisation and admission; etcd storing all state; the scheduler placing pods on nodes; controller managers running reconcile loops; the cloud controller manager talking to AWS for load balancers and volumes. Each node runs kubelet, which starts containers and runs probes; kube-proxy, which programs Service routing; a container runtime like containerd; and a CNI plugin, on EKS the VPC CNI giving pods real VPC IPs. On EKS, AWS runs the control plane.

**Q41. Pod vs Deployment vs ReplicaSet vs StatefulSet vs DaemonSet?**
A Pod is one or more containers sharing a network namespace and volumes; it's ephemeral. A ReplicaSet keeps N identical pods running. A Deployment manages ReplicaSets to give rolling updates and rollbacks for stateless apps. A StatefulSet gives stable pod names and persistent volumes per replica with ordered rollout, for databases and queues. A DaemonSet runs one pod per node, for log shippers and monitoring agents.

**Q42. How do Services and Ingress work?**
A Service selects pods by label and gives them a stable ClusterIP and DNS name; kube-proxy programs iptables or IPVS so traffic to the ClusterIP is load-balanced to ready pod IPs tracked in EndpointSlices. NodePort exposes on every node; LoadBalancer provisions an NLB on EKS. Ingress is layer 7 routing by host and path, implemented by a controller; on EKS the AWS Load Balancer Controller turns an Ingress into an ALB with target-type IP pointing directly at pods. Gateway API is the newer standard.

**Q43. Liveness vs readiness vs startup probes?**
Readiness decides whether a pod receives traffic; failing removes it from Service endpoints without restarting. Liveness decides whether the container is alive; failing restarts it. Startup gates both for slow-starting apps. The classic mistake is a liveness probe that calls the database: when the DB has a blip, every pod restarts and you've turned a degradation into an outage. Liveness should be cheap and self-contained; readiness can check dependencies.

**Q44. Requests vs limits?**
Requests are what the scheduler reserves and determine QoS class. Limits are the ceiling: exceeding the CPU limit throttles the container; exceeding the memory limit kills it with OOMKilled. I set requests from observed usage, always set memory limits, and often skip CPU limits for latency-sensitive services to avoid throttling, enforcing fairness through namespace ResourceQuota and LimitRange instead.

**Q45. How do you achieve zero-downtime deployments on Kubernetes?**
Multiple replicas spread across zones, RollingUpdate with `maxUnavailable: 0` and `maxSurge: 1`, a correct readiness probe so new pods only get traffic when ready, graceful shutdown on SIGTERM with a `preStop` sleep so the endpoint is removed before the process exits, `terminationGracePeriodSeconds` matched to the ALB deregistration delay, a PodDisruptionBudget for node drains, and backward-compatible database migrations. I verify with `kubectl rollout status` and keep `kubectl rollout undo` ready.

**Q46. How do you inject secrets into a pod and how do you secure them?**
Reference a Secret with `env.valueFrom.secretKeyRef`, `envFrom.secretRef`, or mount it as a volume. Secrets are only base64-encoded, so on EKS I enable envelope encryption of etcd with a KMS key, restrict `get secrets` via RBAC, and prefer not to store them in git at all: External Secrets Operator or the Secrets Store CSI driver pulls from AWS Secrets Manager using the pod's IAM identity. For GitOps, Sealed Secrets or SOPS.

**Q47. Explain RBAC.**
Roles and ClusterRoles define rules: API groups, resources, verbs. RoleBindings and ClusterRoleBindings attach them to subjects: users, groups, or ServiceAccounts. Roles are namespaced; ClusterRoles cover cluster-wide or non-namespaced resources. Least privilege: a CI service account that needs to restart a deployment gets `patch` on deployments in that namespace, nothing on secrets. On EKS, humans map in through access entries and IAM; I verify with `kubectl auth can-i --as`.

**Q48. How do pods get AWS permissions on EKS?**
Not from the node role. IRSA: annotate the ServiceAccount with a role ARN; the pod gets a projected OIDC token and the role trusts the cluster's OIDC provider with a condition on the service account subject. EKS Pod Identity is the newer, simpler version: an agent on the node and an association between cluster, namespace, service account and role, no OIDC provider per cluster. Either way the node role stays minimal.

**Q49. What is Helm and what problems does it solve?**
Helm is the Kubernetes package manager: a chart is templated manifests plus a `values.yaml`; a release is an installed chart with specific values, tracked as Secrets in the namespace. It solves parameterisation across environments, versioned upgrades and rollbacks (`helm rollback`), and dependency management. I use `--atomic` so a failed upgrade rolls back automatically, and `helm template | kubectl apply --dry-run=server` in CI. Kustomize is the overlay-based alternative.

**Q50. What is an Operator?**
A custom controller plus a Custom Resource Definition that encodes how to run a specific application. You declare a `PostgresCluster` or `Kafka` resource and the operator handles provisioning, failover, backups and upgrades. Examples: Prometheus Operator, cert-manager, External Secrets, ArgoCD. It's the Kubernetes way to turn runbooks into software.

**Q51. How would you secure an EKS cluster?**
Private API endpoint, access entries with least privilege, audit logging to CloudWatch, secrets encryption with KMS, current version with managed add-ons patched. Nodes on Bottlerocket or a Packer-hardened AMI, IMDSv2 with hop limit 1, SSM instead of SSH, minimal node role. Workloads under Pod Security Standards `restricted`: non-root, read-only filesystem, dropped capabilities, resource limits; default-deny NetworkPolicies; image scanning in ECR and signing with cosign; admission policies via Kyverno; GuardDuty EKS protection at runtime.

**Q52. HPA vs VPA vs Cluster Autoscaler vs Karpenter?**
HPA scales pod replicas on CPU, memory or custom metrics. VPA adjusts requests and limits. Cluster Autoscaler scales existing node groups through ASGs. Karpenter provisions right-sized nodes directly from pending pod requirements, consolidates underused nodes, and handles Spot well; EKS Auto Mode bundles it. Pods scale with HPA, nodes with Karpenter.

**Q53. A pod is in CrashLoopBackOff. How do you debug it?**
`kubectl describe pod` for events and the last termination reason; `kubectl logs --previous` for the crashed container's output. Typical causes: missing env or secret, wrong command, port already bound, failing dependency at startup, OOMKilled when the exit code is 137. Fix the manifest, apply, watch `rollout status`.

**Q54. Pods are Running but the Service returns 503. Where do you look?**
`kubectl get endpoints` for the Service. Empty endpoints means either the selector doesn't match pod labels or the readiness probe is failing, which shows as `0/1 Ready`. `describe pod` shows the probe error: wrong path, wrong port, or the app bound to localhost. If endpoints are populated, check the Ingress or ALB target group health, the `targetPort`, and security groups between ALB and pods.

---

## 6. CI/CD and Git

**Q55. Describe your ideal CI/CD pipeline for an application on EKS.**
On pull request: lint, unit tests, SAST, secret scan, Terraform plan posted as a comment. On merge to main: build the container once, tag with the git SHA, scan with Trivy, sign, push to ECR with tag immutability. Deploy to dev with Helm, run integration tests, promote the same image to staging, then a manual approval tied to a change ticket, deploy to prod with rolling or canary, smoke tests, and automatic rollback if alarms fire. All AWS access via OIDC-assumed roles with no stored keys. Every stage leaves evidence for audit.

**Q56. What rollback strategies do you know?**
Redeploy the previous immutable artifact; `kubectl rollout undo` or ASG instance refresh back to the previous launch template version; blue/green where you switch the load balancer back; canary where you shift traffic to zero; feature flags that turn a change off without a deploy. The database is the hard part: expand/contract migrations so the schema is compatible with the previous release, and snapshots before any migration.

**Q57. Blue/green vs canary vs rolling?**
Rolling replaces instances gradually within one environment; cheap, but two versions coexist and rollback is a new roll. Blue/green runs two complete environments and switches traffic at once; instant rollback, double cost during the switch. Canary sends a small percentage to the new version and promotes on metrics; the safest for user impact, needs good observability and traffic-splitting support like ALB weighted target groups or Argo Rollouts.

**Q58. GitFlow vs trunk-based development?**
GitFlow has long-lived develop, release and hotfix branches; it fits scheduled releases and formal QA sign-off but creates merge pain and slow feedback. Trunk-based has everyone merging to main in small, short-lived branches with feature flags hiding unfinished work; it enables continuous delivery. For a bank I'd recommend trunk-based with release branches cut for compliance evidence, rather than environment branches, which promote by merge instead of by artifact.

**Q59. How do you secure a CI/CD pipeline?**
No static cloud credentials: OIDC federation to roles scoped per repo, branch and environment. Branch protection with required reviews and status checks. Secrets in a vault, masked in logs, scanned for in commits. Pin third-party actions by SHA. Runners inside the VPC for private resources, ephemeral. Signed commits and signed images. Separate plan and apply roles. The pipeline is the only identity with prod write access.

**Q60. Jenkins: how do you structure pipelines for 50 teams?**
Jenkinsfile per repo using declarative syntax, with a Shared Library that provides standard stages (`standardBuild()`, `deployToEks()`) so teams get the same security scans and approval gates without copying code. Ephemeral agents on Kubernetes via the Kubernetes plugin. Controller configuration as code with JCasC, credentials in the Jenkins store backed by Secrets Manager, Role Strategy for RBAC, and backups of JENKINS_HOME.

**Q61. Merge vs rebase?**
Merge preserves history and creates a merge commit; rebase rewrites your commits on top of the target for linear history. Rebase only on your own unpushed branches; never rebase a shared branch because it changes SHAs others have. Teams commonly squash-merge PRs for a clean main.

**Q62. How do you handle hotfixes in production?**
Branch from the tag that's in prod, fix, test through the same pipeline with an expedited approval, deploy, tag, then merge the fix back to main so it isn't lost. Record the change ticket. If the fix is infra, the same flow with Terraform plan review.

---

## 7. Scripting

**Q63. When do you choose Bash and when Python?**
Bash for short glue around CLI tools, under about fifty lines, where I'm piping `aws`, `jq`, `grep` and `awk`. Python when I need data structures, real error handling, JSON manipulation, concurrency, unit tests, or anything that will live longer than a week. Both get `shellcheck` or `ruff` in pre-commit.

**Q64. What does `set -euo pipefail` do?**
`-e` exits on any command failure, `-u` errors on undefined variables, and `pipefail` makes a pipeline fail if any stage fails instead of just the last. Together they stop scripts from silently continuing after an error. I pair them with a `trap` on ERR to log the line and on EXIT to clean up temp files.

**Q65. How would you write a script to stop untagged EC2 instances?**
Describe running instances with a JMESPath filter for those missing the Owner tag, print them, and only stop if a `--apply` flag is passed, so dry-run is the default. Use pagination, handle no results, log what was done, and run it as a Lambda on an EventBridge schedule with a role limited to `ec2:Describe*` and `ec2:StopInstances` with a condition on the tag. Send a summary to SNS.

**Q66. Parse an access log and find the top endpoints by 5xx errors.**
In Bash: `awk '$9 ~ /^5/ {c[$7]++} END {for (p in c) print c[p], p}' access.log | sort -rn | head`. In Python: stream the file line by line, a compiled regex with named groups, a `Counter` keyed by path for statuses starting with 5, and print `most_common(10)` as JSON. Streaming matters so it works on multi-GB logs.

**Q67. How do you make a script idempotent?**
Check before act: look up the resource by name or tag and skip if it exists; use create-if-missing API semantics or Terraform instead of imperative creation; tag anything you create so re-runs recognise it; make deletes tolerate "not found"; keep a lock for cron jobs so overlapping runs don't double-execute.

**Q68. How do you handle AWS API throttling in scripts?**
Use the SDK's built-in retries with adaptive mode in botocore config, paginate instead of pulling everything, add jittered backoff for custom loops, batch where the API allows, and spread scheduled jobs so they don't all hit the API at the top of the hour.

---

## 8. Monitoring and observability

**Q69. How do you monitor a production service?**
Start with the golden signals: latency, traffic, errors, saturation. CloudWatch metrics and alarms for infrastructure, custom metrics from the app via EMF or Prometheus, structured JSON logs to CloudWatch Logs with retention set, distributed tracing with X-Ray or OpenTelemetry, synthetic canaries on key journeys. Define SLOs and alert on error-budget burn rather than CPU. Every alert routes to an on-call rotation with a runbook link. Dashboards per service for humans; alerts for pages.

**Q70. CloudWatch vs ELK?**
CloudWatch is native and zero-ops with Logs Insights, metric filters and alarms, priced per GB ingested and scanned. ELK or OpenSearch gives full-text search, rich dashboards and cheaper long retention but needs operating or the managed OpenSearch Service. Many banks ship CloudWatch logs through Kinesis Firehose to OpenSearch or Splunk for central SIEM. I'd use CloudWatch for operational alerting and OpenSearch for cross-source investigation and compliance retention.

**Q71. What is the difference between monitoring and observability?**
Monitoring tells you a known condition happened: CPU above 80%, 5xx above 1%. Observability lets you ask new questions about the system from its outputs: metrics, logs and traces with enough context, like correlation IDs, to explain an incident you didn't predict. In practice: structured logs, trace propagation, high-cardinality labels, and tools that join them.

**Q72. How do you reduce alert fatigue?**
Alert on symptoms, not causes; use composite alarms so one incident is one page; set `treat missing data` correctly; use anomaly detection for seasonal metrics; tune thresholds from historical data; route informational alerts to a channel not a pager; review every page weekly and delete or fix noisy ones; every alert must be actionable with a runbook.

**Q73. Describe your incident response process.**
Detect through alarms, triage severity, open an incident channel, assign an incident commander and a communicator who updates stakeholders every 30 minutes, mitigate first (rollback, scale, failover) and root-cause later, resolve, then a blameless post-mortem within a few days with action items tracked to completion. Measure MTTD and MTTR.

---

## 9. Docker, microservices, serverless

**Q74. Dockerfile best practices?**
Pin the base image, use slim or distroless, multi-stage builds so build tools don't ship, order layers from least to most frequently changing for cache hits, `.dockerignore`, run as a non-root user, one process per container, exec-form ENTRYPOINT so PID 1 gets SIGTERM, no secrets in layers, add a HEALTHCHECK and OCI labels with the git revision, and scan the image in CI.

**Q75. ENTRYPOINT vs CMD?**
ENTRYPOINT is the fixed executable; CMD supplies default arguments that `docker run` can override. Use exec form, a JSON array, for both so signals reach the process directly instead of through a shell.

**Q76. What makes a good microservice boundary?**
A service owns its data and a business capability; it can be deployed independently; communication is through versioned APIs or events, not shared databases. Cross-cutting concerns such as auth, logging with correlation IDs, tracing, retries with backoff and circuit breakers, and timeouts are handled consistently, often by a gateway or a mesh. The honest trade-off: operational complexity and distributed debugging, so don't split a small team's monolith prematurely.

**Q77. When would you choose serverless over containers?**
Event-driven, spiky or low-volume workloads, glue between AWS services, scheduled jobs, and APIs where per-request pricing wins. Containers when you need long-running processes, steady high load where per-request pricing is more expensive, custom runtimes, or strict latency with no cold starts. Many systems mix both.

**Q78. SQS message processed twice. Why, and how do you handle it?**
Standard SQS is at-least-once; a message reappears if the consumer doesn't delete it before the visibility timeout expires, for example a Lambda that runs longer than the timeout. Set visibility timeout to at least six times the function timeout, make the handler idempotent with an idempotency key stored in DynamoDB, use a DLQ with a maxReceiveCount, and alarm on DLQ depth. FIFO queues add exactly-once processing within a deduplication window.

---

## 10. Security and compliance in a bank

**Q79. How do you implement least privilege for a pipeline deploying to prod?**
Separate roles: a plan role with read-only plus state read, an apply role with the minimum write permissions for that component, both assumable only via OIDC from the specific repo and branch, with a session duration of an hour. Permission boundaries stop the apply role from escalating by creating admin roles. CloudTrail records every call under the role session name, which includes the pipeline run ID.

**Q80. How do you prove to an auditor that a prod change was authorised?**
The pipeline stores the plan output, the approver identity and timestamp, the change ticket number, and the git commit as artifacts for every prod apply. CloudTrail shows the apply role made the changes and that no human role did. Config history shows the resource timeline. Branch protection proves the code was reviewed. All of this is exportable per change.

**Q81. How do you handle encryption at rest and in transit?**
At rest: KMS customer-managed keys per data classification with key policies restricting decrypt to specific roles, enforced on EBS by default encryption, on S3 via bucket policy, on RDS at creation, on EKS secrets via envelope encryption. In transit: TLS 1.2+ at the ALB with ACM certificates, TLS to RDS enforced by parameter group, VPC endpoints so traffic to AWS APIs never leaves the network, mTLS inside the cluster through a mesh where required.

**Q82. How do you handle patching and vulnerability management?**
Immutable images rebuilt monthly and on critical CVEs through the Packer pipeline, scanned by Inspector before promotion; ASG instance refresh and EKS node group updates roll them out. Containers scanned in ECR with enhanced scanning and blocked from deploy on critical findings. Inspector and Security Hub aggregate findings; SLAs by severity. For anything that can't be rebuilt, SSM Patch Manager with maintenance windows.

---

## 11. Scenario and troubleshooting

**Q83. Terraform apply fails midway in production. What now?**
Terraform has already written partial state for created resources, so the state is consistent for what succeeded. Read the error; it's usually an API error on one resource. Fix the cause and re-run plan; it should only show the remaining changes. If the failure left something half-created outside state, import it. Never `rm -rf .terraform` or delete state. If the lock is stuck from a crashed run, confirm nothing is running and `force-unlock`.

**Q84. A deployment doubled latency. How do you investigate?**
Confirm with the ALB TargetResponseTime and app p99 metrics and correlate the timing with the deploy. Roll back first if users are affected, then investigate: compare traces between versions for the slow span, check for new dependency calls, missing indexes, connection pool exhaustion, CPU throttling from new limits, or a config change like a disabled cache. Check HPA scaled as expected and that pods aren't being OOMKilled and restarting.

**Q85. EC2 instances can't reach S3 from a private subnet. Why?**
Private subnet with no NAT route and no S3 gateway endpoint, or an endpoint whose route table isn't associated with that subnet, or an endpoint policy or bucket policy that denies. Also the SG egress and the instance role. Check `aws s3 ls` from the instance via SSM, the route table, the endpoint's associated route tables, then CloudTrail for AccessDenied.

**Q86. Node group upgrade on EKS causes 502s. What went wrong?**
Pods were evicted faster than new ones became ready. Causes: no PodDisruptionBudget, readiness probe missing so the Service sent traffic to pods still starting, ALB target deregistration delay longer than pod termination grace, or a single replica. Fix those, and use `maxUnavailable` on the node group update config, or Karpenter with drift and disruption budgets.

**Q87. Someone deleted the Terraform state file. How do you recover?**
The state bucket has versioning, so restore the previous version. If versioning wasn't on, rebuild state by writing the config and importing every resource, which is painful and the reason versioning and restricted bucket access are non-negotiable from day one.

**Q88. Your pipeline's AWS credentials leaked in logs. What do you do?**
If they were static keys: deactivate immediately, rotate, check CloudTrail for use, then replace with OIDC roles so there's nothing to leak. If they were OIDC session tokens: they expire within the hour, but still review CloudTrail, add log masking for `AWS_SECRET_ACCESS_KEY`-like patterns, and add a secret scanner to the pipeline. Write the incident up.

**Q89. Cost jumped 40% this month. How do you find out why?**
Cost Explorer grouped by service, then by usage type and tag; Cost Anomaly Detection findings; compare to last month. Common culprits: NAT gateway data processing from a new workload, CloudWatch Logs ingestion from debug logging left on, an ASG that scaled and never scaled in, cross-AZ data transfer, snapshots piling up, or a forgotten test environment. Fix the cause, then add a budget alert and tag enforcement.

**Q90. A customer-facing app is down and the on-call engineer is stuck. How do you lead the incident?**
Take or assign the incident commander role, get the facts: what changed, when it started, what the dashboards show. Decide on the fastest safe mitigation, usually rollback of the most recent change, and do it. Assign someone to stakeholder updates on a fixed cadence. Avoid parallel uncoordinated changes. Once stable, root cause and a post-mortem with owners and dates.

---

## 12. Behavioral and lead-level

**Q91. Tell me about yourself.**
Cloud and DevOps engineer with four-plus years across AWS and infrastructure automation. Currently at AWS as a Cloud Support Engineer in the Developer Tools space, where I own enterprise cases end to end on Elastic Beanstalk, EKS, CloudFormation, Control Tower and the CI/CD services: reproducing failures, root-causing them, and delivering validated fixes under SLA. That means I read and debug other people's Terraform, CloudFormation and Kubernetes every day. Outside work I've built production-style platforms end to end: an EKS microservice with Terraform, ALB ingress, Pod Identity, HPA and a GitHub Actions pipeline using OIDC. Before AWS I was at Infosys doing infrastructure and automation. I'm looking to move from fixing platforms to owning and building them, which is why this role interests me.

**Q92. Why IDFC FIRST Bank? Why leave AWS?**
I want to own a platform rather than support many. A bank modernising on AWS is where the problems I like, secure automation, compliance as code, reliable delivery, matter most. At AWS I've seen what goes wrong in hundreds of customer environments; I want to apply that to building one well. *[Add what you've learned about their tech direction.]*

**Q93. Tell me about a difficult production issue you solved.**
*[Use STAR story 1 from the prep guide. Situation: a failing deployment with a misleading error. Task: restore service under SLA. Action: read the actual logs instead of the obvious theory, reproduced in a sandbox, proved the root cause, delivered the fix. Result: resolved, customer confirmed, and the finding documented for the team. Learning: the first plausible theory is usually wrong; evidence first.]*

**Q94. Tell me about a time you disagreed with a stakeholder.**
*[Pick a case where a customer or colleague wanted a quick fix that would have created risk, for example opening a port to 0.0.0.0/0. Explain how you showed the risk concretely, proposed the secure alternative, and got agreement. Result and learning.]*

**Q95. How do you handle a change request from a developer that bypasses the standard pipeline?**
Understand the urgency, offer the fastest compliant path, usually an expedited pipeline run with approval, rather than a manual change. If it's a genuine emergency, break-glass with a documented role, logged, and a follow-up PR the same day. Then look at why the standard path felt too slow and fix that.

**Q96. How would you introduce Terraform standards to teams that each do their own thing?**
Start by listening: inventory what exists and what hurts. Publish a small set of opinionated modules and a reference repo layout, with a pipeline template that gives teams security scanning and plan review for free. Make the golden path the easiest path, not a mandate. Pilot with one willing team, measure, then expand. Policy as code catches the things that must never happen; everything else is guidance and review.

**Q97. How do you keep your skills current?**
Hands-on projects on my own AWS account, reading release notes for Terraform, EKS and the AWS services I work with, and the daily exposure at AWS to new failure modes. Recently: EKS Pod Identity, Terraform 1.10 S3 native locking, and agentic AI tooling for DevOps.

**Q98. What is your biggest weakness?**
*[Pick something real and bounded. Example: I've used Packer in my own lab rather than at enterprise scale, so my first month would include learning the bank's image lifecycle requirements in detail. Or: I can go deep on a root cause when a faster mitigation should come first; I've trained myself to roll back first and investigate after.]*

**Q99. Where do you see yourself in three years?**
Owning platform architecture for a line of business: the landing zone, the IaC standards, the delivery pipelines, and the reliability targets, and mentoring engineers into that work.

**Q100. Do you have questions for us?**
Yes. How is Terraform state and prod apply access managed today? Is Packer already in use for golden AMIs or is that greenfield? Which CI tool is the standard and how does CAB approval integrate? What does the on-call model look like, and what is the biggest reliability pain point right now? How much of the estate is EKS versus EC2 versus serverless, and where is it heading?
