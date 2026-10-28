# 2. AWS

[← Back to index](README.md)

**Q7. Design the network for a three-tier banking application on AWS.**
One VPC per environment per account, /16, across three AZs in ap-south-1. Public subnets hold only the ALB and NAT gateways. Private subnets hold the app tier with routes to NAT per AZ. Isolated subnets hold RDS with no 0.0.0.0/0 route. Security groups reference each other: ALB SG allows 443 from corporate CIDRs or the WAF-fronted CloudFront; app SG allows 8080 only from the ALB SG; DB SG allows 5432 only from the app SG. VPC endpoints for S3, DynamoDB, SSM, ECR, CloudWatch Logs, STS and KMS so the app tier works without internet. Flow Logs on, NACLs coarse. Transit Gateway or Direct Connect back to the data centre. In Control Tower, this sits in a workload OU with SCPs denying region use outside India and denying public S3.

---

**Q8. Security group vs NACL?**
Security groups are stateful, attach to ENIs, allow-only, and can reference other security groups; return traffic is automatically allowed. NACLs are stateless, apply at the subnet, support explicit deny, evaluate numbered rules in order, and need ephemeral ports opened for return traffic. I use SGs for the real access model and NACLs only for coarse blocks like denying a known-bad CIDR.

---

**Q9. How does IAM policy evaluation work?**
Default deny. An explicit deny anywhere wins. Otherwise the request needs an allow from an identity policy, and if there's a permission boundary, SCP, or session policy, each of those must also allow it. Resource-based policies can grant directly within the same account; cross-account needs an allow on both sides. When debugging AccessDenied I read the CloudTrail event, check the action and resource ARN, then walk SCP → boundary → identity policy → resource policy, and use the policy simulator or Access Analyzer.

---

**Q10. How do you avoid long-lived access keys?**
Roles everywhere. EC2 gets an instance profile, ECS a task role, EKS pods use IRSA or EKS Pod Identity, CI uses OIDC federation to assume a role scoped to repo and branch, humans sign in through IAM Identity Center with short sessions and MFA. Anything that still needs a key, like a third-party SaaS, gets a dedicated user with a tight policy, rotation, and a CloudWatch alarm on usage from unexpected IPs. Config rule `iam-user-unused-credentials-check` and Access Analyzer unused-access findings catch stragglers.

---

**Q11. ALB vs NLB?**
ALB is layer 7: host and path routing, HTTP health checks, WAF, OIDC authentication, target groups per microservice, Lambda targets. NLB is layer 4: static IPs per AZ, TLS passthrough, source IP preservation, millions of connections, and it's what PrivateLink needs. API tier behind ALB plus WAF; exposing a service to other accounts privately goes through NLB and PrivateLink.

---

**Q12. RDS Multi-AZ vs read replica?**
Multi-AZ is a synchronous standby for availability; failover flips DNS in about a minute, and you can't read from the standby (except Multi-AZ DB cluster, which has two readable standbys). Read replicas are asynchronous, used to scale reads or seed cross-region DR, and can be promoted. Aurora changes the picture: shared storage, replicas double as failover targets in under 30 seconds, and Global Database for DR.

---

**Q13. How do you reduce an AWS bill by 30%?**
Measure first: Cost Explorer by service and tag, Compute Optimizer and Trusted Advisor. Then: right-size over-provisioned EC2/RDS, move to Graviton, buy a Compute Savings Plan for the steady baseline, use Spot for stateless and batch, schedule non-prod off out of hours, gp2 to gp3, delete unattached EBS, old snapshots and idle load balancers, S3 lifecycle to Intelligent-Tiering or Glacier, set CloudWatch Logs retention, and reserve RDS. Put Budgets with anomaly detection on so it doesn't creep back.

---

**Q14. Explain Lambda cold starts and how to mitigate them.**
A cold start is the time to create a new execution environment and run the initialisation code before the handler. Mitigation: smaller packages and fewer imports, initialise clients outside the handler so warm invocations reuse them, provisioned concurrency for latency-critical paths, SnapStart for Java, and avoid VPC attachment unless needed. If constant low latency at steady load is the requirement, a container on Fargate may be the better fit.

---

**Q15. How do you secure S3 for a bank?**
Account-level Block Public Access, SSE-KMS with a customer-managed key and a key policy that limits who can decrypt, bucket policy denying `aws:SecureTransport = false` and non-VPC-endpoint access, versioning with MFA delete on critical buckets, Object Lock in compliance mode for audit logs, access logging to a separate account, replication to the DR region, and Macie for PII discovery. IAM scoped by prefix, never `s3:*` on `*`.

---

**Q16. Control Tower and Organizations: how do they relate?**
Organizations gives you the account tree, consolidated billing and SCPs. Control Tower is an opinionated landing zone built on top: it creates the Log Archive and Audit accounts, enables CloudTrail and Config organisation-wide, applies preventive (SCP), detective (Config) and proactive (CloudFormation hooks) controls, and provides Account Factory for vending accounts. For a bank I'd use Control Tower for the baseline and add custom SCPs for region restriction, denying root usage, and protecting the security tooling.

---

[← Back to index](README.md)
