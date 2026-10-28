# AWS CLI cheatsheet (DevOps daily)

[← Cheatsheet index](README.md)

## Identity, profiles, config

```bash
aws configure; aws configure --profile dev; aws configure list; aws configure sso; aws sso login --profile x; aws login
export AWS_PROFILE=prod AWS_REGION=ap-south-1 AWS_PAGER=""
aws sts get-caller-identity; aws sts assume-role --role-arn arn:aws:iam::123:role/Admin --role-session-name me --query Credentials
aws sts assume-role ... | jq -r '.Credentials | "export AWS_ACCESS_KEY_ID=\(.AccessKeyId) AWS_SECRET_ACCESS_KEY=\(.SecretAccessKey) AWS_SESSION_TOKEN=\(.SessionToken)"'
# ~/.aws/config: [profile prod] role_arn=... source_profile=base mfa_serial=... region=ap-south-1 ; [profile sso] sso_session=... sso_account_id=... sso_role_name=...
aws --debug ...; aws --cli-read-timeout 0; aws --no-cli-pager; aws --output table|json|text|yaml; aws --query 'JMESPath'
```

## JMESPath `--query` patterns

```bash
'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]'
'Reservations[].Instances[?State.Name==`running`].InstanceId' ; 'sort_by(Images,&CreationDate)[-1].ImageId' ; 'length(Items)'
'Volumes[?State==`available`].{id:VolumeId,size:Size}' ; 'Buckets[].Name' ; 'Functions[?starts_with(FunctionName,`prod-`)].FunctionName'
'Stacks[0].Outputs[?OutputKey==`AlbDns`].OutputValue|[0]' ; 'Parameters[].[Name,Value]' ; '*.Count' ; 'contains(Tags[].Key,`Owner`)'
```

## EC2 / ASG / SSM

```bash
aws ec2 describe-instances --filters Name=tag:Env,Values=prod Name=instance-state-name,Values=running --query '...' --output table
aws ec2 run-instances --image-id ami-x --instance-type t3.micro --subnet-id subnet-x --security-group-ids sg-x --iam-instance-profile Name=app --metadata-options HttpTokens=required --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=test}]' --user-data file://ud.sh
aws ec2 start-instances|stop-instances|reboot-instances|terminate-instances --instance-ids i-x
aws ec2 describe-instance-status --instance-ids i-x --include-all-instances; aws ec2 get-console-output --instance-id i-x --latest
aws ec2 describe-security-groups --group-ids sg-x --query 'SecurityGroups[].IpPermissions'; aws ec2 authorize-security-group-ingress --group-id sg-x --protocol tcp --port 443 --cidr 10.0.0.0/8; aws ec2 revoke-security-group-ingress ...
aws ec2 describe-images --owners self --query 'sort_by(Images,&CreationDate)[-3:].[ImageId,Name,CreationDate]'; aws ec2 describe-images --owners amazon --filters Name=name,Values='al2023-ami-2023*-x86_64' --query 'sort_by(Images,&CreationDate)[-1].ImageId'
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text
aws ec2 create-image --instance-id i-x --name "app-$(date +%F)" --no-reboot; aws ec2 deregister-image --image-id ami-x; aws ec2 delete-snapshot --snapshot-id snap-x
aws ec2 describe-volumes --filters Name=status,Values=available; aws ec2 modify-volume --volume-id vol-x --size 100 --volume-type gp3; aws ec2 create-snapshot --volume-id vol-x
aws ec2 describe-vpcs; describe-subnets --filters Name=vpc-id,Values=vpc-x; describe-route-tables; describe-nat-gateways; describe-vpc-endpoints; describe-network-interfaces --filters Name=addresses.private-ip-address,Values=10.0.1.5
aws ec2 describe-instance-types --filters Name=vcpu-info.default-vcpus,Values=4 --query 'InstanceTypes[].InstanceType'; aws ec2 describe-spot-price-history --instance-types m6i.large --start-time $(date -u +%FT%TZ) --max-items 5
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names app --query 'AutoScalingGroups[].{min:MinSize,max:MaxSize,desired:DesiredCapacity,inst:Instances[].[InstanceId,HealthStatus,LifecycleState]}'
aws autoscaling set-desired-capacity --auto-scaling-group-name app --desired-capacity 4; aws autoscaling start-instance-refresh --auto-scaling-group-name app --preferences MinHealthyPercentage=90; aws autoscaling describe-instance-refreshes --auto-scaling-group-name app
aws autoscaling describe-scaling-activities --auto-scaling-group-name app --max-items 5; aws autoscaling terminate-instance-in-auto-scaling-group --instance-id i-x --no-should-decrement-desired-capacity
aws ssm start-session --target i-x; aws ssm send-command --document-name AWS-RunShellScript --targets Key=tag:Role,Values=web --parameters commands='["uptime"]' --query Command.CommandId; aws ssm list-command-invocations --command-id ID --details
aws ssm describe-instance-information --query 'InstanceInformationList[].[InstanceId,PingStatus,AgentVersion]'
aws ssm get-parameter --name /app/db --with-decryption --query Parameter.Value --output text; aws ssm put-parameter --name /app/x --value y --type SecureString --overwrite; aws ssm get-parameters-by-path --path /app --recursive
aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text | jq; aws secretsmanager list-secrets
```

