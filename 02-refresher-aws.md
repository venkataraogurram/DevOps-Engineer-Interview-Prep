# Refresher – AWS core services for the DevOps role

Focus: what an interviewer asks a 5–10 yr DevOps engineer, not certification trivia.

## Accounts and governance

- **Organizations**: management account + OUs + member accounts. SCPs are *deny-only guardrails* on the OU (they never grant). Consolidated billing. Delegated administrator for services.
- **Control Tower**: opinionated landing zone on top of Organizations: Log Archive + Audit accounts, mandatory/elective controls (preventive = SCP, detective = Config rules, proactive = CFN hooks), Account Factory. Decommission is irreversible and leaves artifacts.
- **Multi-account pattern for a bank**: Security tooling, Log archive, Shared services (CI runners, AMI builds), Network (TGW), Workload-prod, Workload-nonprod per business unit.

## IAM

- Policy types: identity-based, resource-based (S3, KMS, SQS, Lambda), permission boundaries, SCPs, session policies, VPC endpoint policies. Evaluation: explicit deny > any allow; default deny; cross-account needs allow on **both** sides unless same-account resource policy grants.
- Trust policy = who can `sts:AssumeRole`. Confused-deputy protection: `aws:SourceAccount` / `aws:SourceArn` conditions on service roles.
- Temporary credentials everywhere: instance profile (EC2), task role (ECS), IRSA / **EKS Pod Identity** (K8s), OIDC federation (GitHub Actions, GitLab), Identity Center (humans), `sts:AssumeRole` with external ID (third parties).
- Useful conditions: `aws:SecureTransport`, `aws:PrincipalOrgID`, `aws:RequestedRegion`, `aws:MultiFactorAuthPresent`, `kms:ViaService`, `ec2:ResourceTag/...`.
- Tools: Access Analyzer (unused access, external access), IAM policy simulator, CloudTrail `errorCode: AccessDenied` is your friend for debugging.

## VPC and networking

- CIDR planning: /16 per VPC, /20 subnets, non-overlapping across accounts for TGW peering. Reserve ranges with IPAM.
- Subnet tiers: public (route 0.0.0.0/0 → IGW), private (→ NAT GW, one per AZ for HA), isolated (no 0.0.0.0/0 route).
- **SG vs NACL**: SG stateful, instance/ENI level, allow-only, can reference other SGs. NACL stateless, subnet level, allow+deny, ordered rules, must open ephemeral ports (1024–65535) for return traffic.
- **Endpoints**: Gateway (S3, DynamoDB; route-table based, free). Interface (PrivateLink ENIs; hourly + per-GB; Private DNS option overrides the public name). Endpoint policies restrict what can be reached.
- **Connectivity**: VPC peering (non-transitive), Transit Gateway (hub, route tables per attachment, cross-region peering), Direct Connect (private/transit VIF, use with VPN as backup), Site-to-Site VPN.
- **DNS**: Route 53 private hosted zones, Resolver inbound/outbound endpoints for hybrid DNS, DNS Firewall.
- Troubleshooting order for "can't reach X": SG inbound on target → SG outbound on source → NACL both directions → route table → is there a listener at all → Flow Logs (REJECT lines) → Reachability Analyzer.

## EC2 and compute

- Instance families: C compute, M general, R memory, T burstable (CPU credits; `unlimited` mode = surplus charges), Graviton (`g` suffix) ~20–40% better price/perf for Linux.
- **Launch template** (versioned; use `$Latest`/`$Default`) > launch configuration (deprecated). Instance refresh on ASG rolls nodes to a new template version with `MinHealthyPercentage`.
- ASG: desired/min/max, health check type EC2 vs ELB (use ELB), lifecycle hooks, warm pools, mixed instances policy for Spot + On-Demand, target-tracking scaling (CPU 50%, ALB RequestCountPerTarget).
- IMDSv2 (`HttpTokens = required`) – always. Prevents SSRF credential theft.
- Access: SSM Session Manager, not SSH (no bastion, no port 22, audit logs to S3/CloudWatch). Patch Manager for OS patching; or immutable AMIs via Packer (see `04`).
- Storage: EBS gp3 (set IOPS/throughput independent of size), io2 for DBs, instance store ephemeral. Snapshots incremental; Data Lifecycle Manager for retention.
- Pricing: On-Demand, Savings Plans (compute SP is flexible across family/region; EC2 instance SP cheaper but locked), Reserved, Spot (2-min interruption notice; use capacity-optimized allocation).

## Load balancing

