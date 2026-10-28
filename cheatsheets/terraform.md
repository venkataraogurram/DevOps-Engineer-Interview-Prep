# Terraform cheatsheet (annotated)

[← Cheatsheet index](README.md) · Deep dive: [../03-refresher-terraform.md](../03-refresher-terraform.md)

Format: command → what it does and when you reach for it.

## Core workflow

| Command | What it does / when to use |
|---|---|
| `terraform init` | Downloads providers and modules, sets up the backend, writes `.terraform.lock.hcl`. Run first in any new checkout or after changing providers/modules/backend. |
| `terraform init -upgrade` | Re-resolves provider/module versions to the newest allowed by constraints and updates the lock file. Use when bumping a provider version. |
| `terraform init -reconfigure` | Ignores the saved backend config and re-initialises from the current block. Use after editing the `backend` block when you don't want to migrate state. |
| `terraform init -migrate-state` | Copies existing state to the new backend. Use when moving from local to S3, or S3 bucket A to bucket B. |
| `terraform init -backend-config=envs/prod.backend.hcl` | Supplies backend values (bucket, key, region) from a file because the backend block can't use variables. Standard for per-environment directories. |
| `terraform init -backend=false` | Initialises providers only, skipping the backend. Use in CI validate jobs that have no AWS credentials. |
| `terraform fmt -recursive` | Rewrites all `.tf` files into canonical style. Run before committing; `-check -diff` in CI fails the build on unformatted code. |
| `terraform validate` | Checks syntax, types and internal references without touching APIs or state. Fast first gate in CI; `-json` for machine output. |
| `terraform plan -out=tfplan` | Refreshes state, computes the diff, saves it to a file. Saving the plan guarantees `apply` runs exactly what was reviewed. |
| `terraform plan -var-file=envs/prod.tfvars` | Loads variable values from a file. Files not named `terraform.tfvars`/`*.auto.tfvars` must be passed explicitly. |
| `terraform plan -var 'x=1'` | Sets one variable inline. Handy for ad hoc overrides; later `-var` wins over earlier. |
| `terraform plan -target=module.vpc` | Plans only that resource/module and its dependencies. Break-glass only (bootstrapping, untangling a stuck apply); never the normal workflow because it hides other drift. |
| `terraform plan -replace=aws_instance.app` | Forces recreation of a healthy-in-state but broken-in-reality resource. Replaces the old `taint` command. |
| `terraform plan -refresh-only` | Shows how state would change if you accepted real-world drift, without proposing infra changes. Use to review drift before accepting it. |
| `terraform plan -destroy` | Shows what `destroy` would remove. Review before tearing down an environment. |
| `terraform plan -detailed-exitcode` | Exit 0 = no changes, 1 = error, 2 = changes. Lets CI skip the apply stage when nothing changed, or alert on drift in a scheduled run. |
| `terraform plan -parallelism=10` | Limits concurrent resource operations (default 10). Lower it when the AWS API throttles you. |
| `terraform plan -lock-timeout=5m` | Waits up to 5 min for the state lock instead of failing immediately. Useful when pipelines overlap. |
| `terraform apply tfplan` | Applies a saved plan with no prompt. The CI/CD way. |
| `terraform apply -auto-approve` | Plans and applies without a prompt. Fine for dev sandboxes; avoid for prod because the applied plan was never reviewed. |
| `terraform apply -refresh-only` | Writes accepted drift into state without changing infrastructure. Use after reviewing `plan -refresh-only`. |
| `terraform destroy` | Destroys everything in state. Add `-target` to limit. Protect prod with `prevent_destroy` and no human apply rights. |
| `terraform show tfplan` | Renders a saved plan human-readably; `-json` gives the structure for OPA/Conftest policy checks or PR comments. |
| `terraform output` | Prints root outputs. `-json` for scripts, `-raw name` for a single plain value (e.g. `$(terraform output -raw vpc_id)`). |

## State surgery