## Load balancers

```bash
aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerName,DNSName,State.Code]'
aws elbv2 describe-target-groups --load-balancer-arn ARN; aws elbv2 describe-target-health --target-group-arn ARN   # Reason: Target.FailedHealthChecks / Target.Timeout / Target.ResponseCodeMismatch
aws elbv2 describe-listeners --load-balancer-arn ARN; aws elbv2 describe-rules --listener-arn ARN
aws elbv2 modify-target-group-attributes --target-group-arn ARN --attributes Key=deregistration_delay.timeout_seconds,Value=30
aws elbv2 modify-load-balancer-attributes --load-balancer-arn ARN --attributes Key=idle_timeout.timeout_seconds,Value=120
```

## S3

```bash
aws s3 ls; aws s3 ls s3://b/prefix/ --recursive --human-readable --summarize
aws s3 cp f s3://b/k; aws s3 cp s3://b/k - | head; aws s3 sync ./dir s3://b/prefix --delete --exclude '*.tmp'; aws s3 mv; aws s3 rm s3://b/k; aws s3 rm s3://b/prefix --recursive
aws s3 presign s3://b/k --expires-in 3600
aws s3api head-object --bucket b --key k; aws s3api list-object-versions --bucket b --prefix k; aws s3api get-bucket-policy --bucket b | jq -r .Policy | jq; aws s3api get-bucket-encryption; get-public-access-block; get-bucket-versioning; put-bucket-lifecycle-configuration --bucket b --lifecycle-configuration file://lc.json
aws s3api list-objects-v2 --bucket b --query 'sum(Contents[].Size)'
aws s3control ... ; aws s3api put-object-lock-configuration
```

## IAM

```bash
aws iam get-role --role-name X --query Role.AssumeRolePolicyDocument; aws iam list-attached-role-policies --role-name X; aws iam list-role-policies --role-name X; aws iam get-role-policy --role-name X --policy-name Y
aws iam get-policy --policy-arn ARN; aws iam get-policy-version --policy-arn ARN --version-id v3; aws iam list-policy-versions
aws iam create-role --role-name X --assume-role-policy-document file://trust.json; aws iam attach-role-policy --role-name X --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess; aws iam put-role-policy --role-name X --policy-name inline --policy-document file://p.json
aws iam create-instance-profile --instance-profile-name X; aws iam add-role-to-instance-profile --instance-profile-name X --role-name X
aws iam simulate-principal-policy --policy-source-arn ARN --action-names s3:GetObject --resource-arns arn:aws:s3:::b/k
aws iam list-users; aws iam list-access-keys --user-name u; aws iam get-access-key-last-used --access-key-id AKIA...; aws iam update-access-key --status Inactive ...; aws iam generate-credential-report; aws iam get-credential-report --query Content --output text | base64 -d
aws iam get-account-authorization-details > iam-dump.json; aws accessanalyzer list-findings
aws iam list-open-id-connect-providers; aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com --client-id-list sts.amazonaws.com --thumbprint-list ...
```

## CloudWatch / Logs / CloudTrail

```bash
aws logs describe-log-groups --query 'logGroups[].[logGroupName,retentionInDays,storedBytes]' --output table; aws logs put-retention-policy --log-group-name /aws/lambda/x --retention-in-days 30
aws logs tail /aws/eks/prod/cluster --follow --since 10m --filter-pattern ERROR; aws logs filter-log-events --log-group-name g --start-time $(($(date +%s)-3600))000 --filter-pattern '"AccessDenied"'
aws logs start-query --log-group-name g --start-time ... --end-time ... --query-string 'fields @timestamp,@message | filter @message like /ERROR/ | limit 50'; aws logs get-query-results --query-id ID
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-x --start-time $(date -u -v-1H +%FT%TZ) --end-time $(date -u +%FT%TZ) --period 300 --statistics Average
aws cloudwatch describe-alarms --state-value ALARM --query 'MetricAlarms[].[AlarmName,StateReason]'; aws cloudwatch set-alarm-state --alarm-name x --state-value OK --state-reason test
aws cloudwatch put-metric-data --namespace Custom --metric-name X --value 1 --dimensions Host=a
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances --start-time 2026-10-01 --query 'Events[].[EventTime,Username,CloudTrailEvent]' | head
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-x
```

## EKS / ECR / ECS / Lambda

