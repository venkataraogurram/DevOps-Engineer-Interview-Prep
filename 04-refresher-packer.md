# Refresher – Packer (named in the JD; likely your gap)

## 1. What it is and why a bank uses it

Packer builds **machine images** (AMIs, Docker images, VMware, Azure) from a template. Run once → immutable, pre-hardened, pre-patched image. Terraform then launches instances from that AMI.

Why banks care:
- **Immutable infrastructure**: no config drift; replace, don't patch in place.
- **Compliance baked in**: CIS hardening, agents (CloudWatch, SSM, EDR/AV, Qualys), approved packages, all verified before the image is "approved".
- **Faster boot/scale-out**: nothing to install at launch; user data only does environment wiring.
- **Audit**: image has a manifest and tags (source AMI, build date, commit, CVE scan result).

Packer builds; Terraform deploys. Packer **does not** manage running instances.

## 2. How a build works

1. **Builder** (plugin `source`) launches a temporary EC2 from a base AMI in your VPC/subnet with a temporary SG and key pair (or SSM session).
2. **Provisioners** run in order on that instance: shell, Ansible, file upload, PowerShell.
3. Packer stops the instance, creates an AMI (+ snapshots), terminates the builder instance.
4. **Post-processors** run locally: manifest JSON, copy/share AMI, trigger a scan.

## 3. HCL2 template (memorize the shape)

```hcl
# al2023-base.pkr.hcl
packer {
  required_plugins {
    amazon = { source = "github.com/hashicorp/amazon", version = ">= 1.3.0" }
  }
}

variable "region"        { type = string  default = "ap-south-1" }
variable "subnet_id"     { type = string }
variable "vpc_id"        { type = string }
variable "instance_type" { type = string  default = "t3.small" }
variable "env"           { type = string  default = "shared" }

locals {
  timestamp = formatdate("YYYYMMDD-hhmm", timestamp())
  ami_name  = "idfc-al2023-base-${local.timestamp}"
}

data "amazon-ami" "al2023" {
  region      = var.region
  owners      = ["amazon"]
  most_recent = true
  filters = {
    name                = "al2023-ami-2023*-x86_64"
    virtualization-type = "hvm"
    root-device-type    = "ebs"
  }
}

source "amazon-ebs" "base" {
  region        = var.region
  source_ami    = data.amazon-ami.al2023.id
  instance_type = var.instance_type
  ami_name      = local.ami_name
  ami_description = "AL2023 hardened base, CIS L1, CW+SSM agents"

  vpc_id    = var.vpc_id
  subnet_id = var.subnet_id          # private subnet; no public IP
  associate_public_ip_address = false
  ssh_username  = "ec2-user"
  ssh_interface = "session_manager"  # build over SSM, no port 22 opened
  iam_instance_profile = "packer-builder"   # needs AmazonSSMManagedInstanceCore
  temporary_key_pair_type = "ed25519"

  encrypt_boot = true
  kms_key_id   = "alias/ami-encryption"
  launch_block_device_mappings {
    device_name = "/dev/xvda"
    volume_size = 20
    volume_type = "gp3"
    delete_on_termination = true
  }

  ami_regions = ["ap-south-1", "ap-south-2"]          # copy for DR
  ami_users   = ["111111111111", "222222222222"]      # share with workload accounts
  # ami_org_arns = ["arn:aws:organizations::123456789012:organization/o-xxxx"]

  tags = {
    Name        = local.ami_name
    Role        = "base"
    OS          = "al2023"
    SourceAMI   = "{{ .SourceAMI }}"
    BuildDate   = local.timestamp
    GitCommit   = env("GIT_COMMIT")
    Approved    = "false"          # flipped to true by the scan stage
  }
  run_tags = { Name = "packer-build-${local.ami_name}" }
}

build {
  name    = "al2023-base"
  sources = ["source.amazon-ebs.base"]

  provisioner "shell" {
    inline = [
      "sudo dnf -y update",
      "sudo dnf -y install amazon-cloudwatch-agent",
      "sudo systemctl enable amazon-ssm-agent",
    ]
  }

  provisioner "file" {
    source      = "files/cloudwatch-agent.json"
    destination = "/tmp/cloudwatch-agent.json"
  }

  provisioner "shell" {
    scripts = ["scripts/harden-cis.sh", "scripts/install-agents.sh", "scripts/cleanup.sh"]
    execute_command = "chmod +x {{ .Path }}; sudo -E sh -c '{{ .Vars }} {{ .Path }}'"
    environment_vars = ["ENV=${var.env}"]
  }

  # Alternative: Ansible
  # provisioner "ansible" { playbook_file = "ansible/base.yml"  user = "ec2-user" }

  post-processor "manifest" {
    output     = "manifest.json"
    strip_path = true
    custom_data = { git_commit = env("GIT_COMMIT") }
  }
}
```

Commands:

