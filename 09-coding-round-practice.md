# Coding round practice – "half-baked" Terraform and Kubernetes

The recruiter's sheet says you'll get partially completed code and be judged on correctness, reasoning and best practices. Each exercise below has the starter (with deliberate gaps and bugs), what to say out loud, the solution, and the commands to verify. Do them cold first, then compare.

How to work during the round:
1. Read the whole file first. List the gaps and bugs aloud before typing.
2. `terraform fmt && terraform validate` / `kubectl apply --dry-run=client -f` after each change.
3. Narrate trade-offs: "I'm using `for_each` instead of `count` because…".
4. Finish with the verification commands even if you can't run them.

---

## Exercise 1 – Terraform: complete a VPC with public/private subnets

### Starter (given)

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws" }
  }
}

provider "aws" {}

variable "vpc_cidr" { default = "10.20.0.0/16" }
variable "azs" { default = ["ap-south-1a", "ap-south-1b"] }

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.20.1.0/24"
}

resource "aws_internet_gateway" "igw" {}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
  }
}

# TODO: private subnets, NAT, route table associations, outputs
```

### What to say
- Provider has no region/version pin; IGW not attached to the VPC; route has no `gateway_id`; only one hardcoded public subnet, no AZ; no DNS hostnames on VPC (needed for EKS/RDS endpoints); no associations; no private tier; no tags; no outputs.
- Decide: `for_each` over AZs with `cidrsubnet`; one NAT per AZ for HA (or one NAT for cost in non-prod – state the choice).

### Solution

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = var.region
  default_tags { tags = { Project = var.project, Env = var.env, ManagedBy = "terraform" } }
}

variable "region"   { type = string  default = "ap-south-1" }
variable "project"  { type = string  default = "idfc-platform" }
variable "env"      { type = string  default = "dev" }
variable "vpc_cidr" { type = string  default = "10.20.0.0/16" }
variable "azs"      { type = list(string)  default = ["ap-south-1a", "ap-south-1b"] }
variable "single_nat" { type = bool  default = true  description = "One NAT GW for non-prod cost saving" }

locals {
  name = "${var.project}-${var.env}"
  az_map = { for i, az in var.azs : az => i }          # az => index, stable keys
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = local.name }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name}-igw" }
}

resource "aws_subnet" "public" {
  for_each                = local.az_map
  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.key
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, each.value)          # 10.20.0.0/24, 10.20.1.0/24
  map_public_ip_on_launch = false                                            # explicit; ALB/NAT don't need it
  tags = { Name = "${local.name}-public-${each.key}", Tier = "public", "kubernetes.io/role/elb" = "1" }
}

resource "aws_subnet" "private" {
  for_each          = local.az_map
  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, each.value + 10)           # 10.20.10.0/24, 10.20.11.0/24
  tags = { Name = "${local.name}-private-${each.key}", Tier = "private", "kubernetes.io/role/internal-elb" = "1" }
}

resource "aws_eip" "nat" {
  for_each = var.single_nat ? { (var.azs[0]) = 0 } : local.az_map
  domain   = "vpc"
  tags     = { Name = "${local.name}-nat-${each.key}" }
}

resource "aws_nat_gateway" "nat" {
  for_each      = aws_eip.nat
  allocation_id = each.value.id
  subnet_id     = aws_subnet.public[each.key].id
  tags          = { Name = "${local.name}-nat-${each.key}" }
  depends_on    = [aws_internet_gateway.igw]
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name}-public" }
}
resource "aws_route" "public_default" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}
resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  for_each = local.az_map
  vpc_id   = aws_vpc.main.id
  tags     = { Name = "${local.name}-private-${each.key}" }
}
resource "aws_route" "private_default" {
  for_each               = local.az_map
  route_table_id         = aws_route_table.private[each.key].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = var.single_nat ? aws_nat_gateway.nat[var.azs[0]].id : aws_nat_gateway.nat[each.key].id
}
resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private[each.key].id
}

output "vpc_id"             { value = aws_vpc.main.id }
output "public_subnet_ids"  { value = [for s in aws_subnet.public : s.id] }
output "private_subnet_ids" { value = [for s in aws_subnet.private : s.id] }
```

### Verify
```bash
terraform fmt -check && terraform validate && terraform plan
# after apply:
aws ec2 describe-route-tables --filters Name=vpc-id,Values=$(terraform output -raw vpc_id) --query 'RouteTables[].Routes[?DestinationCidrBlock==`0.0.0.0/0`].[GatewayId,NatGatewayId]'
```

---