| Command | What it does / when to use |
|---|---|
| `terraform state list` | Lists every resource address in state. First step when you need the exact address for `mv`/`rm`/`import`. |
| `terraform state show ADDR` | Shows all stored attributes of one resource. Use to see the real ID/ARN or to debug a perpetual diff. |
| `terraform state mv SRC DST` | Renames a resource in state without recreating it. Use when refactoring addresses on old Terraform; prefer `moved {}` blocks on ≥1.1 because they're reviewable in a PR. |
| `terraform state rm ADDR` | Forgets a resource (it keeps existing in AWS). Use when handing a resource to another stack or stopping management; on ≥1.7 prefer a `removed {}` block. |
| `terraform state pull > s.json` | Downloads the raw state. Take this backup before any manual state operation. |
| `terraform state push s.json` | Uploads a state file, overwriting the remote. Only for disaster recovery after a bad operation; S3 versioning is the safer rollback. |
| `terraform state replace-provider old new` | Rewrites provider addresses in state. Needed when a provider moves namespaces (e.g. `-/aws` → `hashicorp/aws`). |
| `terraform import ADDR ID` | Adopts an existing resource into state under the given address. Write the resource block first, import, then plan until clean. Quoting: `'aws_subnet.x["a"]'`, `'module.vpc.aws_vpc.this[0]'`. |
| `terraform force-unlock LOCK_ID` | Removes a stuck state lock. Only after confirming no apply is running anywhere, otherwise you corrupt state. |
| `terraform refresh` | Deprecated; updated state from reality immediately with no review. Use `apply -refresh-only` instead. |
| `terraform taint ADDR` | Deprecated; marked a resource for recreation. Use `apply -replace=ADDR`. |

## Workspaces, providers, inspection

| Command | What it does / when to use |
|---|---|
| `terraform workspace list / new X / select X / show / delete X` | Manages named state instances under the same backend. Good for short-lived variants; not for prod vs non-prod separation (same backend and creds). |
| `terraform providers` | Shows the provider tree and which module requires what. Use to find who pulls in an unexpected provider. |
| `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` | Writes lock-file hashes for several OS/arch combos so Mac devs and Linux CI share one lock file without churn. |
| `terraform providers schema -json` | Dumps every resource/attribute schema. Use to check an argument name without opening the docs. |
| `terraform console` | Interactive expression evaluator against current state/vars. Test `cidrsubnet("10.0.0.0/16", 8, 3)` or a `for` expression before putting it in code. |
| `terraform graph \| dot -Tsvg > g.svg` | Renders the dependency graph. Use to explain or debug ordering and cycles. |
| `terraform test` | Runs `.tftest.hcl` files (≥1.6): plan/apply with assertions. `-filter=tests/x.tftest.hcl` for one file, `-verbose` to print plans. |
| `terraform -chdir=envs/prod plan` | Runs the command as if in that directory. Lets pipelines stay at repo root. |
| `terraform login` | Stores a token for HCP Terraform/private registry. |
| `terraform version` | Shows CLI and provider versions. First thing to paste into a bug report. |

## Environment variables and tooling

| Setting | What it does / when to use |
|---|---|
| `TF_LOG=DEBUG TF_LOG_PATH=./tf.log` | Verbose provider/API logging to a file. Use when a provider error is opaque; TRACE for HTTP bodies. |
| `TF_INPUT=0` | Never prompt for missing variables; fail instead. Set in CI. |
| `TF_IN_AUTOMATION=1` | Suppresses "next step" hints in output. Set in CI. |
| `TF_VAR_name=value` | Sets a variable from the environment. The way to pass secrets from a CI vault without a tfvars file. |
| `TF_CLI_ARGS_plan="-parallelism=5"` | Adds default flags to a specific subcommand. Handy for enforcing `-lock-timeout` org-wide. |
| `TF_PLUGIN_CACHE_DIR=~/.terraform.d/plugin-cache` | Shares downloaded providers across roots. Big speed-up locally and in CI with a cache. |
| `TF_WORKSPACE=prod` | Selects a workspace without `workspace select`. Useful in pipelines. |
| `tfenv install 1.9.5 && tfenv use 1.9.5` | Installs/switches Terraform versions; reads `.terraform-version` in the repo so everyone matches. |
| `tflint --init && tflint --recursive` | Linter: unused declarations, invalid instance types, naming rules. Run in PR checks. |
| `checkov -d . --framework terraform` / `tfsec .` | Static security scanning (open SGs, unencrypted buckets). Run in PR checks; fail on HIGH. |
| `terraform-docs markdown table . > README.md` | Generates inputs/outputs docs for a module. Run in pre-commit so module READMEs never go stale. |
| `infracost breakdown --path .` | Estimates monthly cost of the plan. Post to PRs for cost-aware reviews. |

