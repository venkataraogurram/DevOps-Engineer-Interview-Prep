# 20. System design and platform engineering

[← Back to index](README.md)

Answer these as a whiteboard conversation: clarify requirements, state assumptions, draw the boxes, then go deep where the interviewer pushes.

**Q197. Design an internal developer platform for 50 teams deploying to AWS.**
Goal: golden paths that make the secure, observable way the easy way. Layers: (1) Landing zone from Control Tower with account vending. (2) Shared services: EKS clusters per environment or per BU, central ECR, artifact repo, Packer AMI pipeline, secrets. (3) Templates: a service scaffold (repo template with Dockerfile, Helm chart, pipeline, observability config, runbook stub) created via Backstage or a CLI. (4) Pipeline templates owned by platform, consumed by reference. (5) Policy as code at PR, plan and admission. (6) Observability baked in: logs, metrics, traces, dashboards, SLO templates. (7) A service catalog with ownership and on-call. Success measured by DORA metrics and time-to-first-deploy for a new service. Platform team runs it as a product with a roadmap and support channel.

---

**Q198. Design a highly available web application on AWS with RPO 5 minutes and RTO 30 minutes.**
Multi-AZ in one region for HA: ALB across 3 AZs, stateless app tier on EKS or ASG with autoscaling, Aurora Multi-AZ with automated backups and PITR (RPO seconds), ElastiCache for sessions, S3 for static assets and uploads with versioning. For RPO 5 min / RTO 30 min across region failure: Aurora Global Database replicating to ap-south-2, S3 CRR, ECR and AMI replication, warm standby infrastructure deployed by the same Terraform at minimal scale, Route 53 health-check failover or ARC, runbook automated in Step Functions or a pipeline, DNS TTL 60 s. Test quarterly. Costs: roughly 20–30% of prod for the warm standby.

---

**Q199. Design a CI/CD platform for a bank that passes audit.**
Source: GitHub Enterprise or GitLab with SSO, branch protection, signed commits, CODEOWNERS. Build: ephemeral runners in a dedicated account inside the VPC, no internet except via egress proxy allowlist, builder images from your own registry. Artifacts: ECR with immutability and scanning, CodeArtifact/Nexus for packages, cosign signatures. Deploy: OIDC roles per repo per environment, manual approval by a non-author tied to change tickets, immutable promotion. Evidence: pipeline logs and artifacts retained for 7 years in Object Lock S3, CloudTrail organisation trail, Config. Separation: dev, test, prod accounts; the CI account can assume deploy roles but humans can't assume prod roles. Break-glass role with MFA, alarms on use, and mandatory post-incident review.

---

**Q200. Design centralised logging for 300 accounts.**
Each account: CloudWatch Logs groups with retention, subscription filters (or Logs destinations) to a Kinesis Data Firehose in the Log Archive account via a cross-account destination; VPC Flow Logs and CloudTrail directly to central S3. Central: Firehose transforms and lands in S3 partitioned by account/service/date, then Athena/Glue for ad hoc queries and OpenSearch for the hot 30 days; a SIEM (Splunk/Security Lake) ingests security-relevant subsets. Cost controls: retention tiers, sampling of debug logs, compression, Parquet. Access: IAM and OpenSearch fine-grained roles per team; PII masking with CloudWatch data protection policies.

---

**Q201. Design the golden AMI pipeline end to end.**
Trigger: schedule (monthly), new upstream AMI from the SSM public parameter, or a critical CVE. Build: Packer in the shared-services account, private subnet, SSM communicator, Ansible hardening to CIS, agents installed, cleanup. Test: launch instance, run InSpec/OpenSCAP and smoke tests, Inspector scan; fail on critical. Promote: tag `Approved=true`, copy to DR region, share to workload accounts or the org, write the AMI ID to SSM parameters per OS family. Consume: Terraform data source or pinned variable; ASG instance refresh and EKS node group update roll fleets. Retire: deregister AMIs older than N versions, delete snapshots, SCP or Config rule denying launches from unapproved AMIs. Dashboard of AMI age per account.

---

**Q202. How would you approach cost governance for a cloud estate growing 40% a year?**
Visibility: mandatory tags, Cost Explorer and CUR into Athena/QuickSight, per-team showback dashboards, anomaly detection. Controls: Budgets with alerts to owners, Service Quotas, SCPs blocking expensive instance families in non-prod, automatic shutdown of non-prod off-hours, lifecycle policies on S3 and ECR. Optimisation cadence: monthly right-sizing from Compute Optimizer, Savings Plans coverage target around 70–80% of steady-state, Spot for stateless, Graviton migration. Culture: cost as a non-functional requirement in design reviews, and FinOps reviews with each team quarterly.

---

**Q203. Design zero-trust access for engineers to production.**
No standing access. Identity Center with MFA and short sessions; permission sets are read-only by default; elevated access through a just-in-time approval workflow (e.g. a Slack request that grants a time-boxed role via Lambda and logs to a ticket). No SSH keys; SSM Session Manager with session logging to S3 and CloudWatch, command restrictions. Kubernetes access via EKS access entries bound to Identity Center groups, `view` by default, break-glass admin role alarmed. All actions in CloudTrail and EKS audit logs; weekly access reviews. Database access via IAM auth or a bastion-less proxy with query logging.

---

**Q204. You inherit an AWS account with 500 resources created by hand. How do you bring it under IaC?**
Inventory with Resource Explorer and Config. Prioritise by risk: network, IAM, data stores first. Use Terraform `import` blocks with `-generate-config-out` to draft configs, clean them into modules, and iterate until plan is clean, component by component, each as a PR. Freeze console changes for imported components by removing write permissions. Resources that can be recreated cheaply (stateless compute) may be rebuilt from new code and cut over rather than imported. Expect months, not days; track percentage of resources under management as the metric.

---

**Q205. How do you measure and improve reliability as a platform lead?**
Define SLIs and SLOs per service with owners; error budgets drive the pace: when a budget is burnt, the team prioritises reliability over features. Track MTTD and MTTR from incident data, change failure rate and deployment frequency from the pipeline. Run blameless post-mortems with tracked actions. Invest in the top recurring causes: usually missing probes, bad rollouts, capacity, and config changes. Game days and chaos experiments (AWS Fault Injection Service) to validate failover before an incident does.

---

**Q206. How would you decide between EKS, ECS Fargate, and Lambda for a new service?**
Lambda for event-driven, bursty, short tasks and glue, minimal ops. ECS Fargate for containerised services with simple scaling needs and a team without Kubernetes skills: fewer moving parts, native AWS integration. EKS when you need the Kubernetes ecosystem (Helm charts, operators, service mesh, portability), many services sharing a platform, or teams already fluent in it. For a bank consolidating 50 teams, a shared EKS platform with strong guardrails is usually the answer; for a single new internal API, Fargate is faster to operate. State the trade-off in ops cost versus flexibility.

---

**Q207. Design an incident management process and tooling for a 24x7 platform.**
On-call rotations in PagerDuty/Opsgenie with escalation policies; alerts from CloudWatch, Prometheus Alertmanager routed by severity; a Slack channel per incident auto-created with the runbook link; roles: incident commander, operations lead, communications lead; status page updates; severity definitions with response targets; mitigation-first doctrine (rollback, failover, scale); post-mortem within 5 business days with action items in the backlog with owners; monthly review of incident trends. Tooling as code: alert rules in git, runbooks in the repo next to the service, escalation policies via Terraform.

[← Back to index](README.md)
