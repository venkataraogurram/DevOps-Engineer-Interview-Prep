# Terraform cheatsheet

[← Cheatsheet index](README.md) · Deep dive: [../03-refresher-terraform.md](../03-refresher-terraform.md)

## Commands

```bash
terraform init [-upgrade] [-reconfigure] [-migrate-state] [-backend-config=envs/prod.backend.hcl] [-backend=false]
terraform fmt -recursive [-check -diff]; terraform validate [-json]
terraform plan [-out=tfplan] [-var-file=envs/prod.tfvars] [-var 'x=1'] [-target=module.vpc] [-replace=aws_instance.app] [-refresh-only] [-destroy] [-detailed-exitcode: 0 none,1 err,2 changes] [-parallelism=10] [-lock-timeout=5m] [-compact-warnings]
terraform apply [tfplan | -auto-approve] ; terraform destroy [-target=...]
terraform show [tfplan] [-json]; terraform output [-json] [-raw name]
terraform state list | show ADDR | mv SRC DST | rm ADDR | pull > s.json | push s.json | replace-provider old new
terraform import ADDR ID; terraform import 'aws_subnet.x["a"]' subnet-123; terraform import 'module.vpc.aws_vpc.this[0]' vpc-123
terraform force-unlock LOCK_ID; terraform refresh (deprecated → apply -refresh-only); terraform taint (deprecated → -replace)
terraform workspace list|new|select|show|delete
terraform providers; terraform providers lock -platform=linux_amd64 -platform=darwin_arm64; terraform providers schema -json
terraform console  # > cidrsubnet("10.0.0.0/16",8,3)
terraform graph | dot -Tsvg > g.svg; terraform test [-filter=tests/x.tftest.hcl] [-verbose]
terraform login; terraform version; terraform -chdir=envs/prod plan
TF_LOG=TRACE|DEBUG|INFO TF_LOG_PATH=./tf.log TF_INPUT=0 TF_IN_AUTOMATION=1 TF_VAR_x=1 TF_CLI_ARGS_plan="-parallelism=5" TF_PLUGIN_CACHE_DIR=~/.terraform.d/plugin-cache TF_WORKSPACE=prod
tfenv install 1.9.5; tfenv use 1.9.5; cat .terraform-version
tflint --init; tflint --recursive; checkov -d . --framework terraform; tfsec .; terraform-docs markdown table . > README.md; infracost breakdown --path .
```

## HCL syntax

```hcl
terraform { required_version = ">= 1.6, < 2.0"  required_providers { aws = { source = "hashicorp/aws", version = "~> 5.60" } }  backend "s3" {} }
provider "aws" { region = var.region  alias = "dr"  assume_role { role_arn = "arn:..." }  default_tags { tags = local.tags } }
resource "TYPE" "NAME" { ... }                data "TYPE" "NAME" { ... }
variable "x" { type = string  default = ""  description = ""  sensitive = true  nullable = false  validation { condition = ...  error_message = "" } }
output "x" { value = ...  sensitive = true  description = ""  depends_on = [] }
locals { a = 1 }
module "m" { source = "..."  version = ""  providers = { aws = aws.dr }  count/for_each  depends_on = [] }
moved { from = aws_x.a  to = aws_x.b }   import { to = aws_x.a  id = "..." }   removed { from = aws_x.a  lifecycle { destroy = false } }
check "name" { assert { condition = ...  error_message = "" } }
```

Version constraints: `= 1.2`, `!= 1.2`, `>= 1.2`, `~> 1.2` (>=1.2,<2.0), `~> 1.2.0` (>=1.2.0,<1.3.0), `>= 1.2, < 1.5`.

Module sources: `./local`, `git::https://host/repo.git//sub/dir?ref=v1.2.0`, `git@github.com:org/repo.git//path?ref=`, `terraform-aws-modules/vpc/aws` (registry), `app.terraform.io/org/name/aws`, `s3::https://...`, `https://.../x.zip`.

## Expressions and functions