## HCL block reference

```hcl
terraform {                                   # settings for this root/module
  required_version = ">= 1.6, < 2.0"          # pin the CLI range so CI and laptops match
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.60" }   # ~> 5.60 = >=5.60,<6.0: patches/minors ok, no major jumps
  }
  backend "s3" {}                             # empty: values come from -backend-config at init
}

provider "aws" {
  region = var.region
  assume_role { role_arn = "arn:aws:iam::123:role/tf" }   # run as a role in the target account
  default_tags { tags = local.tags }          # every taggable resource gets these without repeating
}
provider "aws" { alias = "dr"  region = "ap-south-2" }     # second provider instance; refer to as aws.dr

resource "TYPE" "NAME" { ... }                # something Terraform creates and owns
data "TYPE" "NAME" { ... }                    # read-only lookup of something that already exists

variable "x" {
  type        = string                        # enforce a type; object({}) with optional() for structured input
  default     = ""                            # omit default to make it required
  description = ""                            # shows in docs and errors
  sensitive   = true                          # redacted in CLI output (still in state)
  nullable    = false                         # reject explicit null
  validation { condition = ...  error_message = "" }   # fail early on bad input
}
output "x" { value = ...  sensitive = true  description = "" }   # expose values to callers / CLI
locals { a = 1 }                              # computed constants; avoid repeating expressions

module "m" {
  source  = "..."                             # see sources below
  version = ""                                # registry modules only
  providers = { aws = aws.dr }                # pass an aliased provider into the module
  count / for_each                            # multiple instances of a module
  depends_on = []                             # explicit ordering when no attribute reference exists
}

moved   { from = aws_x.a  to = aws_x.b }      # rename without destroy/create; delete after everyone applied
import  { to = aws_x.a  id = "..." }          # declarative import (≥1.5); add -generate-config-out to draft config
removed { from = aws_x.a  lifecycle { destroy = false } }   # stop managing without destroying (≥1.7)
check "name" { assert { condition = ...  error_message = "" } }   # post-apply assertion that warns, doesn't block
```

Module sources: `./local` (same repo) · `git::https://host/repo.git//sub/dir?ref=v1.2.0` (pin a tag) · `git@github.com:org/repo.git//path?ref=` (SSH) · `terraform-aws-modules/vpc/aws` (public registry, needs `version`) · `app.terraform.io/org/name/aws` (private registry) · `s3::https://...` / `https://.../x.zip` (archives).

## Expressions you'll write in the coding round

| Expression | What it does / when to use |
|---|---|
| `aws_x.a.id` | Attribute of a single resource. |
| `aws_x.a[0].id` / `aws_x.a["k"].id` | Instance of a `count` / `for_each` resource. Forgetting the index is the #1 "Invalid index" error. |
| `aws_x.a[*].id` | Splat: list of an attribute across all `count` instances. For `for_each` use `values(aws_x.a)[*].id` or a `for`. |
| `module.m.out` | A module output. The module must declare it; `module.vpc.id` with no such output is "Unsupported attribute". |
| `each.key` / `each.value` | Current item in a `for_each` resource. |
| `count.index` | Current index in a `count` resource. |
| `self.x` | The resource's own attribute, inside `postcondition`/provisioner/`connection`. |
| `path.module` / `path.root` / `path.cwd` | Directory paths for `file()`/`templatefile()`. Use `path.module` inside modules so they work from anywhere. |
| `terraform.workspace` | Current workspace name; use in names/tags if you use workspaces. |
| `count = var.enabled ? 1 : 0` | Conditional resource. Reference it as `one(aws_x.y[*].id)` to get null-or-value. |
| `for_each = { for r in var.rules : "${r.port}-${r.cidr}" => r }` | Turn a list of objects into a map with stable keys so adding/removing one doesn't reshuffle the rest. |
| `[for x in list : upper(x) if x != ""]` | Transform and filter a list. |
| `{ for k, v in map : k => v.id }` | Build a map from another map. |
| `flatten([for s in subnets : [for p in ports : { s = s, p = p }]])` | Cartesian product for e.g. one rule per subnet × port. |
| `dynamic "ingress" { for_each = var.rules  content { ... } }` | Generate repeated nested blocks. Prefer standalone rule resources where they exist. |
| `depends_on = [aws_iam_role_policy_attachment.x]` | Explicit dependency. Only when there's no attribute to reference (classic: IAM attachment before EKS node group). |
| `lifecycle { create_before_destroy = true }` | Build replacement first, then destroy old. Zero-downtime for SGs, launch templates, certs. |
| `lifecycle { prevent_destroy = true }` | Plan fails if this would be destroyed. For databases, state buckets, KMS keys. |
| `lifecycle { ignore_changes = [desired_capacity] }` | Don't fight something managed elsewhere (autoscaling, a Lambda deploy tool). |
| `lifecycle { replace_triggered_by = [aws_launch_template.x.latest_version] }` | Recreate when another resource changes. |
| `precondition` / `postcondition` | Assertions evaluated at plan/apply; fail with a clear message instead of a cryptic API error. |
| `provider = aws.dr` | Create this resource with the aliased provider (other region/account). |

