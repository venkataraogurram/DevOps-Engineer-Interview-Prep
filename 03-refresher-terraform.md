# Refresher – Terraform (highest-weight topic)

## 1. Mental model

Terraform reads `.tf` files → builds a dependency graph → compares desired config against **state** (its record of what it created, plus refreshed real attributes) → produces a plan (create / update in-place / replace / destroy / read) → applies through **providers** that call APIs.

Idempotent: re-running `apply` on unchanged config does nothing. Declarative: you describe the end state, not steps.

## 2. Core workflow and commands

```bash
terraform init            # download providers/modules, configure backend; -upgrade, -reconfigure, -migrate-state
terraform fmt -recursive  # canonical formatting
terraform validate        # syntax + internal consistency (no API calls)
terraform plan -out=tfplan -var-file=envs/prod.tfvars
terraform show tfplan     # human-readable; -json for policy checks (OPA/Sentinel/Checkov)
terraform apply tfplan    # applies exactly the reviewed plan
terraform destroy         # or apply -destroy
terraform output -json
terraform state list | pull | show ADDR | mv OLD NEW | rm ADDR
terraform import ADDR ID
terraform taint ADDR      # deprecated → use: terraform apply -replace=ADDR
terraform refresh         # deprecated → terraform apply -refresh-only
terraform console         # evaluate expressions
terraform graph | dot -Tpng > graph.png
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
TF_LOG=DEBUG TF_LOG_PATH=tf.log terraform plan
```

**`plan` vs `apply`:** plan is read-only (refresh + diff). Apply makes changes. Saved plan files guarantee what was approved is what runs. Plan files can contain secrets; treat as sensitive and short-lived.

## 3. Block types

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }   # ~> 5.0 == >= 5.0, < 6.0
  }
  backend "s3" { ... }
}

provider "aws" {
  region = var.region
  default_tags { tags = { Project = "idfc-platform", ManagedBy = "terraform" } }
}

provider "aws" {
  alias  = "dr"
  region = "ap-south-2"
}

resource "aws_vpc" "main" { ... }            # ADDR: aws_vpc.main
data "aws_ami" "al2023" { ... }              # read-only lookup
module "vpc" { source = "..." }              # ADDR: module.vpc.aws_vpc.this
variable "x" {}                              # input
output "y" {}                                # output
locals { name = "${var.project}-${var.env}" }
moved { from = aws_instance.a  to = aws_instance.b }     # ≥1.1 refactor without destroy
import { to = aws_s3_bucket.logs  id = "my-logs-bucket" } # ≥1.5
removed { from = aws_instance.old  lifecycle { destroy = false } } # ≥1.7 forget without destroy
check "alb_healthy" { data "http" "x" {...}  assert { condition = ... error_message = "..." } } # ≥1.5
```

## 4. Variables, tfvars, precedence

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 type for app tier"
  default     = "t3.micro"
  validation {
    condition     = can(regex("^t3\\.|^m6", var.instance_type))
    error_message = "Use t3 or m6 families."
  }
}

variable "db_password" {
  type      = string
  sensitive = true          # redacted in plan/apply output; still in state
  ephemeral = true          # ≥1.10: never written to state/plan
}

variable "subnets" {
  type = map(object({ cidr = string, az = string, public = optional(bool, false) }))
}
```

Precedence (later wins): defaults → `TF_VAR_name` env → `terraform.tfvars` / `*.auto.tfvars` → `-var-file` → `-var` (in command-line order). Files: `prod.tfvars` must be passed explicitly with `-var-file`.

Types: `string number bool list(T) set(T) map(T) object({}) tuple([]) any`. `optional()` in object attrs. Type conversion is automatic where safe.

## 5. Expressions you will need in the coding round

