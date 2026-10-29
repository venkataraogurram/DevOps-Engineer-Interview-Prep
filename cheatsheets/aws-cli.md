# AWS CLI cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: command → what it does and when you reach for it.

## Identity, profiles, output

- `aws configure` / `aws configure --profile dev`

  Interactive setup of keys/region for the default or a named profile. Prefer SSO/roles over long-lived keys.

- `aws configure list`

  Which credentials and region are active and where they came from (env, profile, instance role). First check when "wrong account".

- `aws configure sso` / `aws sso login --profile x` / `aws login`

  Set up and refresh IAM Identity Center sessions. No static keys.

- `export AWS_PROFILE=prod AWS_REGION=ap-south-1 AWS_PAGER=""`

  Select profile/region for the shell and disable the pager so output streams.

- `aws sts get-caller-identity`

  Account, ARN and user ID you're acting as. Run before anything destructive.

- `aws sts assume-role --role-arn arn:aws:iam::123:role/Admin --role-session-name me --query Credentials`

  Get temporary credentials for a role (cross-account access).

- `aws sts assume-role ... | jq -r '.Credentials | "export AWS_ACCESS_KEY_ID=\(.AccessKeyId) AWS_SECRET_ACCESS_KEY=\(.SecretAccessKey) AWS_SESSION_TOKEN=\(.SessionToken)"'`

  Turn the assume-role output into export statements (`eval "$(...)"`). For scripts without profile support.

- `~/.aws/config` → `[profile prod] role_arn=... source_profile=base mfa_serial=... region=ap-south-1`

  Profile that auto-assumes a role (with MFA) from base credentials. The CLI handles the STS call.

- `aws --debug ...`

  Full request/response logging. For signature or endpoint problems.

- `aws --output table|json|text|yaml`

  Output format; `text` for shell pipelines, `table` for humans.

- `aws --query 'JMESPath'`

  Server-side-ish filtering/reshaping of JSON output. Learn this instead of piping everything to jq.

- `aws --cli-read-timeout 0` / `--no-cli-pager`

  Disable read timeout for long operations / disable paging for one command.

### JMESPath patterns

- `'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]'`

  Flatten nested lists and pick columns.

- `'Reservations[].Instances[].Tags[?Key==`Name`].Value|[0]'`

  Pull one tag value (backticks for literals).

- `'Reservations[].Instances[?State.Name==`running`].InstanceId'`

  Filter by a field.

- `'sort_by(Images,&CreationDate)[-1].ImageId'`

  Newest item by a field.

- `'length(Items)'`

  Count.

- `'Volumes[?State==`available`].{id:VolumeId,size:Size}'`

  Rename/reshape fields into objects.

- `'Functions[?starts_with(FunctionName,`prod-`)].FunctionName'`

  String functions.

- `'Stacks[0].Outputs[?OutputKey==`AlbDns`].OutputValue|[0]'`

  One CloudFormation output as a scalar.

- `'contains(Tags[].Key,`Owner`)'`

  Boolean test.

## EC2, Auto Scaling, SSM

- `aws ec2 describe-instances --filters Name=tag:Env,Values=prod Name=instance-state-name,Values=running --query '...' --output table`

  Inventory by tag and state.

- `aws ec2 run-instances --image-id ami-x --instance-type t3.micro --subnet-id subnet-x --security-group-ids sg-x --iam-instance-profile Name=app --metadata-options HttpTokens=required --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=test}]' --user-data file://ud.sh`

  Launch with IMDSv2 enforced, a role, tags and bootstrap script. The secure one-liner for a test box (Terraform for anything permanent).

- `aws ec2 start-instances|stop-instances|reboot-instances|terminate-instances --instance-ids i-x`

  Lifecycle. Terminate is irreversible unless termination protection is on.

- `aws ec2 describe-instance-status --instance-ids i-x --include-all-instances`

  System and instance status checks (hardware vs OS).

- `aws ec2 get-console-output --instance-id i-x --latest`

  Boot console: kernel panics, cloud-init errors when you can't SSH/SSM.

- `aws ec2 describe-security-groups --group-ids sg-x --query 'SecurityGroups[].IpPermissions'`

  Inbound rules of a SG.

- `aws ec2 authorize-security-group-ingress --group-id sg-x --protocol tcp --port 443 --cidr 10.0.0.0/8` / `revoke-security-group-ingress`

  Add/remove a rule.

- `aws ec2 describe-images --owners self --query 'sort_by(Images,&CreationDate)[-3:].[ImageId,Name,CreationDate]'`

  Your newest three AMIs.

