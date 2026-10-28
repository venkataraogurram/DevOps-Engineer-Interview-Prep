# 4. Packer

[← Back to index](README.md)

**Q35. What is Packer and why would a bank use it?**
Packer builds machine images from code: it launches a temporary instance from a base AMI, runs provisioners (shell, Ansible), creates an AMI, and terminates the builder. Banks use it to produce golden AMIs with CIS hardening, patches, monitoring and security agents already installed and scanned, so every instance starts compliant, boots fast, and there's no configuration drift. Packer builds; Terraform deploys the AMI.

---

**Q36. Walk me through a Packer HCL template.**
`packer { required_plugins }` for the Amazon plugin; variables; a `data "amazon-ami"` block to find the latest base image; a `source "amazon-ebs"` block with region, source AMI, instance type, VPC and private subnet, SSM as the communicator so no port 22, encrypted boot with a CMK, `ami_regions` to copy to the DR region, `ami_users` or org ARNs to share, and tags like SourceAMI, BuildDate, GitCommit; then a `build` block with shell provisioners for updates, hardening and agent install, a cleanup script that removes keys and logs, and a `manifest` post-processor that writes the AMI IDs to JSON for the pipeline.

---

**Q37. How does the Packer AMI reach Terraform?**
Two patterns. Explicit promotion: the pipeline writes the approved AMI ID to an SSM parameter like `/ami/al2023/base/latest` and Terraform reads `data "aws_ssm_parameter"`. Discovery: Terraform uses `data "aws_ami"` with `most_recent = true`, `owners = ["self"]` and tag filters like `Role=base, Approved=true`. For prod I pin the AMI ID in tfvars and bump it through a PR so a new image never rolls to prod implicitly; the ASG instance refresh then replaces nodes.

---

**Q38. Packer vs EC2 Image Builder?**
Image Builder is the managed AWS equivalent: recipes, components including AWS-managed CIS hardening, scheduled pipelines, built-in tests and distribution. No infrastructure to run, but AWS-only and less flexible. Packer is portable and fits any CI. I'd pick Packer if the team already has CI and wants multi-cloud or Docker images too; Image Builder if they want fully managed and AWS-only.

---

**Q39. Common Packer failures?**
Timeout waiting for SSH or SSM: wrong subnet routing, security group, missing SSM instance profile, or wrong `ssh_username`. "AMI name already exists": include a timestamp. Encrypted AMI can't be shared: the default EBS key isn't shareable; use a CMK with a key policy for the target accounts. Slow boots after a "successful" build: cleanup missing or install steps left in user data instead of the image.

---

[← Back to index](README.md)