- ALB (L7): listeners → rules (host/path/header/query) → target groups (instance, IP, Lambda). Health checks per TG (path, codes, interval, thresholds). Sticky sessions, WAF association, OIDC/Cognito auth action, slow start, deregistration delay (default 300 s; lower it for faster deploys).
- NLB (L4): static IP/EIP per AZ, TLS termination or passthrough, preserves source IP, PrivateLink.
- Cross-zone LB: ALB on by default, NLB off by default (charge).
- Common mistake: opened port 443 in SG but no HTTPS listener/ACM cert → still fails.

## S3

- Strong read-after-write consistency. Storage classes: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier Instant/Flexible/Deep Archive. Lifecycle rules transition/expire.
- Security: account-level Block Public Access, bucket policy, SSE-S3 default / SSE-KMS (CMK, audit via CloudTrail, cost per request – use Bucket Keys), Object Lock (WORM – audit logs), versioning, MFA delete, access logs, replication (CRR/SRR).
- Presigned URLs for temporary access. Transfer Acceleration. Event notifications → Lambda/SQS/EventBridge.

## RDS / Aurora / DynamoDB

- RDS Multi-AZ: synchronous standby, automatic failover (DNS endpoint unchanged). Multi-AZ DB cluster (2 readable standbys). Read replicas async, cross-region OK. Automated backups (PITR up to 35 days), manual snapshots persist. Parameter groups, option groups. Encryption at rest only at creation (restore from snapshot to encrypt). IAM DB auth; Secrets Manager rotation.
- Aurora: shared storage across 3 AZs (6 copies), fast failover (~30 s), up to 15 replicas, Serverless v2 scales in ACUs, Global Database for DR, Blue/Green deployments for upgrades.
- DynamoDB: partition key design (avoid hot keys), on-demand vs provisioned + autoscaling, GSI/LSI, TTL, Streams, DAX, PITR, global tables. Terraform state locking uses a table with `LockID` (S) hash key.

## Lambda and serverless

- Limits: 15-min timeout, 10 GB memory, 250 MB unzipped / 10 GB container image, /tmp up to 10 GB. CPU scales with memory.
- Cold start → provisioned concurrency, smaller deps, SnapStart (Java/.NET/Python). Reserved concurrency caps a function and guarantees capacity.
- Event sources: synchronous (API Gateway, ALB), async (S3, SNS, EventBridge – retries 2x, DLQ/on-failure destinations), poll-based (SQS, Kinesis, DynamoDB Streams – event source mapping, batch size, bisect on error).
- Versions + aliases for traffic shifting (canary 10%). Layers for shared deps. VPC-attached Lambda needs NAT or endpoints for AWS APIs.
- API Gateway REST vs HTTP API (HTTP cheaper, fewer features); Step Functions for orchestration (Standard vs Express); EventBridge for decoupled events.

## Monitoring and audit (brief; more in `08`)

- CloudWatch metrics (1-min standard, detailed monitoring), custom metrics via EMF, alarms (static, anomaly, composite), Logs Insights, Contributor Insights, Synthetics canaries, Container Insights, Application Signals.
- CloudTrail: management events on by default (90 days in event history); create an org trail to S3 + CloudWatch Logs for retention; data events (S3 object, Lambda invoke) are opt-in and billed.
- AWS Config: resource inventory + rules + remediation (SSM Automation). Conformance packs (e.g. PCI, RBI-aligned custom packs).
- Security Hub, GuardDuty, Inspector (EC2/ECR/Lambda vulns), Macie (S3 PII).

## Well-Architected one-liners (use when asked "how would you design")

- Operational excellence: everything as code, runbooks, game days.
- Security: identity foundation, traceability, defense in depth, encrypt in transit and at rest, keep people away from data.
- Reliability: multi-AZ, auto-recover, scale horizontally, stop guessing capacity, test recovery.
- Performance: right service, right size, measure.
- Cost: pay for what you use, measure efficiency, managed services.
- Sustainability: Graviton, right-size, lifecycle data.

## Quick commands worth having in muscle memory

```bash
aws sts get-caller-identity
aws ec2 describe-instances --filters Name=tag:Env,Values=prod --query 'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]' --output table
aws ec2 describe-security-groups --group-ids sg-123 --query 'SecurityGroups[].IpPermissions'
aws logs tail /aws/lambda/my-fn --follow --since 10m
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 20
aws ssm start-session --target i-0123456789abcdef0
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::123:role/App --action-names s3:GetObject --resource-arns arn:aws:s3:::bucket/key
aws eks update-kubeconfig --name cluster --region ap-south-1
```
