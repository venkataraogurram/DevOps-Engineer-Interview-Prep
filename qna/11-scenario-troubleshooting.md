# 11. Scenario and troubleshooting

[← Back to index](README.md)

**Q83. Terraform apply fails midway in production. What now?**
Terraform has already written partial state for created resources, so the state is consistent for what succeeded. Read the error; it's usually an API error on one resource. Fix the cause and re-run plan; it should only show the remaining changes. If the failure left something half-created outside state, import it. Never `rm -rf .terraform` or delete state. If the lock is stuck from a crashed run, confirm nothing is running and `force-unlock`.

---

**Q84. A deployment doubled latency. How do you investigate?**
Confirm with the ALB TargetResponseTime and app p99 metrics and correlate the timing with the deploy. Roll back first if users are affected, then investigate: compare traces between versions for the slow span, check for new dependency calls, missing indexes, connection pool exhaustion, CPU throttling from new limits, or a config change like a disabled cache. Check HPA scaled as expected and that pods aren't being OOMKilled and restarting.

---

**Q85. EC2 instances can't reach S3 from a private subnet. Why?**
Private subnet with no NAT route and no S3 gateway endpoint, or an endpoint whose route table isn't associated with that subnet, or an endpoint policy or bucket policy that denies. Also the SG egress and the instance role. Check `aws s3 ls` from the instance via SSM, the route table, the endpoint's associated route tables, then CloudTrail for AccessDenied.

---

**Q86. Node group upgrade on EKS causes 502s. What went wrong?**
Pods were evicted faster than new ones became ready. Causes: no PodDisruptionBudget, readiness probe missing so the Service sent traffic to pods still starting, ALB target deregistration delay longer than pod termination grace, or a single replica. Fix those, and use `maxUnavailable` on the node group update config, or Karpenter with drift and disruption budgets.

---

**Q87. Someone deleted the Terraform state file. How do you recover?**
The state bucket has versioning, so restore the previous version. If versioning wasn't on, rebuild state by writing the config and importing every resource, which is painful and the reason versioning and restricted bucket access are non-negotiable from day one.

---

**Q88. Your pipeline's AWS credentials leaked in logs. What do you do?**
If they were static keys: deactivate immediately, rotate, check CloudTrail for use, then replace with OIDC roles so there's nothing to leak. If they were OIDC session tokens: they expire within the hour, but still review CloudTrail, add log masking for `AWS_SECRET_ACCESS_KEY`-like patterns, and add a secret scanner to the pipeline. Write the incident up.

---

**Q89. Cost jumped 40% this month. How do you find out why?**
Cost Explorer grouped by service, then by usage type and tag; Cost Anomaly Detection findings; compare to last month. Common culprits: NAT gateway data processing from a new workload, CloudWatch Logs ingestion from debug logging left on, an ASG that scaled and never scaled in, cross-AZ data transfer, snapshots piling up, or a forgotten test environment. Fix the cause, then add a budget alert and tag enforcement.

---

**Q90. A customer-facing app is down and the on-call engineer is stuck. How do you lead the incident?**
Take or assign the incident commander role, get the facts: what changed, when it started, what the dashboards show. Decide on the fastest safe mitigation, usually rollback of the most recent change, and do it. Assign someone to stakeholder updates on a fixed cadence. Avoid parallel uncoordinated changes. Once stable, root cause and a post-mortem with owners and dates.

---

[← Back to index](README.md)