## Functions (grouped)

| Group | Functions and typical use |
|---|---|
| Strings | `format("%s-%03d", n, i)` zero-padded names · `join(",", l)` / `split(",", s)` · `replace(s, "/re/", "x")` regex replace · `lower/upper/title` · `trimspace/trim/trimprefix/trimsuffix` · `substr(s,0,5)` · `startswith/endswith/strcontains` · `regex("^(\\w+)", s)` capture · `can(regex(...))` validation · `indent(2, s)` for YAML · `templatefile(path, vars)` render user data / policies |
| Numeric | `min/max/ceil/floor/abs/pow` · `parseint("ff",16)` · `sum([1,2])` |
| Collections | `length` · `element(l, i)` wraps around (use for round-robin AZs) · `index(l, v)` · `contains(l, v)` validation · `concat/distinct/flatten` · `keys/values` · `lookup(m, k, default)` safe map read · `merge(a, b)` combine tag maps · `zipmap(keys, vals)` · `setunion/setintersection/setsubtract/setproduct` · `slice(l, 0, 2)` · `reverse/sort` · `compact` drops empty strings · `coalesce(a, b)` first non-null · `one(list)` 0-or-1 → value/null · `alltrue/anytrue` · `range(0, 5)` · `chunklist(l, 2)` · `tomap/toset/tolist/tostring/tonumber/tobool` |
| Encoding/files | `jsonencode` IAM policies · `jsondecode` parse API output · `yamlencode/yamldecode` K8s manifests · `base64encode` user data · `file(path)` read a file · `fileexists` · `fileset(path, "**/*.yaml")` iterate files · `filesha256` trigger on content change · `abspath/dirname/basename/pathexpand` |
| Network | `cidrsubnet("10.0.0.0/16", 8, 3)` = 10.0.3.0/24 (add 8 bits, take the 3rd) · `cidrsubnets("10.0.0.0/16", 4, 4, 8)` several at once · `cidrhost("10.0.0.0/24", 5)` nth IP · `cidrnetmask` |
| Crypto/date | `md5/sha1/sha256/sha512` · `bcrypt` · `uuid()` (changes every plan – avoid in resources) · `timestamp()` (same warning) · `timeadd` · `formatdate("YYYY-MM-DD", ts)` · `plantimestamp()` stable within a plan |
| Safety | `try(expr, fallback)` tolerate missing attributes · `can(expr)` boolean test · `sensitive/nonsensitive` mark/unmark · `type(x)` in console · `ephemeralasnull` |

## Backend (S3)

```hcl
backend "s3" {
  bucket         = "tfstate-ACCOUNT-REGION"             # one bucket per account; versioning on for recovery
  key            = "component/env/terraform.tfstate"   # one key per component per env = small blast radius
  region         = "ap-south-1"
  encrypt        = true                                 # SSE on the state object (state holds secrets)
  kms_key_id     = "alias/tfstate"                      # CMK so access is auditable and restrictable
  dynamodb_table = "terraform-locks"                    # locking; table needs hash key LockID (String)
  # use_lockfile = true                                 # TF >= 1.10: S3-native lock, DynamoDB becomes optional
  assume_role    = { role_arn = "arn:aws:iam::ACCOUNT:role/tf-state" }   # state in a different account
}
```

