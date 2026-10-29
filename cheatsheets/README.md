# Command cheatsheets

Annotated references: each command is followed by a line explaining what it does and when to use it, and multi-line snippets (Dockerfile, playbook, Jenkinsfile, workflows) carry inline comments. Skim one per day; keep them open during the hands-on round.

| File | Covers |
|---|---|
| [linux.md](linux.md) | files, permissions, users, processes, systemd, journalctl, packages, text processing (grep/awk/sed/jq), disks/LVM, kernel/sysctl, cron, cloud-init, IMDSv2, log locations, performance triage order |
| [networking.md](networking.md) | ip/ss/dig, connectivity tests, curl/openssl TLS checks, tcpdump, iptables/firewalld/SELinux, SSH tunnels and jump hosts, HTTP status decoder, subnetting, AWS connectivity debug order |
| [git.md](git.md) | daily commands, undo table, merge/rebase/cherry-pick, tags and semver, bisect, worktrees/submodules/LFS, hygiene (gitignore, pre-commit, filter-repo), branching strategies |
| [docker.md](docker.md) | build/buildx, ECR, run/exec/logs, networks/volumes, Dockerfile reference, Compose, security flags, exit-code and error decoder, daemon.json |
| [kubernetes.md](kubernetes.md) | kubectl get/describe/explain, create/apply/patch, imperative generators (create deploy/expose/run/set/create ingress/quota/pdb/role) with the dry-run → yaml → apply workflow, rollouts, logs/exec/debug/port-forward, nodes/drain, RBAC and EKS access entries, secrets, storage, networking, pod status decoder, jq one-liners, EKS CLI and upgrade order |
| [helm.md](helm.md) | repos/OCI, install/upgrade/rollback flags, chart layout, template functions, hooks, values precedence, gotchas |
| [ansible.md](ansible.md) | ad-hoc and playbook CLI, inventory (static, YAML, aws_ec2 dynamic), variable precedence, full playbook with handlers/blocks/rolling, module table, Jinja2, roles, ansible.cfg, SSM connection, interview one-liners |
| [terraform.md](terraform.md) | every CLI flag you'll use, HCL block reference, version constraints, module sources, expressions and functions, S3 backend, common patterns, `terraform test`, error-to-fix table |
| [packer.md](packer.md) | CLI, template blocks, amazon-ebs arguments, template functions, provisioner tips, cleanup script, manifest parsing, SSM promotion, AMI lifecycle, builder IAM |
| [aws-cli.md](aws-cli.md) | profiles/SSO/assume-role, JMESPath patterns, EC2/ASG/SSM, ELB, S3, IAM, CloudWatch/Logs/CloudTrail, EKS/ECR/ECS/Lambda, CloudFormation, RDS/DynamoDB/Route 53/ACM/KMS, Organizations/Config/Cost/Quotas, shell wrappers |
| [cicd-syntax.md](cicd-syntax.md) | concept map across Jenkins / GitHub Actions / GitLab CI / CodeBuild, full Jenkinsfile, GitHub Actions workflow with OIDC and plan comment, GitLab CI with OIDC and manual apply, buildspec.yml, pipeline hygiene checklist |
| [monitoring-queries.md](monitoring-queries.md) | CloudWatch Logs Insights queries (JSON, parse, Lambda, Flow Logs, EKS audit, CloudTrail), metric filters and alarm flags, metric math, PromQL (node, Kubernetes, SLO burn rate), alert rules, Grafana, X-Ray filter expressions, golden-signal dashboard layout |

Not included on purpose: Python/Bash syntax (see [../07-refresher-scripting.md](../07-refresher-scripting.md)), Kubernetes manifests (see [../05-refresher-kubernetes.md](../05-refresher-kubernetes.md)).
