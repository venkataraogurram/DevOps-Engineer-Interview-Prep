# Packer cheatsheet (annotated)

[← Cheatsheet index](README.md) · Deep dive and full template: [../04-refresher-packer.md](../04-refresher-packer.md)

Format: command → what it does and when you reach for it.

## CLI

| Command | What it does / when to use |
|---|---|
| `packer init .` | Installs the plugins declared in `required_plugins` (e.g. the Amazon plugin). Run first in a fresh checkout, like `terraform init`. |
| `packer fmt -recursive .` | Canonical formatting for `.pkr.hcl` files. Pre-commit. |
| `packer validate -var-file=vars.pkrvars.hcl .` | Checks syntax and that all variables/sources resolve, without launching anything. CI gate. `-syntax-only` skips variable evaluation. |
| `packer build .` | Runs every `build` block: launches the builder instance, provisions, creates the AMI, cleans up. |
| `packer build -var 'subnet_id=subnet-x' -var-file=prod.pkrvars.hcl .` | Supply variables inline or from a file (same precedence idea as Terraform). |
| `packer build -only='amazon-ebs.base' .` / `-except='manifest'` | Run only one source, or skip a post-processor. Useful when a template builds several images. |
| `packer build -debug .` | Pauses between steps and saves the temporary SSH key so you can log into the builder. For "provisioner fails and I can't see why". |
| `packer build -on-error=ask .` | On failure, asks whether to abort, retry, or leave the instance running for inspection (`abort`, `cleanup`, `run-cleanup-provisioner` are the other values). Use in interactive debugging; default `cleanup` for CI. |
| `packer build -force .` | Deregister an existing AMI with the same name first. For local iteration; in CI use timestamped names instead. |
| `packer build -parallel-builds=2 .` | Limit concurrent builds. |
| `packer build -timestamp-ui -color=false . \| tee build.log` | Timestamped, colourless output saved to a file. CI logging. |
| `PACKER_LOG=1 PACKER_LOG_PATH=packer.log packer build .` | Full debug log (API calls, SSH/SSM handshakes). For connection timeouts. |
| `packer inspect .` | Lists variables, sources, provisioners found in the template. Quick orientation. |
| `packer console .` | Evaluate HCL expressions interactively (`formatdate(...)`, `env("X")`). |
| `packer hcl2_upgrade template.json` | Convert a legacy JSON template to HCL2. |
| `packer plugins installed` / `packer plugins install github.com/hashicorp/amazon` | Show / install plugins manually. |
| `packer version` | CLI version (≥1.7 for HCL2 defaults). |

## Template anatomy

| Block | Purpose |
|---|---|
| `packer { required_plugins { amazon = { source = "github.com/hashicorp/amazon", version = ">= 1.3.0" } } }` | Pin the builder plugin so CI and laptops match. |
| `variable "x" { type = string  default = "" }` | Inputs; set via `-var`, `-var-file`, or `PKR_VAR_x` env vars. |
| `locals { ami_name = "base-${formatdate("YYYYMMDD-hhmm", timestamp())}" }` | Computed values; the timestamped name avoids "AMI name already exists". |
| `data "amazon-ami" "al2023" { filters = { name = "al2023-ami-2023*-x86_64" }  owners = ["amazon"]  most_recent = true }` | Look up the newest base AMI instead of hardcoding an ID. |
| `data "amazon-parameterstore" "x" { name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64" }` | Same via the public SSM parameter. |
| `source "amazon-ebs" "base" { ... }` | How to launch the builder and what AMI to make. `amazon-ebssurrogate` for custom root volumes; `docker`, `azure-arm`, `googlecompute` for other targets. |
| `build { sources = [...]  provisioner ... post-processor ... }` | Ties a source to the steps that run on it and what happens after. |

## Key `amazon-ebs` arguments

