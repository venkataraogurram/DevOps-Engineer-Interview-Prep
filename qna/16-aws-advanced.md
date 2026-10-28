# 16. AWS – advanced and scenario questions

[← Back to index](README.md)

**Q140. Design a multi-account landing zone for a bank on AWS.**
Organizations with Control Tower: management account does billing and SCPs only. Core OU: Log Archive (org CloudTrail, Config, VPC Flow Logs, immutable S3 with Object Lock), Audit/Security tooling (Security Hub, GuardDuty, Inspector, Macie delegated admin), Network (Transit Gateway, Direct Connect, Route 53 Resolver, egress inspection with Network Firewall), Shared Services (CI runners, Packer builds, artifact repos, ECR). Workload OUs per business unit split into prod and non-prod accounts, vended by Account Factory with baseline: standard VPC, IAM Identity Center permission sets, mandatory tags, Config conformance pack. SCPs: deny regions outside ap-south-1/ap-south-2, deny leaving the org, deny root actions, deny disabling CloudTrail/GuardDuty/Config, deny public S3, require IMDSv2. Everything deployed by Terraform and StackSets, no console.

---

**Q141. How do you implement DR for a Tier-1 app across ap-south-1 and ap-south-2?**
Decide RPO/RTO first. Pilot light or warm standby for a bank's core: Aurora Global Database (RPO ~1 s, RTO ~1 min on managed failover), S3 cross-region replication, ECR replication, AMIs copied by the Packer pipeline, Terraform modules deployed in both regions with the standby at reduced capacity, Route 53 failover or ARC routing controls for deterministic failover, secrets replicated in Secrets Manager, KMS multi-region keys. Runbook as code, quarterly game days, and DR tests evidenced for regulators. Active-active only for stateless tiers; data consistency makes active-active databases a different conversation.

---

**Q142. An EC2 instance in a private subnet can't reach Secrets Manager. Debug.**
No route to the internet (private subnet, no NAT) and no interface endpoint for `secretsmanager`, or an endpoint exists but its security group doesn't allow 443 from the instance, or Private DNS is off so the SDK hits the public name with no path. Also check the endpoint policy, the instance role's `secretsmanager:GetSecretValue` and `kms:Decrypt` on the secret's key, and the resource policy on the secret. Test from the instance via SSM: `dig secretsmanager.ap-south-1.amazonaws.com` should return a 10.x address if the endpoint is working; `aws secretsmanager get-secret-value` error text separates network from IAM.

---

**Q143. How would you migrate 200 EC2-based applications to containers?**
Portfolio first: classify by stateless/stateful, OS, dependencies, compliance tier. Platform first: an EKS landing zone with the golden path (Helm chart template, pipeline template, observability, IRSA/Pod Identity, policies) so each team's migration is a mostly mechanical fit. Start with a stateless, low-risk app; measure; iterate the platform; then wave migrations by business unit. Strangler pattern for monoliths. Don't migrate what should be retired or rebuilt serverless. Keep the rollback path: old ASG stays until the new path has run through a full business cycle.

---

**Q144. Explain IAM permission boundaries and when you'd use them.**
A boundary is a managed policy attached to a user or role that caps the maximum permissions regardless of identity policies; effective permissions are the intersection. Use it to let developers create their own roles (for Lambda, EKS pods) without being able to grant more than the boundary allows, enforced by an IAM condition `iam:PermissionsBoundary` on `iam:CreateRole`. SCPs are the org-wide equivalent; boundaries are per-principal and delegatable.

---

**Q145. KMS: customer managed vs AWS managed keys, key policy vs IAM, multi-region.**
AWS managed keys are per service, free, rotate yearly, no key policy control. Customer managed keys give you the key policy (the root of trust: IAM policies only work if the key policy delegates to IAM), rotation control, grants, aliases, cross-account sharing, and deletion with a waiting period. Banks use CMKs per data classification and often per application. `kms:ViaService` conditions restrict use to specific services. Multi-region keys share key material across regions so replicated data (S3 CRR, DynamoDB global tables, Aurora Global) decrypts without re-encryption. Envelope encryption: KMS encrypts a data key; the data key encrypts the data.

---

**Q146. How do you handle a compromised access key?**
Immediately deactivate (not delete, you want the audit trail) the key; create a new one if the workload needs it; search CloudTrail for every call by that key in the window, including other regions and `sts:AssumeRole` calls, which pivot to roles; check for new users, keys, roles, policy changes, and resources created (crypto mining in unused regions is the classic); revoke sessions on affected roles with a deny-older-than policy; rotate any secrets the key could read; engage security; then fix the root cause, which is usually that a static key existed at all.