```bash
aws eks list-clusters; aws eks describe-cluster --name x; aws eks update-kubeconfig --name x; aws eks list-nodegroups --cluster-name x; aws eks describe-addon --cluster-name x --addon-name vpc-cni
aws ecr describe-repositories; aws ecr describe-images --repository-name r --query 'sort_by(imageDetails,&imagePushedAt)[-1]'; aws ecr get-login-password | docker login --username AWS --password-stdin ACC.dkr.ecr.REG.amazonaws.com; aws ecr describe-image-scan-findings --repository-name r --image-id imageTag=t
aws ecs list-clusters; aws ecs list-services --cluster c; aws ecs describe-services --cluster c --services s --query 'services[].{d:desiredCount,r:runningCount,ev:events[0:3].message}'; aws ecs update-service --cluster c --service s --force-new-deployment; aws ecs execute-command --cluster c --task t --container app --interactive --command sh; aws ecs describe-tasks --cluster c --tasks t --query 'tasks[].{stop:stoppedReason,c:containers[].reason}'
aws lambda list-functions --query 'Functions[].[FunctionName,Runtime,MemorySize,Timeout]'; aws lambda invoke --function-name f --payload '{"a":1}' --cli-binary-format raw-in-base64-out out.json && cat out.json; aws lambda update-function-code --function-name f --zip-file fileb://f.zip; aws lambda update-function-configuration --function-name f --environment Variables={X=1}; aws lambda get-function-configuration --function-name f; aws lambda put-function-concurrency --function-name f --reserved-concurrent-executions 50; aws lambda list-event-source-mappings --function-name f
```

## CloudFormation

```bash
aws cloudformation deploy --stack-name s --template-file t.yaml --parameter-overrides K=V --capabilities CAPABILITY_NAMED_IAM --no-fail-on-empty-changeset
aws cloudformation describe-stacks --stack-name s --query 'Stacks[0].[StackStatus,Outputs]'; aws cloudformation describe-stack-events --stack-name s --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`||ResourceStatus==`UPDATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' --output table
aws cloudformation create-change-set ... ; describe-change-set; execute-change-set; aws cloudformation detect-stack-drift --stack-name s; describe-stack-resource-drifts
aws cloudformation delete-stack --stack-name s; aws cloudformation continue-update-rollback --stack-name s; aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE
cfn-lint t.yaml; aws cloudformation validate-template --template-body file://t.yaml
```

## RDS / DynamoDB / Route 53 / ACM / KMS

```bash
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceStatus,Engine,MultiAZ,Endpoint.Address]' --output table; aws rds describe-db-clusters
aws rds create-db-snapshot --db-instance-identifier x --db-snapshot-identifier x-$(date +%F); aws rds describe-events --source-identifier x --duration 1440; aws rds reboot-db-instance --db-instance-identifier x --force-failover; aws rds modify-db-instance --db-instance-identifier x --apply-immediately --allocated-storage 200
aws dynamodb describe-table --table-name t; aws dynamodb scan --table-name terraform-locks; aws dynamodb delete-item --table-name terraform-locks --key '{"LockID":{"S":"bucket/key"}}'   # stuck lock
aws route53 list-hosted-zones; aws route53 list-resource-record-sets --hosted-zone-id Z --query "ResourceRecordSets[?Name=='app.example.com.']"; aws route53 change-resource-record-sets --hosted-zone-id Z --change-batch file://c.json; aws route53 test-dns-answer --hosted-zone-id Z --record-name app.example.com --record-type A
aws acm list-certificates --query 'CertificateSummaryList[].[DomainName,Status,NotAfter]'; aws acm describe-certificate --certificate-arn ARN --query 'Certificate.DomainValidationOptions'
aws kms list-aliases; aws kms describe-key --key-id alias/x; aws kms get-key-policy --key-id K --policy-name default --query Policy --output text | jq; aws kms encrypt --key-id alias/x --plaintext fileb://f --query CiphertextBlob --output text
```

## Organizations / Config / Cost / Quotas / Health

```bash
aws organizations list-accounts --query 'Accounts[].[Id,Name,Status]' --output table; aws organizations list-policies --filter SERVICE_CONTROL_POLICY; aws organizations describe-policy --policy-id p-x
aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT; aws configservice get-resource-config-history --resource-type AWS::EC2::SecurityGroup --resource-id sg-x
aws ce get-cost-and-usage --time-period Start=2026-10-01,End=2026-10-28 --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE --query 'ResultsByTime[].Groups[].[Keys[0],Metrics.UnblendedCost.Amount]' --output table
aws service-quotas list-service-quotas --service-code ec2 --query 'Quotas[?contains(QuotaName,`Running On-Demand Standard`)]'; aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value 256
aws health describe-events --filter eventStatusCodes=open,upcoming --region us-east-1
aws support describe-cases --include-resolved-cases false   # Business+ support
aws resourcegroupstaggingapi get-resources --tag-filters Key=Env,Values=prod --query 'ResourceTagMappingList[].ResourceARN'
```

## Handy shell wrappers

```bash
alias awsw='aws sts get-caller-identity --query "[Account,Arn]" --output text'
ec2ip() { aws ec2 describe-instances --filters "Name=tag:Name,Values=$1" --query 'Reservations[].Instances[].PrivateIpAddress' --output text; }
ssm() { aws ssm start-session --target "$(aws ec2 describe-instances --filters "Name=tag:Name,Values=$1" Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].InstanceId' --output text)"; }
```
