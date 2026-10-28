# 10. Security and compliance in a bank

[← Back to index](README.md)

**Q79. How do you implement least privilege for a pipeline deploying to prod?**
Separate roles: a plan role with read-only plus state read, an apply role with the minimum write permissions for that component, both assumable only via OIDC from the specific repo and branch, with a session duration of an hour. Permission boundaries stop the apply role from escalating by creating admin roles. CloudTrail records every call under the role session name, which includes the pipeline run ID.

---

**Q80. How do you prove to an auditor that a prod change was authorised?**
The pipeline stores the plan output, the approver identity and timestamp, the change ticket number, and the git commit as artifacts for every prod apply. CloudTrail shows the apply role made the changes and that no human role did. Config history shows the resource timeline. Branch protection proves the code was reviewed. All of this is exportable per change.

---

**Q81. How do you handle encryption at rest and in transit?**
At rest: KMS customer-managed keys per data classification with key policies restricting decrypt to specific roles, enforced on EBS by default encryption, on S3 via bucket policy, on RDS at creation, on EKS secrets via envelope encryption. In transit: TLS 1.2+ at the ALB with ACM certificates, TLS to RDS enforced by parameter group, VPC endpoints so traffic to AWS APIs never leaves the network, mTLS inside the cluster through a mesh where required.

---

**Q82. How do you handle patching and vulnerability management?**
Immutable images rebuilt monthly and on critical CVEs through the Packer pipeline, scanned by Inspector before promotion; ASG instance refresh and EKS node group updates roll them out. Containers scanned in ECR with enhanced scanning and blocked from deploy on critical findings. Inspector and Security Hub aggregate findings; SLAs by severity. For anything that can't be rebuilt, SSM Patch Manager with maintenance windows.

---

[← Back to index](README.md)