```bash
packer init .                              # install plugins
packer fmt .  && packer validate -var-file=shared.pkrvars.hcl .
packer build -var-file=shared.pkrvars.hcl -var "subnet_id=subnet-abc" .
PACKER_LOG=1 packer build ...              # debug
packer build -debug ...                    # step-by-step, keeps key
packer build -on-error=ask ...             # inspect failed instance
jq -r '.builds[-1].artifact_id' manifest.json   # "ap-south-1:ami-0abc,ap-south-2:ami-0def"
```

`cleanup.sh` essentials: remove `/home/ec2-user/.ssh/authorized_keys`, `/var/log/*` contents, `cloud-init clean`, shell history, temporary packer files. Otherwise every instance carries build-time secrets/logs.

## 4. Legacy JSON template (recognise it if shown)

```json
{
  "variables": { "region": "ap-south-1" },
  "builders": [{ "type": "amazon-ebs", "region": "{{user `region`}}", "source_ami_filter": {...}, "instance_type": "t3.small", "ssh_username": "ec2-user", "ami_name": "base-{{timestamp}}" }],
  "provisioners": [{ "type": "shell", "script": "setup.sh" }],
  "post-processors": [{ "type": "manifest" }]
}
```
`packer hcl2_upgrade template.json` converts it. HCL2 is the standard since 1.7.

## 5. Golden AMI pipeline (how to describe it)

```
Git push (template + scripts)
  → CI job: packer validate
  → packer build in "shared-services" account, private subnet, SSM
  → Inspector / Trivy / OpenSCAP scan of the new AMI (launch, scan, terminate)
  → if pass: tag Approved=true, write AMI ID to SSM Parameter Store /ami/al2023/base/latest
  → notify (SNS) + optional PR to Terraform repos
  → Terraform data "aws_ssm_parameter" or data "aws_ami" picks it up
  → ASG instance refresh rolls nodes
Schedule: monthly (patch Tuesday) + on critical CVE. Deregister AMIs older than N versions (keep last 3), delete snapshots.
```

Terraform hand-off, two patterns:

```hcl
# Pattern A: SSM parameter (explicit promotion, recommended)
data "aws_ssm_parameter" "base_ami" { name = "/ami/al2023/base/latest" }
image_id = data.aws_ssm_parameter.base_ami.value

# Pattern B: tag lookup
data "aws_ami" "golden" { most_recent = true  owners = ["self"]  filter { name = "tag:Role" values = ["base"] }  filter { name = "tag:Approved" values = ["true"] } }
```

Pin in prod: store the exact AMI ID in `prod.tfvars` and bump via PR, so a new image never rolls to prod implicitly.

## 6. Packer vs alternatives (expect this)

| | Packer | EC2 Image Builder | Config mgmt at boot (Ansible/user data) |
|---|---|---|---|
| Model | Immutable image, code in git | Managed AWS service, recipes/components, scheduled pipelines, built-in tests | Mutable, configures at launch |
| Multi-cloud | Yes | AWS only | Yes |
| Pros | Portable, scriptable, fits CI | No infra to run, native sharing/distribution, AWS-managed hardening components (CIS) | Simple, no image lifecycle |
| Cons | You run the pipeline + scanning | AWS lock-in, less flexible | Slow boot, drift, needs internet/repos at launch |

Reasonable answer: Packer in CI for portability, or Image Builder if the team wants managed. Often both: Packer template, Image Builder-style promotion.

## 7. Troubleshooting

- **Timeout waiting for SSH/SSM**: wrong subnet (no route), SG blocks 22, instance profile missing SSM policy, SSM agent not in base AMI, `ssh_username` wrong (`ec2-user` AL/RHEL, `ubuntu` Ubuntu, `admin` Debian).
- **`AMI name already exists`**: name must be unique → include timestamp.
- **Provisioner script fails**: run with `-on-error=ask`, SSM into the instance; check `sudo` and `set -euo pipefail`.
- **Encrypted AMI can't be shared**: default EBS KMS key can't be shared cross-account; use a CMK with a key policy granting the target accounts, and `kms_key_id` in source.
- **Build succeeds, instances slow to boot**: cleanup missing; cloud-init re-running heavy steps; move install steps into the image.
- **Permissions**: builder role needs `ec2:RunInstances, CreateImage, CreateSnapshot, CreateTags, Describe*, TerminateInstances, Create/DeleteSecurityGroup, Create/DeleteKeyPair, ModifyImageAttribute, CopyImage`, `iam:PassRole` for the instance profile, `kms:*` for the CMK.

## 8. One-hour hands-on before the interview

1. `brew install packer` (or `brew tap hashicorp/tap && brew install hashicorp/tap/packer`).
2. Copy the template above into `~/Desktop/Upskill/Interview/packer-lab/`, drop `scripts/cleanup.sh` with the cleanup commands, set `ssh_interface = "ssh"` and a public subnet if you don't want to set up SSM for the lab.
3. `packer init . && packer validate . && packer build .` in your sandbox account.
4. Reference the AMI from a 10-line Terraform `aws_instance` using `data "aws_ami"` with `owners = ["self"]`.
5. Deregister the AMI and delete the snapshot afterwards (`aws ec2 deregister-image`, `aws ec2 delete-snapshot`).

Then you can say in the interview "I built one last week; here's what broke" instead of "I know the concept".