```hcl
# References
aws_subnet.private[0].id                 # count index
aws_subnet.private["a"].id               # for_each key
module.vpc.private_subnet_ids            # module output
data.aws_ami.al2023.id
var.env  local.name  path.module  terraform.workspace

# count vs for_each (prefer for_each – stable keys, no reshuffle on removal)
resource "aws_subnet" "private" {
  for_each          = { for k, v in var.subnets : k => v if !v.public }
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  tags              = { Name = "${local.name}-private-${each.key}" }
}

# Collections
[for s in aws_subnet.private : s.id]                 # list of ids
{ for k, s in aws_subnet.private : k => s.id }       # map
values(aws_subnet.private)[*].id                     # splat
length(var.azs)  element(var.azs, count.index)  cidrsubnet(var.vpc_cidr, 4, count.index)
merge(local.tags, { Name = "x" })  lookup(var.m, "k", "default")  try(var.x.y, null)  coalesce(a, b)
join(",", list)  split(",", s)  format("%s-%02d", name, i)  templatefile("${path.module}/user_data.sh.tpl", { x = 1 })
jsonencode({ Version = "2012-10-17", Statement = [...] })   # IAM policies
file("${path.module}/key.pub")  fileexists()  base64encode()
timestamp()  uuid()  # cause perpetual diff – avoid in resources

# Conditionals
count = var.create_nat ? 1 : 0
instance_type = var.env == "prod" ? "m6i.large" : "t3.small"

# Dynamic blocks (repeat a nested block)
resource "aws_security_group" "app" {
  name   = "${local.name}-app"
  vpc_id = aws_vpc.main.id
  dynamic "ingress" {
    for_each = var.ingress_rules          # list(object({from=number,to=number,proto=string,cidrs=list(string)}))
    content {
      from_port   = ingress.value.from
      to_port     = ingress.value.to
      protocol    = ingress.value.proto
      cidr_blocks = ingress.value.cidrs
    }
  }
  egress { from_port = 0  to_port = 0  protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }
}
# Prefer aws_vpc_security_group_ingress_rule / _egress_rule resources (one rule each) over inline blocks – avoids whole-SG churn.

# Dependencies
depends_on = [aws_iam_role_policy_attachment.node]   # explicit; use only when implicit refs don't exist

# Lifecycle
lifecycle {
  create_before_destroy = true
  prevent_destroy       = true
  ignore_changes        = [tags["LastScanned"], desired_capacity]
  replace_triggered_by  = [aws_launch_template.app.latest_version]
  precondition  { condition = ... error_message = "..." }
  postcondition { condition = self.public_ip == null  error_message = "No public IP allowed" }
}
```

## 6. Remote state and locking (will be asked)

```hcl
terraform {
  backend "s3" {
    bucket         = "idfc-tfstate-123456789012-ap-south-1"
    key            = "platform/network/prod/terraform.tfstate"   # one key per component+env
    region         = "ap-south-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:ap-south-1:123456789012:key/..."
    dynamodb_table = "terraform-locks"      # classic locking; table hash key = LockID (String)
    # use_lockfile = true                   # Terraform ≥1.10: S3-native lock (.tflock object); DynamoDB then optional/deprecated
  }
}
```

Backend block cannot use variables → use `-backend-config=envs/prod.backend.hcl` or `-backend-config="key=..."` at `init`.

Bootstrap: the bucket + table themselves are created once (separate tiny config with local state, or CFN). Bucket: versioning on, Block Public Access, SSE-KMS, bucket policy denying non-TLS, lifecycle to expire noncurrent versions after N days.

Why remote state: shared source of truth, locking, encryption, versioning for recovery, CI can run it.

Why locking: prevents concurrent writes corrupting state. Stuck lock → `terraform force-unlock LOCK_ID` only after confirming no apply is running.

