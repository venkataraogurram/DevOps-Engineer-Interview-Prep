# IDFC FIRST Bank – Platform / DevOps Engineer – Interview Prep

Source material: recruiter's "DevOps Interview Prep Guide" PDF + the JD PDF (DevOps Engineer, IT, AWS focus, Terraform + Packer, 5–10 yrs).

## Files

| File | What it is | When to read |
|---|---|---|
| `01-interview-prep-guide.md` | Role analysis, how to position yourself, round-by-round plan, likely questions with answer outlines, STAR story bank, questions to ask them | First. Re-read the night before. |
| `02-refresher-aws.md` | EC2, VPC, IAM, S3, RDS, Lambda, ALB/ASG, cost + security, Well-Architected one-liners | Day 1 |
| `03-refresher-terraform.md` | Providers, state, modules, variables/tfvars, remote state + locking, workspaces, dynamic blocks, lifecycle, import, debugging | Day 1–2 (heaviest weight) |
| `04-refresher-packer.md` | HCL2 templates, builders/provisioners/post-processors, AMI pipeline, Packer + Terraform hand-off | Day 2 (JD gap – do not skip) |
| `05-refresher-kubernetes.md` | Objects, networking, probes, resources, rollouts, RBAC, Helm, Operators, security | Day 2–3 |
| `06-refresher-cicd-git.md` | Pipeline stages, rollback strategies, Jenkins / GitHub Actions / GitLab CI, branching, OIDC to AWS | Day 3 |
| `07-refresher-scripting.md` | Bash + Python patterns for automation, log parsing, boto3, typical whiteboard problems with solutions | Day 3–4 |
| `08-refresher-monitoring-docker-serverless.md` | CloudWatch, ELK, Prometheus/Grafana, Docker, microservices, serverless | Day 4 |
| `09-coding-round-practice.md` | 6 "half-baked" Terraform and Kubernetes exercises in the exact style the recruiter described, with solutions and the commands to verify | Day 4–5, then repeat |
| `10-interview-questions-and-answers.md` | 100 questions with full spoken-style answers across all topics, scenarios, and behavioral/lead questions | Every day, 20 at a time; read aloud |

## 5-day plan (adjust to your interview date)

- **Day 1** – `01` + `02` + first half of `03`. Rebuild your ShopStack Terraform mentally: provider → VPC → EKS → outputs.
- **Day 2** – rest of `03` + `04`. Install Packer, build one AL2023 AMI with a shell provisioner, reference it from Terraform via `data "aws_ami"`. One hour is enough to be able to talk about it honestly.
- **Day 3** – `05` + `06`. On a kind/minikube cluster: Deployment + Secret + `secretKeyRef` + probes + `kubectl rollout status`.
- **Day 4** – `07` + `08`. Write the three Python/Bash problems in `07` from scratch without looking.
- **Day 5** – `09` twice. Then re-read the STAR stories in `01` out loud.
- **Every day** – 20 questions from `10`, answered aloud before reading the answer.

## Ground rules for the interview

- Use `terraform validate` / `terraform plan` / `kubectl apply --dry-run=client` / `kubectl explain` openly. The recruiter's note says the coding round is about correctness and reasoning, not memory.
- Say what you're checking before you type it. Interviewers score the reasoning.
- When you don't know, say "I haven't used X in production; here's how I'd find out and here's the closest thing I have done." Then give the closest thing. Packer is your most likely place to need this line.
- Every answer about a tool should end with one sentence on *why* (idempotency, blast radius, cost, auditability, rollback).