- `aws ec2 describe-images --owners amazon --filters Name=name,Values='al2023-ami-2023*-x86_64' --query 'sort_by(Images,&CreationDate)[-1].ImageId'`

  Latest Amazon Linux 2023 AMI by name pattern.

- `aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text`

  Same via the public SSM parameter (cleaner).

- `aws ec2 create-image --instance-id i-x --name "app-$(date +%F)" --no-reboot`

  Snapshot an instance into an AMI (manual; Packer for repeatable).

- `aws ec2 deregister-image --image-id ami-x` / `aws ec2 delete-snapshot --snapshot-id snap-x`

  Retire an AMI and its snapshot (both steps needed).

- `aws ec2 describe-volumes --filters Name=status,Values=available`

  Unattached volumes = wasted money.

- `aws ec2 modify-volume --volume-id vol-x --size 100 --volume-type gp3`

  Grow/convert online; then `growpart` + `xfs_growfs` on the host.

- `aws ec2 create-snapshot --volume-id vol-x`

  Backup before risky changes.

- `aws ec2 describe-vpcs` / `describe-subnets --filters Name=vpc-id,Values=vpc-x` / `describe-route-tables` / `describe-nat-gateways` / `describe-vpc-endpoints`

  Network inventory for connectivity debugging.

- `aws ec2 describe-network-interfaces --filters Name=addresses.private-ip-address,Values=10.0.1.5`

  What owns an IP (ENI → instance/LB/Lambda/endpoint).

- `aws ec2 describe-instance-types --filters Name=vcpu-info.default-vcpus,Values=4 --query 'InstanceTypes[].InstanceType'`

  Find instance types by spec.

- `aws ec2 describe-spot-price-history --instance-types m6i.large --start-time $(date -u +%FT%TZ) --max-items 5`

  Current Spot prices.

- `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names app --query 'AutoScalingGroups[].{min:MinSize,max:MaxSize,desired:DesiredCapacity,inst:Instances[].[InstanceId,HealthStatus,LifecycleState]}'`

  ASG sizing and instance health.

- `aws autoscaling set-desired-capacity --auto-scaling-group-name app --desired-capacity 4`

  Manual scale.

- `aws autoscaling start-instance-refresh --auto-scaling-group-name app --preferences MinHealthyPercentage=90`

  Roll all instances onto the latest launch template version (new AMI) keeping 90% healthy. `describe-instance-refreshes` to watch.

- `aws autoscaling describe-scaling-activities --auto-scaling-group-name app --max-items 5`

  Why instances launched/terminated (capacity errors show here).

- `aws autoscaling terminate-instance-in-auto-scaling-group --instance-id i-x --no-should-decrement-desired-capacity`

  Replace one bad instance (ASG launches a new one).

- `aws ssm start-session --target i-x`

  Shell without SSH.

- `aws ssm send-command --document-name AWS-RunShellScript --targets Key=tag:Role,Values=web --parameters commands='["uptime"]' --query Command.CommandId`

  Run a command on every instance with a tag; `list-command-invocations --command-id ID --details` for output.

- `aws ssm describe-instance-information --query 'InstanceInformationList[].[InstanceId,PingStatus,AgentVersion]'`

  Which instances SSM can manage. Missing = agent/role/endpoint problem.

- `aws ssm get-parameter --name /app/db --with-decryption --query Parameter.Value --output text`

  Read a SecureString parameter.

- `aws ssm put-parameter --name /app/x --value y --type SecureString --overwrite`

  Write/update.

- `aws ssm get-parameters-by-path --path /app --recursive`

  All parameters under a prefix.

- `aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text | jq`

  Read a secret (JSON). `list-secrets` to find names.

## Load balancers

- `aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerName,DNSName,State.Code]'`

  ALBs/NLBs with DNS names.

- `aws elbv2 describe-target-groups --load-balancer-arn ARN`

  Target groups behind an LB.

- `aws elbv2 describe-target-health --target-group-arn ARN`

  Health per target with a reason: `Target.FailedHealthChecks`, `Target.Timeout`, `Target.ResponseCodeMismatch`, `Target.NotRegistered`. The 502/503 debugging command.

- `aws elbv2 describe-listeners --load-balancer-arn ARN` / `describe-rules --listener-arn ARN`

  Ports/certs and routing rules. "Opened 443 in SG but HTTPS fails" = no 443 listener.

- `aws elbv2 modify-target-group-attributes --target-group-arn ARN --attributes Key=deregistration_delay.timeout_seconds,Value=30`

  Shorten drain time for faster deploys.