## Exercise 2 – Terraform: fix the security groups and module wiring

### Starter (given)

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.30.0.0/16"
}

resource "aws_security_group" "alb" {
  name   = "alb"
  vpc_id = module.vpc.id
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app" {
  name   = "app"
  vpc_id = module.vpc.id
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "db" {
  name   = "db"
  vpc_id = module.vpc.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app]
  }
}

# modules/vpc/outputs.tf
output "vpc_id" { value = aws_vpc.this.id }
```

### What to say
- `module.vpc.id` doesn't exist; the module outputs `vpc_id`.
- `security_groups = [aws_security_group.app]` passes the object, needs `.id`.
- App tier open to the world on 8080 and SSH on 22 from 0.0.0.0/0 – should allow 8080 only from the ALB SG and drop 22 entirely (SSM). ALB can be restricted to corporate CIDRs for an internal bank app.
- No egress blocks: AWS provider removes the default allow-all egress when you manage the SG, so the ALB can't reach targets. Add explicit egress.
- `name` fixed → conflicts across environments; use `name_prefix` or prefix with env. Inline rules + SG cross-references risk cycles; prefer standalone rule resources.

### Solution

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = var.vpc_cidr
}

resource "aws_security_group" "alb" {
  name_prefix = "${local.name}-alb-"
  description = "Internal ALB"
  vpc_id      = module.vpc.vpc_id
  lifecycle { create_before_destroy = true }
}
resource "aws_vpc_security_group_ingress_rule" "alb_https" {
  for_each          = toset(var.allowed_cidrs)       # e.g. ["10.0.0.0/8"] corporate
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = each.value
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
}
resource "aws_vpc_security_group_egress_rule" "alb_to_app" {
  security_group_id            = aws_security_group.alb.id
  referenced_security_group_id = aws_security_group.app.id
  from_port                    = 8080
  to_port                      = 8080
  ip_protocol                  = "tcp"
}

resource "aws_security_group" "app" {
  name_prefix = "${local.name}-app-"
  description = "App tier"
  vpc_id      = module.vpc.vpc_id
  lifecycle { create_before_destroy = true }
}
resource "aws_vpc_security_group_ingress_rule" "app_from_alb" {
  security_group_id            = aws_security_group.app.id
  referenced_security_group_id = aws_security_group.alb.id
  from_port                    = 8080
  to_port                      = 8080
  ip_protocol                  = "tcp"
}
resource "aws_vpc_security_group_egress_rule" "app_all" {      # or restrict to db SG + VPC endpoints
  security_group_id = aws_security_group.app.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

resource "aws_security_group" "db" {
  name_prefix = "${local.name}-db-"
  description = "PostgreSQL"
  vpc_id      = module.vpc.vpc_id
  lifecycle { create_before_destroy = true }
}
resource "aws_vpc_security_group_ingress_rule" "db_from_app" {
  security_group_id            = aws_security_group.db.id
  referenced_security_group_id = aws_security_group.app.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}
# no egress on db SG = no outbound (deliberate)
```

---

## Exercise 3 – Terraform: configure remote state with locking and a tfvars-driven environment

### Starter (given)

```hcl
terraform {
  backend "s3" {
    bucket = var.state_bucket
    key    = "terraform.tfstate"
  }
}

variable "environment" {}
variable "instance_type" {}

resource "aws_instance" "app" {
  ami           = "ami-0abcdef1234567890"
  instance_type = var.instance_type
  tags = { Name = "app-${environment}" }
}
```

### What to say
- Backend blocks can't reference variables → use `-backend-config` file or literals. Missing `region`, `encrypt`, locking (`dynamodb_table` or `use_lockfile`). Key should include component + env so envs don't share state.
- `${environment}` must be `${var.environment}`. Variables lack types/descriptions.
- Hardcoded AMI → `data "aws_ami"` or SSM parameter; no subnet, no SG, no instance profile, IMDSv2 not enforced, root volume unencrypted.
- Show the tfvars + how to run per env.

### Solution