| Argument | What it does / why it matters |
|---|---|
| `region`, `source_ami`, `instance_type` | Where to build, from what, on what size (t3.small is enough; bigger speeds up compile-heavy provisioning). |
| `ami_name`, `ami_description` | Output name must be unique per region. |
| `ami_regions = ["ap-south-1","ap-south-2"]` | Copy the finished AMI to other regions (DR). |
| `ami_users = ["111111111111"]` / `ami_org_arns = [...]` | Share with specific accounts or the whole organisation. |
| `encrypt_boot = true`, `kms_key_id = "alias/ami"`, `region_kms_key_ids = {...}` | Encrypted root volume with a CMK (required for sharing encrypted AMIs across accounts; the default EBS key can't be shared). |
| `vpc_id`, `subnet_id`, `security_group_ids`, `associate_public_ip_address = false` | Build inside a private subnet; no public IP. |
| `iam_instance_profile = "packer-builder"` or `temporary_iam_instance_profile_policy_document` | Role on the builder instance (needs SSM core policy for the SSM communicator). |
| `ssh_username = "ec2-user"`, `ssh_interface = "session_manager"` | Login user for the base OS (`ubuntu` on Ubuntu) and connect over SSM instead of opening port 22. |
| `temporary_key_pair_type = "ed25519"` | Modern throwaway key. |
| `launch_block_device_mappings { device_name = "/dev/xvda" volume_size = 20 volume_type = "gp3" delete_on_termination = true }` | Root disk size/type for the build; `ami_block_device_mappings` for what the AMI advertises. |
| `tags = {...}`, `run_tags = {...}`, `snapshot_tags = {...}` | Tags on the AMI / on the temporary instance (so cost reports and SCPs are happy) / on snapshots. |
| `imds_support = "v2.0"` | Marks the AMI as IMDSv2-only so instances launched from it enforce token-based metadata. |
| `force_deregister`, `force_delete_snapshot` | Overwrite same-named AMI and its snapshots. |
| `skip_create_ami = true` | Run provisioners for testing without producing an AMI. |
| `spot_price = "auto"`, `spot_instance_types` | Build on Spot to cut cost. |

Template strings available in tags: `{{ .SourceAMI }}`, `{{ .SourceAMIName }}`, `{{ .BuildRegion }}`, `{{ build_name }}`, `{{ timestamp }}`, `{{ uuid }}`. HCL functions: `formatdate`, `timestamp()`, `env("GIT_COMMIT")`, `regex_replace`, `vault("secret/x","key")`, `aws_secretsmanager("name","key")`.

## Provisioners

| Setting | What it does / when to use |
|---|---|
| `provisioner "shell" { inline = [...] }` / `scripts = [...]` / `script = "x.sh"` | Run commands or scripts over the connection. Most builds are shell. |
| `execute_command = "chmod +x {{ .Path }}; sudo -E sh -c '{{ .Vars }} {{ .Path }}'"` | Run scripts as root while passing environment variables through. Needed because the login user isn't root. |
| `environment_vars = ["ENV=${var.env}"]` | Pass values to scripts. |
| `expect_disconnect = true` + `pause_before = "30s"` | Survive a reboot inside a provisioner (kernel updates). |
| `valid_exit_codes = [0, 2]`, `max_retries = 3` | Tolerate specific exit codes / retry flaky steps (package mirrors). |
| `provisioner "file" { source = "files/x.json" destination = "/tmp/x.json" }` | Upload files; destination must be writable by the login user, then `mv` with sudo in a shell step. |
| `provisioner "ansible" { playbook_file = "ansible/base.yml"  user = "ec2-user" }` | Run Ansible from your machine against the builder. `ansible-local` runs it on the instance. |
| `provisioner "powershell"` / `"windows-restart"` | Windows images. |
| `provisioner "breakpoint" { disable = false note = "inspect" }` | Pause the build so you can log in and look around. |
| `post-processor "manifest" { output = "manifest.json" strip_path = true }` | Write the AMI IDs per region to JSON for the pipeline to consume. |
| `post-processor "shell-local"` | Run a local script after the build (e.g. trigger a scan). |

Cleanup script (last provisioner) must remove: `authorized_keys` for all users, `cloud-init clean --logs` (so cloud-init runs fresh on launch), `/var/log/*` and `/tmp/*` contents, shell history, `/etc/machine-id` (empty file, regenerated at boot), any temporary packer user, then `sync`.

## After the build

| Command | What it does / when to use |
|---|---|
| `jq -r '.builds[-1].artifact_id \| split(":")[1]' manifest.json` | Extract the AMI ID of the last build (single region). |
| `jq -r '.builds[-1].artifact_id \| split(",")[] \| split(":") \| {(.[0]): .[1]}' manifest.json` | Region → AMI map when `ami_regions` copied it. |
| `aws ssm put-parameter --name /ami/al2023/base/latest --type String --value ami-x --overwrite --description "git=$GIT_COMMIT"` | Promote: publish the approved AMI ID where Terraform reads it (`data "aws_ssm_parameter"`). |
| `aws ec2 describe-images --owners self --filters Name=tag:Role,Values=base --query 'sort_by(Images,&CreationDate)[:-3].[ImageId,BlockDeviceMappings[].Ebs.SnapshotId]'` | List all but the newest three AMIs of a role, with their snapshots, for retirement. |
| `aws ec2 deregister-image --image-id ami-x` then `aws ec2 delete-snapshot --snapshot-id snap-x` | Retire an AMI; snapshots aren't deleted automatically and keep costing. |
| `aws inspector2 ...` / launch-scan-terminate | Scan the AMI before tagging `Approved=true`. |

## Builder IAM (minimum)

EC2: `AttachVolume AuthorizeSecurityGroupIngress CopyImage CreateImage CreateKeyPair CreateSecurityGroup CreateSnapshot CreateTags CreateVolume DeleteKeyPair DeleteSecurityGroup DeleteSnapshot DeleteVolume DeregisterImage Describe* DetachVolume GetPasswordData ModifyImageAttribute ModifyInstanceAttribute ModifySnapshotAttribute RegisterImage RunInstances StopInstances TerminateInstances` · IAM: `PassRole` + `GetInstanceProfile` for the builder instance profile · KMS: encrypt/decrypt/grant on the CMK · SSM: `StartSession`, `TerminateSession` when using the SSM communicator.

## Failure → cause

| Error | Cause → fix |
|---|---|
| `Timeout waiting for SSH/SSM` | No route from your runner (SSH) or missing SSM agent/instance profile (SSM); wrong `ssh_username`; SG blocks 22. |
| `AMI Name: '...' is used by an existing AMI` | Static name → add a timestamp or `-force`. |
| `Error launching source instance: ... PassRole` | Builder role may not pass the instance profile → add `iam:PassRole`. |
| Shared AMI launches fail in another account | Encrypted with the default EBS key → use a CMK whose key policy grants the target account. |
| Provisioner script exits non-zero | Missing `sudo`, package mirror flake, `set -e` on a harmless failure → `-on-error=ask`, inspect, add retries. |
| Instances from the AMI boot slowly / rerun installs | Cleanup missing (`cloud-init clean`) or install steps left in user data → move them into the image. |