---

**Q147. S3 bucket policy denies you even as admin. Why?**
Explicit deny wins over any allow, including administrator identity policies. Common: a deny for requests not through a specific VPC endpoint (`aws:SourceVpce`), not using TLS, or from outside the org (`aws:PrincipalOrgID`), and you're calling from your laptop. Also Block Public Access, a KMS key policy that excludes you (shows as AccessDenied on GetObject), or an SCP. Read the policy, check the condition keys against your request context, and use the root user only as last resort to fix a lockout.

---

**Q148. Design the observability for 50 microservices on EKS in a bank.**
Standardise the SDK: OpenTelemetry via the ADOT collector as a DaemonSet or sidecar. Metrics to Amazon Managed Prometheus with kube-state-metrics and node exporter, dashboards in Managed Grafana, alerts in Alertmanager routed to the on-call tool; CloudWatch Container Insights for cluster-level and alarms that must integrate with AWS. Logs: Fluent Bit DaemonSet shipping structured JSON to CloudWatch Logs with per-namespace groups and retention, subscription to OpenSearch for search and to the SIEM for security. Traces to X-Ray or Tempo with trace IDs in logs. Per-service SLOs and burn-rate alerts. Audit logs from the EKS control plane and CloudTrail retained per regulation.

---

**Q149. EBS vs EFS vs S3 vs FSx: when each?**
EBS: block storage for one instance (or multi-attach io2), databases, boot volumes; AZ-bound. EFS: NFS shared across instances and AZs, elastic, pay per GB, for shared content and home directories, slow for small-file metadata-heavy workloads. S3: object storage, unlimited, eleven nines, for anything accessed by API, static assets, backups, data lakes. FSx: managed Windows file shares (SMB, AD-joined), Lustre for HPC, NetApp ONTAP for enterprise NAS features, OpenZFS. Kubernetes: EBS CSI for ReadWriteOnce, EFS CSI for ReadWriteMany.

---

**Q150. How do you enforce tagging across the organisation?**
Tag policies in Organizations define allowed keys and values; SCPs deny `ec2:RunInstances` and similar create actions without required tags (`aws:RequestTag` and `aws:TagKeys` conditions); Config rule `required-tags` flags non-compliant resources with auto-remediation or a ticket; Terraform `default_tags` at the provider plus module validation; cost allocation tags activated in Billing so Cost Explorer can group by them. Reports weekly to owners.

---

**Q151. Explain EC2 instance lifecycle hooks and a real use.**
ASG lifecycle hooks pause an instance in `Pending:Wait` or `Terminating:Wait` for a configurable time until you send `complete-lifecycle-action`. Use on launch to run bootstrap and register with a config service before it receives traffic; on terminate to drain connections, flush logs to S3, deregister from a non-ELB registry, or let a Kubernetes node drain gracefully (the EKS node termination handler does exactly this on Spot interruptions and ASG scale-in).

---

**Q152. Explain the difference between CloudWatch metrics resolution, retention, and why a 1-minute alarm sometimes doesn't fire.**
Standard EC2 metrics are 5-minute unless detailed monitoring is on; custom metrics can be 1-second high-resolution. Retention tiers down: 1-second data kept 3 hours, 1-minute for 15 days, 5-minute for 63 days, 1-hour for 15 months. An alarm with a 60-second period on a 5-minute metric sees missing data four out of five periods; depending on `TreatMissingData` it may stay INSUFFICIENT_DATA or never breach. Match the alarm period to the metric's resolution, and use `datapoints to alarm` M out of N to avoid flapping.

---

**Q153. What is AWS Config and how is it different from CloudTrail?**
CloudTrail records API calls: who did what, when, from where. Config records resource configuration state over time and evaluates it against rules: is this security group compliant right now, and what did it look like last Tuesday. Together they answer "it changed, who changed it". Config also powers conformance packs for compliance frameworks and auto-remediation through SSM Automation.

---

**Q154. How do you right-size and when is Graviton not an option?**
Compute Optimizer and CloudWatch utilisation over at least two weeks including peak; memory needs the CloudWatch agent. Pick the smallest instance where p95 utilisation sits around 50–70%. Graviton (arm64) requires arm64 builds of your software and containers, so closed-source x86-only binaries, some commercial agents, and older JVM or .NET versions block it; multi-arch images with buildx solve most cases. Test performance per workload; it's not universally faster.

[← Back to index](README.md)