- `aws elbv2 modify-load-balancer-attributes --load-balancer-arn ARN --attributes Key=idle_timeout.timeout_seconds,Value=120`

  Raise idle timeout for slow backends (fixes some 504s).

## S3

- `aws s3 ls` / `aws s3 ls s3://b/prefix/ --recursive --human-readable --summarize`

  Buckets / objects with total size and count.

- `aws s3 cp f s3://b/k` / `aws s3 cp s3://b/k - | head`

  Upload / stream an object to stdout.

- `aws s3 sync ./dir s3://b/prefix --delete --exclude '*.tmp'`

  Mirror a directory (only changed files; `--delete` removes extras). Static sites, backups. `--dryrun` first.

- `aws s3 mv` / `aws s3 rm s3://b/k` / `aws s3 rm s3://b/prefix --recursive`

  Move / delete / delete a prefix.

- `aws s3 presign s3://b/k --expires-in 3600`

  Temporary download URL without granting IAM access.

- `aws s3api head-object --bucket b --key k`

  Metadata: size, encryption, storage class, version.

- `aws s3api list-object-versions --bucket b --prefix k`

  Versions and delete markers (recover deleted objects).

- `aws s3api get-bucket-policy --bucket b | jq -r .Policy | jq`

  Readable bucket policy.

- `aws s3api get-bucket-encryption` / `get-public-access-block` / `get-bucket-versioning`

  Security posture checks.

- `aws s3api put-bucket-lifecycle-configuration --bucket b --lifecycle-configuration file://lc.json`

  Transition/expire rules (cost control).

- `aws s3api list-objects-v2 --bucket b --query 'sum(Contents[].Size)'`

  Total bytes under a bucket (paginates; use CloudWatch `BucketSizeBytes` for big buckets).

- `aws s3api put-object-lock-configuration`

  WORM retention for audit logs.

## IAM

- `aws iam get-role --role-name X --query Role.AssumeRolePolicyDocument`

  Trust policy: who can assume the role. First check for `AssumeRole` AccessDenied.

- `aws iam list-attached-role-policies --role-name X` / `list-role-policies` / `get-role-policy --policy-name Y`

  Managed policies / inline policy names / inline policy body.

- `aws iam get-policy --policy-arn ARN` / `get-policy-version --version-id v3` / `list-policy-versions`

  Managed policy metadata / its JSON / history.

- `aws iam create-role --role-name X --assume-role-policy-document file://trust.json`

  Create a role with a trust policy.

- `aws iam attach-role-policy --role-name X --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess`

  Attach a managed policy.

- `aws iam put-role-policy --role-name X --policy-name inline --policy-document file://p.json`

  Add an inline policy.

- `aws iam create-instance-profile --instance-profile-name X && aws iam add-role-to-instance-profile --instance-profile-name X --role-name X`

  EC2 needs an instance profile wrapping the role.

- `aws iam simulate-principal-policy --policy-source-arn ARN --action-names s3:GetObject --resource-arns arn:aws:s3:::b/k`

  Would this principal be allowed? Tests policies without making the call.

- `aws iam list-access-keys --user-name u` / `get-access-key-last-used --access-key-id AKIA...`

  Keys and when last used (find unused/leaked).

- `aws iam update-access-key --access-key-id AKIA... --status Inactive --user-name u`

  Disable a key immediately (incident response; keep for audit, delete later).

- `aws iam generate-credential-report && aws iam get-credential-report --query Content --output text | base64 -d`

  CSV of all users' key age, MFA, last use. Quarterly audit.

- `aws iam get-account-authorization-details > iam-dump.json`

  Everything IAM in one file for offline analysis.

- `aws accessanalyzer list-findings`

  External/unused access findings.

- `aws iam list-open-id-connect-providers` / `create-open-id-connect-provider --url https://token.actions.githubusercontent.com --client-id-list sts.amazonaws.com --thumbprint-list ...`

  OIDC providers for CI federation (GitHub Actions).

## CloudWatch, Logs, CloudTrail

- `aws logs describe-log-groups --query 'logGroups[].[logGroupName,retentionInDays,storedBytes]' --output table`

  Groups with retention and size: find "never expire" groups costing money.

- `aws logs put-retention-policy --log-group-name /aws/lambda/x --retention-in-days 30`

  Set retention.

- `aws logs tail /aws/eks/prod/cluster --follow --since 10m --filter-pattern ERROR`

  Live tail with a filter.

- `aws logs filter-log-events --log-group-name g --start-time $(($(date +%s)-3600))000 --filter-pattern '"AccessDenied"'`

  Search the last hour (epoch ms).

- `aws logs start-query --log-group-name g --start-time ... --end-time ... --query-string '...'` then `get-query-results --query-id ID`

  Logs Insights from the CLI (async).