```hcl
# backend.tf
terraform {
  required_version = ">= 1.6"
  required_providers { aws = { source = "hashicorp/aws", version = "~> 5.0" } }
  backend "s3" {}                                   # values from -backend-config
}
```
```hcl
# envs/prod.backend.hcl
bucket         = "idfc-tfstate-123456789012-ap-south-1"
key            = "apps/payments/prod/terraform.tfstate"
region         = "ap-south-1"
encrypt        = true
dynamodb_table = "terraform-locks"
# use_lockfile = true     # TF >= 1.10 alternative
```
```hcl
# variables.tf
variable "region"        { type = string }
variable "environment"   { type = string  validation { condition = contains(["dev","staging","prod"], var.environment)  error_message = "dev|staging|prod" } }
variable "instance_type" { type = string }
variable "subnet_id"     { type = string }
variable "ami_ssm_param" { type = string  default = "/ami/al2023/base/latest" }
```
```hcl
# envs/prod.tfvars
region        = "ap-south-1"
environment   = "prod"
instance_type = "m6i.large"
subnet_id     = "subnet-0123456789abcdef0"
```
```hcl
# main.tf
provider "aws" { region = var.region  default_tags { tags = { Env = var.environment, ManagedBy = "terraform" } } }

data "aws_ssm_parameter" "ami" { name = var.ami_ssm_param }

resource "aws_instance" "app" {
  ami                    = data.aws_ssm_parameter.ami.value
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile   = aws_iam_instance_profile.app.name
  metadata_options { http_tokens = "required"  http_endpoint = "enabled" }
  root_block_device { encrypted = true  volume_type = "gp3" }
  tags = { Name = "app-${var.environment}" }
  lifecycle { ignore_changes = [ami] }       # new AMI rolls via explicit replace, not surprise plan
}
```
```bash
terraform init -backend-config=envs/prod.backend.hcl -reconfigure
terraform plan -var-file=envs/prod.tfvars -out=tfplan
terraform apply tfplan
```

Bootstrap resources (separate root, once):
```hcl
resource "aws_s3_bucket" "tfstate" { bucket = "idfc-tfstate-${data.aws_caller_identity.me.account_id}-${var.region}" }
resource "aws_s3_bucket_versioning" "v" { bucket = aws_s3_bucket.tfstate.id  versioning_configuration { status = "Enabled" } }
resource "aws_s3_bucket_server_side_encryption_configuration" "e" { bucket = aws_s3_bucket.tfstate.id  rule { apply_server_side_encryption_by_default { sse_algorithm = "aws:kms" } } }
resource "aws_s3_bucket_public_access_block" "p" { bucket = aws_s3_bucket.tfstate.id  block_public_acls = true  block_public_policy = true  ignore_public_acls = true  restrict_public_buckets = true }
resource "aws_dynamodb_table" "locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute { name = "LockID"  type = "S" }
  point_in_time_recovery { enabled = true }
}
```

---

## Exercise 4 – Kubernetes: inject a Secret, add probes/resources, make it rollable

### Starter (given)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payments
    spec:
      containers:
      - name: payments
        image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/payments:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "SuperSecret123"
---
apiVersion: v1
kind: Service
metadata:
  name: payments
spec:
  selector:
    app: payments
  ports:
  - port: 8080
    targetPort: 80
```

### What to say
- Selector `app: payment` ≠ template label `app: payments` → apply fails ("selector does not match template labels").
- Plaintext password in manifest → Secret + `secretKeyRef`.
- `:latest` → pin tag; `imagePullPolicy` implicit Always with latest.
- `replicas: 1`, no strategy, no probes, no resources, no securityContext → not production-ready, not zero-downtime.
- Service `port: 8080 targetPort: 80` is backwards; container listens on 8080. Add a namespace. Mention RBAC/ServiceAccount.

### Solution

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
---
apiVersion: v1
kind: Secret
metadata:
  name: payments-db
  namespace: payments
type: Opaque
stringData:
  DB_PASSWORD: "SuperSecret123"     # in reality: External Secrets / Secrets Manager, never in git
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-config
  namespace: payments
data:
  LOG_LEVEL: info
  DB_HOST: payments-db.cluster-xyz.ap-south-1.rds.amazonaws.com
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments
  namespace: payments
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: payments
  labels:
    app: payments
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payments
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: payments
    spec:
      serviceAccountName: payments
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: payments
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/payments:1.8.3
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payments-db
                  key: DB_PASSWORD
          envFrom:
            - configMapRef:
                name: payments-config
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: payments
  namespace: payments
spec:
  type: ClusterIP
  selector:
    app: payments
  ports:
    - name: http
      port: 80
      targetPort: http
```