Partial config: leave `backend "s3" {}` empty and pass `-backend-config=envs/prod.backend.hcl` so the same code serves every environment. Workspaces store state under `env:/NAME/key`.

## Patterns (copy and adapt)

```hcl
# Latest approved golden AMI by tag – pin the ID in prod tfvars instead if you want explicit promotion
data "aws_ami" "golden" {
  most_recent = true
  owners      = ["self"]
  filter { name = "tag:Role"     values = ["base"] }
  filter { name = "tag:Approved" values = ["true"] }
}

# Read another stack's outputs without coupling apply order
data "terraform_remote_state" "net" {
  backend = "s3"
  config  = { bucket = "...", key = "network/prod/terraform.tfstate", region = "ap-south-1" }
}  # → data.terraform_remote_state.net.outputs.vpc_id

# IAM policy as a data source: validated structure, no JSON string escaping
data "aws_iam_policy_document" "p" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.b.arn}/*"]
    condition { test = "Bool"  variable = "aws:SecureTransport"  values = ["true"] }
  }
}

# Optional object attributes with defaults – callers pass only what they need
variable "x" { type = object({ a = string, b = optional(number, 5) }) }

# ASG that rolls when the launch template changes
launch_template { id = aws_launch_template.x.id  version = aws_launch_template.x.latest_version }
instance_refresh { strategy = "Rolling"  preferences { min_healthy_percentage = 90 } }

# Replacement for null_resource: re-run when input changes
resource "terraform_data" "trigger" { input = var.version  triggers_replace = [var.version] }
```

## Testing (`tests/vpc.tftest.hcl`)

```hcl
variables { cidr = "10.0.0.0/16" }          # inputs for every run block
run "plan_vpc" {
  command = plan                              # plan-only: fast, no credentials for cloud resources needed beyond provider init
  assert { condition = aws_vpc.main.cidr_block == "10.0.0.0/16"  error_message = "wrong cidr" }
}
run "apply_vpc" {
  command = apply                             # real apply in a sandbox account, auto-destroyed after the test
  assert { condition = length(aws_subnet.private) == 3  error_message = "need 3 subnets" }
}
```

## Error → fix

| Error | Cause → fix |
|---|---|
| `Backend configuration changed` | You edited the backend block → `init -reconfigure` (keep state where it is) or `-migrate-state` (move it). |
| `Error acquiring the state lock` | Another run holds the lock or a run crashed → wait; if confirmed stale, `force-unlock ID`. |
| `Invalid index` / `Missing resource instance key` | Referenced a `count`/`for_each` resource without an index → `[0]`, `["k"]`, or `[*]`. |
| `for_each ... cannot be determined until apply` | Keys depend on computed values → key on inputs you control; `-target` the dependency first as a one-off. |
| `Cycle: ...` | Two resources reference each other (inline SG rules) → split into `aws_vpc_security_group_ingress_rule` resources. |
| `Provider produced inconsistent result after apply` | Provider bug or eventual consistency → re-run plan; pin/upgrade provider. |
| `Variables not allowed` (backend) | Backend block can't use `var.` → `-backend-config`. |
| `Unsupported attribute` on `module.x.y` | Module has no output `y` → check `outputs.tf` of the module. |
| `... already exists` from AWS | Created outside Terraform or state lost → `terraform import`. |
| Plan always shows changes | `timestamp()`/`uuid()` in a resource, list vs set ordering, provider normalising JSON/tags → `ignore_changes`, use sets, `jsonencode`. |
| `ExpiredToken` / `NoCredentialProviders` | Credentials expired or wrong profile → `aws sts get-caller-identity`, re-login, check `AWS_PROFILE`. |
| `Saved plan is stale` | State changed since the plan was saved → re-plan. |
| `# forces replacement` on an attribute | Immutable attribute changed (name, AZ, user_data) → accept, `create_before_destroy`, or `ignore_changes`. |