- `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-x --start-time $(date -u -v-1H +%FT%TZ) --end-time $(date -u +%FT%TZ) --period 300 --statistics Average`

  Pull a metric series (macOS date syntax shown; GNU: `-d '1 hour ago'`).

- `aws cloudwatch describe-alarms --state-value ALARM --query 'MetricAlarms[].[AlarmName,StateReason]'`

  What's firing right now.

- `aws cloudwatch set-alarm-state --alarm-name x --state-value OK --state-reason test`

  Force a state to test notifications.

- `aws cloudwatch put-metric-data --namespace Custom --metric-name X --value 1 --dimensions Host=a`

  Publish a custom metric from a script.

- `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances --start-time 2026-10-01`

  Who called an API (last 90 days of management events).

- `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-x`

  Every change to a specific resource.

## EKS, ECR, ECS, Lambda

- `aws eks list-clusters` / `describe-cluster --name x` / `update-kubeconfig --name x`

  Clusters / details / configure kubectl.

- `aws eks list-nodegroups --cluster-name x` / `describe-addon --cluster-name x --addon-name vpc-cni`

  Node groups / add-on version and health.

- `aws ecr describe-repositories` / `describe-images --repository-name r --query 'sort_by(imageDetails,&imagePushedAt)[-1]'`

  Repos / newest image.

- `aws ecr get-login-password | docker login --username AWS --password-stdin ACC.dkr.ecr.REG.amazonaws.com`

  Docker auth to ECR.

- `aws ecr describe-image-scan-findings --repository-name r --image-id imageTag=t`

  Vulnerability findings for an image.

- `aws ecs list-clusters` / `list-services --cluster c`

  Inventory.

- `aws ecs describe-services --cluster c --services s --query 'services[].{d:desiredCount,r:runningCount,ev:events[0:3].message}'`

  Desired vs running and the last service events (placement failures, health check failures).

- `aws ecs update-service --cluster c --service s --force-new-deployment`

  Redeploy the same task definition (pick up a new `latest` image or restart).

- `aws ecs execute-command --cluster c --task t --container app --interactive --command sh`

  Shell into a running Fargate/ECS container (ECS Exec).

- `aws ecs describe-tasks --cluster c --tasks t --query 'tasks[].{stop:stoppedReason,c:containers[].reason}'`

  Why a task stopped (OOM, essential container exited, image pull).

- `aws lambda list-functions --query 'Functions[].[FunctionName,Runtime,MemorySize,Timeout]'`

  Inventory with runtime (find deprecated runtimes).

- `aws lambda invoke --function-name f --payload '{"a":1}' --cli-binary-format raw-in-base64-out out.json && cat out.json`

  Test invoke; the binary-format flag is needed on CLI v2 for raw JSON.

- `aws lambda update-function-code --function-name f --zip-file fileb://f.zip`

  Deploy code.

- `aws lambda update-function-configuration --function-name f --environment Variables={X=1}`

  Change env/timeout/memory.

- `aws lambda get-function-configuration --function-name f`

  Current config.

- `aws lambda put-function-concurrency --function-name f --reserved-concurrent-executions 50`

  Cap concurrency to protect a downstream.

- `aws lambda list-event-source-mappings --function-name f`

  SQS/Kinesis/DynamoDB triggers and their state (disabled mappings = silent queue backlog).

## CloudFormation

- `aws cloudformation deploy --stack-name s --template-file t.yaml --parameter-overrides K=V --capabilities CAPABILITY_NAMED_IAM --no-fail-on-empty-changeset`

  Create-or-update in one idempotent command (uses a change set). Pipeline-friendly.

- `aws cloudformation describe-stacks --stack-name s --query 'Stacks[0].[StackStatus,Outputs]'`

  Status and outputs.

- `aws cloudformation describe-stack-events --stack-name s --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`||ResourceStatus==`UPDATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' --output table`

  The failing resource and the real error. Faster than scrolling the console.

- `aws cloudformation create-change-set` / `describe-change-set` / `execute-change-set`

  Preview changes before applying (the `plan` equivalent).

- `aws cloudformation detect-stack-drift --stack-name s` / `describe-stack-resource-drifts`

  Find resources changed outside CFN.

- `aws cloudformation continue-update-rollback --stack-name s`

  Recover a stack stuck in `UPDATE_ROLLBACK_FAILED`.

- `aws cloudformation delete-stack --stack-name s`

  Delete (retains resources with `DeletionPolicy: Retain`).