```hcl
# refs
aws_x.a.id  aws_x.a[0].id  aws_x.a["k"].id  aws_x.a[*].id  module.m.out  data.d.x.id  var.v  local.l  each.key each.value  count.index  self.x  path.module path.root path.cwd  terraform.workspace
# meta-args
count = N | for_each = map|set | depends_on | provider = aws.dr | lifecycle { create_before_destroy ignore_changes prevent_destroy replace_triggered_by precondition postcondition }
# conditionals / loops
cond ? a : b
[for x in list : upper(x) if x != ""]      { for k, v in map : k => v.id }      { for x in list : x.name => x }
[for i, x in list : "${i}-${x}"]           flatten([for s in subnets : [for p in ports : { s = s, p = p }]])
dynamic "block" { for_each = var.list  iterator = it  content { x = it.value.x } }
# strings
"${var.a}-${var.b}"  format("%s-%03d", n, i)  join(",", l)  split(",", s)  replace(s, "/re/", "x")  lower upper title trimspace trim(s,"-") trimprefix trimsuffix substr(s,0,5) startswith endswith strcontains regex("^(\\w+)", s) regexall can(regex(...)) chomp indent(2, s) templatefile(path, vars)
# numeric
min max ceil floor abs pow parseint("ff",16) sum([1,2])
# collections
length  element(l, i)  index(l, v)  contains(l, v)  concat(a, b)  distinct  flatten  keys values  lookup(m, k, default)  merge(a, b)  zipmap(keys, vals)  setunion setintersection setsubtract setproduct  slice(l, 0, 2)  reverse  sort  compact (drop "")  coalesce(a, b)  coalescelist  one(list)  alltrue anytrue  range(0, 5)  chunklist(l, 2)  transpose  tomap toset tolist tostring tonumber tobool
# encoding / files
jsonencode jsondecode yamlencode yamldecode base64encode base64decode base64gzip textencodebase64 urlencode csvdecode  file(path) fileexists filebase64 fileset(path, "**/*.yaml") filesha256  abspath dirname basename pathexpand
# network / crypto / date / misc
cidrsubnet("10.0.0.0/16", 8, 3)=10.0.3.0/24  cidrsubnets("10.0.0.0/16", 4, 4, 8)  cidrhost("10.0.0.0/24", 5)  cidrnetmask
md5 sha1 sha256 sha512 bcrypt uuid uuidv5 rsadecrypt  timestamp() timeadd(ts, "1h") formatdate("YYYY-MM-DD", ts) plantimestamp()
try(expr, fallback)  can(expr)  sensitive(x) nonsensitive(x)  type(x)  issensitive  ephemeralasnull
```

## Backend (S3)

```hcl
backend "s3" {
  bucket = "tfstate-ACCOUNT-REGION"  key = "component/env/terraform.tfstate"  region = "ap-south-1"
  encrypt = true  kms_key_id = "alias/tfstate"
  dynamodb_table = "terraform-locks"   # or use_lockfile = true (TF >= 1.10)
  assume_role = { role_arn = "arn:aws:iam::ACCOUNT:role/tf-state" }   # TF >= 1.6 syntax
}
```

Partial config: `backend "s3" {}` + `-backend-config=file.hcl`. Workspaces prefix key with `env:/NAME/`.

## Patterns

```hcl
# conditional resource
resource "aws_nat_gateway" "x" { count = var.create_nat ? 1 : 0 ... }   → ref: one(aws_nat_gateway.x[*].id)
# for_each from list of objects
for_each = { for r in var.rules : "${r.port}-${r.cidr}" => r }
# optional object attrs
variable "x" { type = object({ a = string, b = optional(number, 5) }) }
# multiple providers in module
module "dr" { source = "./m"  providers = { aws = aws.dr } }
# remote state read
data "terraform_remote_state" "net" { backend = "s3"  config = { bucket = "...", key = "...", region = "..." } }  → .outputs.vpc_id
# IAM policy doc
data "aws_iam_policy_document" "p" { statement { effect = "Allow"  actions = ["s3:GetObject"]  resources = ["${aws_s3_bucket.b.arn}/*"]  condition { test = "Bool"  variable = "aws:SecureTransport"  values = ["true"] } } }
# zero-downtime SG / LT
lifecycle { create_before_destroy = true }
# ASG rolling on AMI change
launch_template { id = aws_launch_template.x.id  version = aws_launch_template.x.latest_version }  instance_refresh { strategy = "Rolling" }
# protect data
lifecycle { prevent_destroy = true }
# null resource replacement
resource "terraform_data" "trigger" { input = var.version  triggers_replace = [var.version] }
```

## Testing (`tests/vpc.tftest.hcl`)

```hcl
variables { cidr = "10.0.0.0/16" }
run "plan_vpc" {
  command = plan
  assert { condition = aws_vpc.main.cidr_block == "10.0.0.0/16"  error_message = "wrong cidr" }
}
run "apply_vpc" { command = apply  assert { condition = length(aws_subnet.private) == 3  error_message = "need 3 subnets" } }
```

## Error → fix table (short)

| Error | Fix |
|---|---|
| `Backend configuration changed` | `init -reconfigure` / `-migrate-state` |
| `Error acquiring the state lock` | wait / `force-unlock ID` |
| `Invalid index` on `.id` | resource has count/for_each → `[0]` / `["k"]` / `[*]` |
| `for_each ... cannot be determined until apply` | static keys; `-target` first |
| `Cycle` | split inline SG rules into rule resources |
| `Provider produced inconsistent result` | provider bug; re-plan, pin version |
| `Variables not allowed` in backend | `-backend-config` |
| `Unsupported attribute` on module | check module outputs |
| `Resource already exists` | `import` |
| perpetual diff | `ignore_changes`, avoid `timestamp()`, sets vs lists, tag normalisation |
| `ExpiredToken` / `NoCredentialProviders` | `aws sts get-caller-identity`, profile/role |
| `Saved plan is stale` | re-plan |
