# DevOps / Platform Engineer – Interview Prep

Built for an IDFC FIRST Bank Platform/DevOps Engineer interview (AWS focus, Terraform + Packer, Kubernetes, CI/CD, Python/Bash, 5–10 yrs), from the recruiter's prep sheet and the JD. Reusable for any AWS-centric DevOps or platform role.

**Version:** v2.0 · [Changelog](CHANGELOG.md) · v1.0 is tagged if you want the shorter set.

## Layout

| Path | What it is | When to use |
|---|---|---|
| `01-interview-prep-guide.md` | Role analysis, positioning, round-by-round plan, question outlines, STAR story bank, questions to ask | First; again the night before |
| `02-refresher-aws.md` | EC2, VPC, IAM, S3, RDS, Lambda, ALB/ASG, cost, security, Well-Architected | Day 1 |
| `03-refresher-terraform.md` | Workflow, state, modules, variables/tfvars, remote state + locking, workspaces, dynamic blocks, lifecycle, import, debugging | Day 1–2 (heaviest weight) |
| `04-refresher-packer.md` | HCL2 templates, build lifecycle, golden AMI pipeline, Packer → Terraform hand-off, one-hour lab | Day 2 (JD gap) |
| `05-refresher-kubernetes.md` | Objects, full manifest set, probes, rollouts, networking, RBAC, Helm, Operators, troubleshooting, security | Day 2–3 |
| `06-refresher-cicd-git.md` | Pipeline stages, rollback strategies, Jenkins / GitHub Actions / GitLab CI, OIDC, branching | Day 3 |
| `07-refresher-scripting.md` | Bash + Python patterns, boto3, worked automation problems | Day 3–4 |
| `08-refresher-monitoring-docker-serverless.md` | CloudWatch, ELK, Prometheus/Grafana, Docker, microservices, serverless | Day 4 |
| `09-coding-round-practice.md` | Six "half-baked" Terraform and Kubernetes exercises with solutions and verification commands | Day 4–5, repeat |
| [`qna/`](qna/README.md) | **218 questions with full spoken-style answers**, one file per topic (21 files): core topics, Linux, networking, Ansible, advanced AWS/Terraform/Kubernetes/CI-CD, system design, advanced scripting with code | 20 a day, aloud |
| [`cheatsheets/`](cheatsheets/README.md) | **12 command cheatsheets**: Linux, networking, Git, Docker, Kubernetes/kubectl + EKS, Helm, Ansible, Terraform, Packer, AWS CLI, CI/CD syntax (Jenkins/GHA/GitLab/CodeBuild), monitoring queries (Logs Insights, PromQL, X-Ray) | Keep open during hands-on rounds |

## 7-day plan (adjust to the interview date)

- **Day 1** – `01` + `02` + first half of `03`. Skim `cheatsheets/terraform.md` and `cheatsheets/aws-cli.md`.
- **Day 2** – rest of `03` + `04`. Run the Packer lab in `04`. Q&A: `qna/03`, `qna/17`, `qna/04`.
- **Day 3** – `05` + `cheatsheets/kubernetes.md` + `cheatsheets/helm.md`. On kind/minikube: Deployment + Secret via `secretKeyRef` + probes + `rollout status`. Q&A: `qna/05`, `qna/18`.
- **Day 4** – `06` + `cheatsheets/cicd-syntax.md` + `cheatsheets/git.md`. Q&A: `qna/06`, `qna/19`, `qna/02`, `qna/16`.
- **Day 5** – `07` + `08` + `cheatsheets/linux.md`, `networking.md`, `docker.md`, `monitoring-queries.md`. Write three problems from `07` and two from `qna/21` cold. Q&A: `qna/13`, `qna/14`, `qna/08`, `qna/09`.
- **Day 6** – `09` twice. `cheatsheets/ansible.md`, `packer.md`. Q&A: `qna/15`, `qna/10`, `qna/11`, `qna/20`.
- **Day 7** – STAR stories from `01 §5` out loud. Q&A: `qna/12`, `qna/01`, `qna/07`. Re-read the prep guide.

## Ground rules for the interview

- Use `terraform validate` / `terraform plan` / `kubectl apply --dry-run=client` / `kubectl explain` openly. The recruiter's note says the coding round is about correctness and reasoning, not memory.
- Say what you're checking before you type it. Interviewers score the reasoning.
- When you don't know, say "I haven't used X in production; here's how I'd find out and here's the closest thing I have done." Then give the closest thing.
- Every answer about a tool should end with one sentence on *why* (idempotency, blast radius, cost, auditability, rollback).
- Bank context: mention audit trail, segregation of duties, encryption with CMKs, private connectivity, and change management whenever it fits.