- `aws cloudformation validate-template --template-body file://t.yaml` / `cfn-lint t.yaml`

  Syntax check / deeper lint.

## RDS, DynamoDB, Route 53, ACM, KMS

- `aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceStatus,Engine,MultiAZ,Endpoint.Address]' --output table`

  Inventory with HA flag and endpoint. `describe-db-clusters` for Aurora.

- `aws rds create-db-snapshot --db-instance-identifier x --db-snapshot-identifier x-$(date +%F)`

  Manual snapshot before a migration.

- `aws rds describe-events --source-identifier x --duration 1440`

  Last 24 h of events (failovers, maintenance).

- `aws rds reboot-db-instance --db-instance-identifier x --force-failover`

  Test Multi-AZ failover.

- `aws rds modify-db-instance --db-instance-identifier x --apply-immediately --allocated-storage 200`

  Change settings now instead of in the maintenance window.

- `aws dynamodb describe-table --table-name t`

  Capacity mode, indexes, item count.

- `aws dynamodb scan --table-name terraform-locks`

  See Terraform lock items.

- `aws dynamodb delete-item --table-name terraform-locks --key '{"LockID":{"S":"bucket/key"}}'`

  Remove a stuck Terraform lock (alternative to `force-unlock`).

- `aws route53 list-hosted-zones` / `list-resource-record-sets --hosted-zone-id Z --query "ResourceRecordSets[?Name=='app.example.com.']"`

  Zones / one record (note trailing dot).

- `aws route53 change-resource-record-sets --hosted-zone-id Z --change-batch file://c.json`

  Create/update records (UPSERT).

- `aws route53 test-dns-answer --hosted-zone-id Z --record-name app.example.com --record-type A`

  What Route 53 would answer, bypassing caches.

- `aws acm list-certificates --query 'CertificateSummaryList[].[DomainName,Status,NotAfter]'`

  Certs with expiry; `PENDING_VALIDATION` means DNS/email validation incomplete.

- `aws acm describe-certificate --certificate-arn ARN --query 'Certificate.DomainValidationOptions'`

  The CNAME you must create to validate.

- `aws kms list-aliases` / `describe-key --key-id alias/x`

  Keys and their state/rotation.

- `aws kms get-key-policy --key-id K --policy-name default --query Policy --output text | jq`

  The key policy: who can use the key (root of trust for KMS AccessDenied).

- `aws kms encrypt --key-id alias/x --plaintext fileb://f --query CiphertextBlob --output text`

  Encrypt a small blob (testing key access).

## Organizations, Config, Cost, Quotas, Health

- `aws organizations list-accounts --query 'Accounts[].[Id,Name,Status]' --output table`

  All accounts in the org (from the management/delegated account).

- `aws organizations list-policies --filter SERVICE_CONTROL_POLICY` / `describe-policy --policy-id p-x`

  SCPs and their content. Check when an admin gets AccessDenied.

- `aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT`

  Rules currently failing.

- `aws configservice get-resource-config-history --resource-type AWS::EC2::SecurityGroup --resource-id sg-x`

  Configuration timeline of a resource (what changed, when).

- `aws ce get-cost-and-usage --time-period Start=2026-10-01,End=2026-10-28 --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE --query 'ResultsByTime[].Groups[].[Keys[0],Metrics.UnblendedCost.Amount]' --output table`

  Spend by service for the month. Swap `Key=SERVICE` for `Type=TAG,Key=Team` with tags.

- `aws service-quotas list-service-quotas --service-code ec2 --query 'Quotas[?contains(QuotaName,`Running On-Demand Standard`)]'`

  Current vCPU quota.

- `aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value 256`

  Request an increase from the CLI.

- `aws health describe-events --filter eventStatusCodes=open,upcoming --region us-east-1`

  Open AWS Health events (needs Business/Enterprise support; Health API lives in us-east-1).

- `aws support describe-cases --include-resolved-cases false`

  Open support cases.

- `aws resourcegroupstaggingapi get-resources --tag-filters Key=Env,Values=prod --query 'ResourceTagMappingList[].ResourceARN'`

  Every resource with a tag, across services. Audit and cleanup.

## Shell helpers

```bash
alias awsw='aws sts get-caller-identity --query "[Account,Arn]" --output text'   # "where am I" in one word
ec2ip() { aws ec2 describe-instances --filters "Name=tag:Name,Values=$1" --query 'Reservations[].Instances[].PrivateIpAddress' --output text; }   # IP by Name tag
ssm() { aws ssm start-session --target "$(aws ec2 describe-instances --filters "Name=tag:Name,Values=$1" Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].InstanceId' --output text)"; }   # shell by Name tag
```