### Verify
```bash
kubectl apply -f payments.yaml --dry-run=client
kubectl apply -f payments.yaml --dry-run=server
kubectl apply -f payments.yaml
kubectl -n payments rollout status deploy/payments --timeout=120s
kubectl -n payments get pods -o wide
kubectl -n payments exec deploy/payments -- sh -c 'echo $DB_PASSWORD | cut -c1-3; env | grep -E "LOG_LEVEL|DB_HOST"'
kubectl -n payments get endpoints payments            # should list 3 pod IPs
kubectl -n payments describe pod -l app=payments | grep -A3 -E "Liveness|Readiness|Limits|Requests"
# rolling update test
kubectl -n payments set image deploy/payments payments=…/payments:1.8.4 && kubectl -n payments rollout status deploy/payments
kubectl -n payments rollout history deploy/payments
kubectl -n payments rollout undo deploy/payments
```

---

## Exercise 5 – Kubernetes: RBAC for a read-only CI service account

### Task (given as text)
"A CI job in namespace `ci` must be able to read Deployments and Pods in namespace `payments`, trigger `kubectl rollout restart`, but must not read Secrets."

### What to say
- `rollout restart` patches the Deployment (adds an annotation) → needs `patch` on `deployments`. Reading pods needs `get/list/watch` on `pods` (and `pods/log` if logs needed). Role in `payments` namespace, RoleBinding to the SA in `ci`. Explicitly no `secrets` resource.

### Solution

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: ci
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payments-deployer
  namespace: payments
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer
  namespace: payments
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: ci
roleRef:
  kind: Role
  name: payments-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Verify
```bash
kubectl auth can-i patch deployments -n payments --as=system:serviceaccount:ci:ci-deployer     # yes
kubectl auth can-i get secrets -n payments --as=system:serviceaccount:ci:ci-deployer           # no
kubectl auth can-i --list -n payments --as=system:serviceaccount:ci:ci-deployer
```

---

## Exercise 6 – Debug: "Deployment applied, Service returns 503"

### Given
```
$ kubectl get pods -n shop
NAME                     READY   STATUS    RESTARTS   AGE
web-7d9f8c6b5d-abcde     0/1     Running   0          3m
web-7d9f8c6b5d-fghij     0/1     Running   0          3m
$ kubectl get endpoints web -n shop
NAME   ENDPOINTS   AGE
web    <none>      3m
```

### Walk-through to narrate
1. `0/1 Running` with no restarts → readiness probe failing (liveness would restart). Endpoints empty confirms it.
2. `kubectl describe pod web-… -n shop` → Events: `Readiness probe failed: HTTP probe failed with statuscode: 404` (wrong path) or `connection refused` (wrong port / app not listening on 0.0.0.0) or timeout (probe too tight).
3. `kubectl logs web-… -n shop` → app listening port; `kubectl exec … -- wget -qO- localhost:8080/ready`.
4. Also check `kubectl get svc web -o yaml` selector vs pod labels (`kubectl get pods --show-labels`) – a label mismatch gives the same empty endpoints with pods Ready.
5. Fix the probe path/port or labels, `kubectl apply`, `rollout status`, `get endpoints` shows IPs, `kubectl port-forward svc/web 8080:80` → 200.

Common root causes to list: probe path/port, selector mismatch, container binding to 127.0.0.1, `targetPort` mismatch, NetworkPolicy blocking kubelet probes (no – probes come from kubelet on the node, not subject to NetworkPolicy, but ALB health checks are), app needs a secret that is missing (check logs).

---

## Bonus: Terraform debugging drills (read the error, say the fix)

1. `Error: Invalid index ... aws_subnet.private is tuple with 2 elements` → used `aws_subnet.private.id` on a `count` resource; use `[0]` or `[*]`.
2. `Error: Cycle: aws_security_group.a, aws_security_group.b` → inline cross-referencing SG rules; split into `aws_vpc_security_group_ingress_rule`.
3. `Error: creating EC2 Instance: InvalidParameterCombination: Network interfaces and an instance-level security groups may not be specified` → used both `network_interface` and `vpc_security_group_ids`.
4. `Error: Error acquiring the state lock` → check DynamoDB item / other pipeline; `force-unlock <ID>` only if confirmed stale.
5. `Error: Unsupported attribute: This object does not have an attribute named "id"` on `module.vpc.id` → check module outputs.
6. `Error: Variables not allowed ... in backend configuration` → `-backend-config`.
7. Plan shows `# forces replacement` on `user_data` → expected; use `create_before_destroy` or ASG + launch template versioning instead of `aws_instance`.
8. `Error: creating S3 Bucket: BucketAlreadyExists` → global namespace; add account ID/region to the name.
9. `Error: Provider configuration not present` after removing a module that used an aliased provider → re-add provider temporarily to destroy, or `state rm`.
10. `Error: InvalidClientTokenId / ExpiredToken` → credentials; `aws sts get-caller-identity`.