Read another stack's outputs:

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config  = { bucket = "idfc-tfstate-...", key = "platform/network/prod/terraform.tfstate", region = "ap-south-1" }
}
subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
```

Alternative that avoids state coupling: publish outputs to SSM Parameter Store and read with `data "aws_ssm_parameter"`.

State hygiene: never commit `.tfstate`; `.gitignore` → `.terraform/`, `*.tfstate*`, `*.tfplan`, `crash.log`, `*.auto.tfvars` if they hold secrets. State contains secrets in plaintext – restrict bucket IAM.

## 7. Modules

```
infra/
├── modules/
│   ├── vpc/        main.tf variables.tf outputs.tf versions.tf README.md
│   ├── eks/
│   └── app-sg/
├── envs/
│   ├── dev/        main.tf backend.hcl dev.tfvars
│   └── prod/
└── .github/workflows/terraform.yml
```

```hcl
module "vpc" {
  source  = "git::https://git.idfc.internal/platform/tf-modules.git//vpc?ref=v1.4.0"   # or registry "terraform-aws-modules/vpc/aws" version = "~> 5.0"
  name    = local.name
  cidr    = var.vpc_cidr
  azs     = var.azs
  providers = { aws = aws }        # pass aliased providers explicitly when needed
}
output "private_subnet_ids" { value = module.vpc.private_subnets }
```

Rules: pin `ref`/`version`; modules declare `required_providers` but no `provider` blocks; expose outputs; validate inputs; keep modules small and composable; `terraform-docs` for README; test with `terraform test` (≥1.6, `.tftest.hcl`) or Terratest.

## 8. Workspaces

`terraform workspace new|select|list|show`. Same config, separate state files under the same backend (`env:/<name>/<key>` in S3). `terraform.workspace` available in config.

Use for: short-lived variants, per-developer sandboxes. Avoid for prod vs non-prod in regulated setups – shared backend and credentials, easy to apply to the wrong one, can't vary backend/provider per workspace. Prefer directory-per-env with its own backend and account.

## 9. Provisioners, null/terraform_data, user data

Provisioners (`local-exec`, `remote-exec`, `file`) are a last resort: not idempotent, not in plan, fail-tainting. Prefer `user_data`/cloud-init, SSM documents, Packer-baked AMIs, or the Kubernetes/Helm provider. `terraform_data` (≥1.4) replaces `null_resource` for triggers.

## 10. Importing and drift

- `terraform import aws_security_group.app sg-0123` then write matching config until `plan` is clean. Or `import {}` block + `terraform plan -generate-config-out=generated.tf`.
- Drift: `terraform plan` shows it (refresh). `apply -refresh-only` accepts reality into state without changing infra. Policy: console changes forbidden in prod; detect with scheduled plan in CI, alert on non-empty.

## 11. Debugging checklist (coding-round gold)

| Symptom | Likely cause | Fix |
|---|---|---|
| `Error: Reference to undeclared resource` | typo / wrong address / resource in another module | check `terraform state list`, module output names |
| `Invalid index` / `Missing resource instance key` | referenced `aws_subnet.x.id` but resource uses `count`/`for_each` | `aws_subnet.x[0].id`, `aws_subnet.x["key"].id`, or `[*]` |
| `Error acquiring the state lock` | parallel run or crashed run | wait; `force-unlock ID` once confirmed |
| `Backend configuration changed` | backend block edited | `init -reconfigure` or `-migrate-state` |
| `Provider produced inconsistent final plan` | value unknown at plan, e.g. `for_each` on computed set | use static keys; `-target` to create deps first; `apply` in two phases |
| `The "for_each" set includes values derived from resource attributes that cannot be determined until apply` | same | keys must be known at plan |
| `Cycle:` | mutual references (SG A ↔ SG B inline) | split into `aws_vpc_security_group_ingress_rule` resources |
| `InvalidParameterCombination` / AWS API 400 | provider passes bad combo (e.g. `associate_public_ip_address` with multiple ENIs) | read AWS error, fix resource args |
| `ResourceAlreadyExists` | created outside TF or lost state | `import` |
| Perpetual diff every plan | `timestamp()`, unordered lists, provider-normalized JSON, default tags | `ignore_changes`, `jsonencode`, sets, fix tag strategy |
| Replacement when you expected update | immutable attribute (name, AZ, `user_data`) | `create_before_destroy`, accept, or `ignore_changes` |
| `No valid credential sources` | missing profile/role, expired SSO | `aws sts get-caller-identity`, `AWS_PROFILE`, `assume_role` in provider |
| Subnet CIDR errors | `cidrsubnet` newbits too small / overlap | compute: `cidrsubnet("10.0.0.0/16", 8, 1)` = 10.0.1.0/24 |

Always run `terraform fmt && terraform validate && terraform plan` after each edit during the round.

## 12. Terraform in CI/CD

- PR: `fmt -check`, `validate`, `tflint`, `checkov`/`tfsec`, `plan` posted as PR comment (Atlantis / GitHub Actions). Merge: `apply` with saved plan, only from CI role, with manual approval environment for prod.
- OIDC trust from GitHub/GitLab to an IAM role scoped by repo + branch. No static keys.
- Pin Terraform version (`.terraform-version` / `tfenv`), commit `.terraform.lock.hcl`.
- Separate pipelines per state/component; `-target` only for break-glass.
- Policy as code: OPA/Conftest on `plan -json`, Sentinel in HCP Terraform.

## 13. Design decisions to be able to explain

- **Idempotency**: declarative diff; avoid provisioners and `timestamp()`.
- **Modularity**: modules = reusable units with contracts (inputs/outputs); envs compose them.
- **Blast radius**: state per component per environment; prevent_destroy on data stores.
- **Least privilege**: CI role per environment; apply-only from pipelines; read-only plan role for PRs.
- **Reproducibility**: pinned providers, lock file, versioned modules, saved plan.
- **Secrets**: data sources for Secrets Manager/SSM, sensitive outputs, encrypted state, ephemeral values.
- **Tagging**: `default_tags` + mandatory tags validated by Config rule (CostCenter, Owner, Env, DataClassification).

## 14. Quick-recall snippets

```hcl
# Latest AL2023 AMI
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name"   values = ["al2023-ami-2023*-x86_64"] }
  filter { name = "state"  values = ["available"] }
}

