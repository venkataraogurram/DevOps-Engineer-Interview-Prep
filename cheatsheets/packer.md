# Packer cheatsheet

[← Cheatsheet index](README.md) · Deep dive: [../04-refresher-packer.md](../04-refresher-packer.md)

```bash
packer init .                         # install required_plugins
packer fmt -recursive .; packer validate -var-file=vars.pkrvars.hcl .; packer validate -syntax-only .
packer build .; packer build -var 'subnet_id=subnet-x' -var-file=prod.pkrvars.hcl -only='amazon-ebs.base' -except='manifest' .
packer build -debug . ; packer build -on-error=ask|abort|cleanup|run-cleanup-provisioner . ; packer build -force . (overwrite AMI name) ; packer build -parallel-builds=2 .
packer build -timestamp-ui -color=false . | tee build.log
PACKER_LOG=1 PACKER_LOG_PATH=packer.log packer build .
packer inspect .; packer console .; packer hcl2_upgrade template.json; packer plugins installed; packer plugins install github.com/hashicorp/amazon
packer version
```

Template blocks: `packer {}` · `variable {}` · `locals {}` · `data "amazon-ami" {}` / `data "amazon-parameterstore" {}` · `source "amazon-ebs|amazon-ebssurrogate|amazon-instance|docker|azure-arm|googlecompute" {}` · `build { sources provisioner "shell|file|ansible|ansible-local|powershell|windows-restart|breakpoint" post-processor "manifest|amazon-import|docker-tag|docker-push|checksum|shell-local|compress" }`.

Key `amazon-ebs` args: `region source_ami instance_type ami_name ami_description ami_regions ami_users ami_org_arns encrypt_boot kms_key_id region_kms_key_ids vpc_id subnet_id security_group_ids associate_public_ip_address iam_instance_profile ssh_username ssh_interface=session_manager|public_ip|private_ip temporary_key_pair_type temporary_iam_instance_profile_policy_document launch_block_device_mappings{} ami_block_device_mappings{} tags{} run_tags{} snapshot_tags{} force_deregister force_delete_snapshot skip_create_ami imds_support=v2.0 metadata_options{} user_data_file spot_price spot_instance_types`.

Template variables inside tags/strings: `{{ .SourceAMI }} {{ .SourceAMIName }} {{ .BuildRegion }} {{ build_name }} {{ timestamp }} {{ uuid }}` (legacy engine) ; HCL functions: `formatdate("YYYYMMDD-hhmm", timestamp())`, `env("GIT_COMMIT")`, `regex_replace`, `legacy_isotime`, `vault("secret/x","key")`, `aws_secretsmanager("name","key")`, `consul_key`.

Shell provisioner tips: `execute_command = "chmod +x {{ .Path }}; sudo -E sh -c '{{ .Vars }} {{ .Path }}'"`, `environment_vars = ["ENV=${var.env}"]`, `expect_disconnect = true` + `pause_before = "30s"` around reboots, `inline_shebang`, `valid_exit_codes`, `max_retries`.

Cleanup script must: `rm -f /home/*/.ssh/authorized_keys /root/.ssh/authorized_keys`, `cloud-init clean --logs`, `rm -rf /var/log/* /tmp/*`, `history -c`, `rm -f /etc/machine-id && touch /etc/machine-id`, remove packer temp user if created, `sync`.

Manifest → AMI id: `jq -r '.builds[-1].artifact_id | split(":")[1]' manifest.json`; per region: `jq -r '.builds[-1].artifact_id | split(",")[] | split(":") | {(.[0]): .[1]}' manifest.json`.

Promote to SSM: `aws ssm put-parameter --name /ami/al2023/base/latest --type String --value ami-x --overwrite --description "git=$GIT_COMMIT"`.

Lifecycle: `aws ec2 describe-images --owners self --filters Name=tag:Role,Values=base --query 'sort_by(Images,&CreationDate)[:-3].[ImageId,BlockDeviceMappings[].Ebs.SnapshotId]'` → deregister + delete snapshots.

Builder IAM (minimum): `ec2:AttachVolume AuthorizeSecurityGroupIngress CopyImage CreateImage CreateKeyPair CreateSecurityGroup CreateSnapshot CreateTags CreateVolume DeleteKeyPair DeleteSecurityGroup DeleteSnapshot DeleteVolume DeregisterImage Describe* DetachVolume GetPasswordData ModifyImageAttribute ModifyInstanceAttribute ModifySnapshotAttribute RegisterImage RunInstances StopInstances TerminateInstances` + `iam:PassRole` (instance profile) + `iam:GetInstanceProfile` + `kms:*` on CMK + `ssm:StartSession` / `ssm:TerminateSession` for SSM connector.