# Packer-built golden AMI by tag
data "aws_ami" "golden" {
  most_recent = true
  owners      = ["self"]
  filter { name = "tag:Role"     values = ["app-base"] }
  filter { name = "tag:Approved" values = ["true"] }
}

# IAM role for EC2 + SSM
data "aws_iam_policy_document" "ec2_trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals { type = "Service"  identifiers = ["ec2.amazonaws.com"] }
  }
}
resource "aws_iam_role" "app" {
  name               = "${local.name}-app"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust.json
}
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
resource "aws_iam_instance_profile" "app" { name = aws_iam_role.app.name  role = aws_iam_role.app.name }

# Launch template + ASG with instance refresh
resource "aws_launch_template" "app" {
  name_prefix   = "${local.name}-"
  image_id      = data.aws_ami.golden.id
  instance_type = var.instance_type
  iam_instance_profile { name = aws_iam_instance_profile.app.name }
  vpc_security_group_ids = [aws_security_group.app.id]
  metadata_options { http_tokens = "required" }
  user_data = base64encode(templatefile("${path.module}/user_data.sh", { env = var.env }))
  tag_specifications { resource_type = "instance"  tags = merge(local.tags, { Name = local.name }) }
}
resource "aws_autoscaling_group" "app" {
  name                = local.name
  min_size            = 2
  max_size            = 6
  desired_capacity    = 2
  vpc_zone_identifier = module.vpc.private_subnets
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  launch_template { id = aws_launch_template.app.id  version = "$Latest" }
  instance_refresh { strategy = "Rolling"  preferences { min_healthy_percentage = 90 } }
  tag { key = "Name"  value = local.name  propagate_at_launch = true }
}
```
