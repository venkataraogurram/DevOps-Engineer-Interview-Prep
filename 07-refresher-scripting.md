# Refresher – Scripting (Bash + Python) for automation rounds

Interviewers want: correct logic, error handling, idempotency, and that you reach for the right tool (AWS CLI + jq vs boto3). Write these from scratch at least once.

## 1. Bash essentials

```bash
#!/usr/bin/env bash
set -euo pipefail          # exit on error, undefined var, pipe failure
IFS=$'\n\t'
trap 'echo "ERROR line $LINENO: $BASH_COMMAND" >&2' ERR
trap 'rm -f "$tmp"' EXIT   # cleanup
tmp=$(mktemp)

usage() { echo "Usage: $0 -e <env> [-d]" >&2; exit 1; }
DRY_RUN=false
while getopts "e:d" opt; do
  case $opt in
    e) ENV=$OPTARG ;;
    d) DRY_RUN=true ;;
    *) usage ;;
  esac
done
: "${ENV:?-e env is required}"            # fail if unset
command -v aws >/dev/null || { echo "aws cli missing" >&2; exit 2; }

log() { printf '%s %s\n' "$(date -u +%FT%TZ)" "$*" >&2; }
run() { if $DRY_RUN; then log "DRY: $*"; else "$@"; fi; }
```

Patterns:

```bash
# Read lines safely
while IFS= read -r line; do echo "$line"; done < file.txt
# Arrays
arr=(a b c); echo "${arr[@]}"; echo "${#arr[@]}"; for x in "${arr[@]}"; do ...; done
mapfile -t ids < <(aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --output text | tr '\t' '\n')
# String ops
${var#prefix} ${var%suffix} ${var/old/new} ${var:-default} ${var:=assign} ${#var} ${var^^} ${var,,}
# Conditionals
[[ -f "$f" ]] [[ -d ]] [[ -z "$s" ]] [[ -n ]] [[ "$a" == "$b" ]] [[ "$a" =~ ^[0-9]+$ ]] (( n > 5 ))
# Arithmetic
n=$(( n + 1 )); (( n++ ))
# Functions return via stdout
get_ip() { aws ec2 describe-instances --instance-ids "$1" --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text; }
ip=$(get_ip i-123)
# Here-doc
cat <<EOF > "$tmp"
env=$ENV
EOF
# Retry with backoff
retry() { local n=0 max=5; until "$@"; do n=$((n+1)); (( n >= max )) && return 1; sleep $((2**n)); done; }
retry curl -fsS https://example.com/health
# Parallel
xargs -P 8 -I{} sh -c 'process {}' < list.txt
# Lock to avoid double-run (cron)
exec 9>/var/lock/myjob.lock; flock -n 9 || { echo "already running"; exit 0; }
```

Text tools: `grep -E -o -c -v`, `awk '{print $1}' / awk -F, '$3 > 100 {sum+=$3} END {print sum}'`, `sed -E 's/x/y/g'`, `sort | uniq -c | sort -rn | head`, `cut -d' ' -f1`, `tr`, `wc -l`, `jq -r '.Reservations[].Instances[] | select(.State.Name=="running") | .InstanceId'`, `yq`, `curl -fsS -o /dev/null -w '%{http_code}'`, `find . -name '*.log' -mtime +7 -delete`, `du -sh * | sort -h`, `df -h`, `ps aux --sort=-%mem | head`, `ss -tulpn`, `journalctl -u app --since '1 hour ago'`, `systemctl status`, `tail -F`, `nc -zv host 443`, `openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -dates`.

Exit codes: `$?`, 0 success; `||` and `&&`; `set -e` doesn't fire inside `if`/`||` contexts – know that.

## 2. Classic Bash tasks

### 2a. Count HTTP status codes per endpoint from an access log

```
# format: 10.0.1.5 - - [28/Oct/2026:10:01:02 +0530] "GET /api/pay HTTP/1.1" 502 123 0.342
```

```bash
awk '{ split($6, m, "\""); path=$7; code=$9; key=path" "code; c[key]++ } END { for (k in c) print c[k], k }' access.log | sort -rn | head -20
# Only 5xx, per endpoint
awk '$9 ~ /^5/ { c[$7]++ } END { for (p in c) print c[p], p }' access.log | sort -rn
# p95 latency (last field)
awk '{print $NF}' access.log | sort -n | awk '{a[NR]=$1} END {print a[int(NR*0.95)]}'
# Requests per minute
awk -F'[:[]' '{print $3":"$4}' access.log | sort | uniq -c
# Top client IPs
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
```

### 2b. Disk usage alert

```bash
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD=${1:-80}
df -P -x tmpfs -x devtmpfs | awk 'NR>1 {gsub("%","",$5); print $5, $6}' | while read -r pct mount; do
  if (( pct >= THRESHOLD )); then
    msg="Disk ${mount} at ${pct}% on $(hostname)"
    echo "$msg" >&2
    aws cloudwatch put-metric-data --namespace Custom/Disk --metric-name UsedPercent --dimensions Host="$(hostname)",Mount="$mount" --value "$pct"
    aws sns publish --topic-arn "$SNS_TOPIC" --message "$msg" >/dev/null
  fi
done
```

### 2c. Stop untagged EC2 instances (dry-run by default)

```bash
#!/usr/bin/env bash
set -euo pipefail
REGION=${AWS_REGION:-ap-south-1}; APPLY=${1:-}
ids=$(aws ec2 describe-instances --region "$REGION" \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[?!not_null(Tags[?Key==`Owner`].Value|[0])].InstanceId' --output text)
[[ -z "$ids" ]] && { echo "nothing to do"; exit 0; }
echo "Untagged running instances: $ids"
if [[ "$APPLY" == "--apply" ]]; then
  aws ec2 stop-instances --region "$REGION" --instance-ids $ids
else
  echo "dry-run; pass --apply to stop"
fi
```

### 2d. Delete EBS snapshots older than 30 days (own account)

```bash
cutoff=$(date -u -d '30 days ago' +%FT%TZ 2>/dev/null || date -u -v-30d +%FT%TZ)   # GNU vs BSD date
aws ec2 describe-snapshots --owner-ids self --query "Snapshots[?StartTime<'${cutoff}'].SnapshotId" --output text | tr '\t' '\n' | while read -r id; do
  [[ -n "$id" ]] && aws ec2 delete-snapshot --snapshot-id "$id" && echo "deleted $id"
done
```

### 2e. Health check + rollback helper

```bash
deploy_and_verify() {
  local img=$1
  kubectl set image deploy/payments-api api="$img" -n payments
  if ! kubectl rollout status deploy/payments-api -n payments --timeout=180s; then
    echo "rollout failed, rolling back" >&2
    kubectl rollout undo deploy/payments-api -n payments
    return 1
  fi
  for i in {1..10}; do
    code=$(curl -fsS -o /dev/null -w '%{http_code}' https://payments.internal/health || true)
    [[ "$code" == "200" ]] && { echo "healthy"; return 0; }
    sleep 3
  done
  kubectl rollout undo deploy/payments-api -n payments; return 1
}
```

## 3. Python essentials for DevOps

```python
#!/usr/bin/env python3
"""Script skeleton."""
import argparse, json, logging, os, sys
from pathlib import Path
import boto3
from botocore.exceptions import ClientError

logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"), format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)

def parse_args():
    p = argparse.ArgumentParser(description="...")
    p.add_argument("--region", default=os.getenv("AWS_REGION", "ap-south-1"))
    p.add_argument("--apply", action="store_true", help="actually make changes")
    return p.parse_args()

def main():
    args = parse_args()
    session = boto3.Session(region_name=args.region)      # profile_name=... for local
    ec2 = session.client("ec2")
    try:
        ...
    except ClientError as e:
        log.error("AWS error %s: %s", e.response["Error"]["Code"], e.response["Error"]["Message"])
        return 1
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

boto3 patterns:

```python
# Pagination – always
paginator = ec2.get_paginator("describe_instances")
for page in paginator.paginate(Filters=[{"Name": "instance-state-name", "Values": ["running"]}]):
    for r in page["Reservations"]:
        for i in r["Instances"]:
            tags = {t["Key"]: t["Value"] for t in i.get("Tags", [])}
# Waiters
ec2.get_waiter("instance_stopped").wait(InstanceIds=ids)
# Resource API
s3 = session.resource("s3"); s3.Bucket("b").objects.filter(Prefix="logs/")
# Assume role
sts = session.client("sts")
creds = sts.assume_role(RoleArn=arn, RoleSessionName="script")["Credentials"]
other = boto3.Session(aws_access_key_id=creds["AccessKeyId"], aws_secret_access_key=creds["SecretAccessKey"], aws_session_token=creds["SessionToken"])
# Retries/config
from botocore.config import Config
ec2 = session.client("ec2", config=Config(retries={"max_attempts": 10, "mode": "adaptive"}))
# Secrets
secret = json.loads(session.client("secretsmanager").get_secret_value(SecretId="prod/db")["SecretString"])
# CloudWatch metric
cw.put_metric_data(Namespace="Custom/App", MetricData=[{"MetricName": "Errors", "Value": n, "Unit": "Count", "Dimensions": [{"Name": "Service", "Value": "payments"}]}])
```

Stdlib you'll use: `subprocess.run(["kubectl","get","pods","-o","json"], capture_output=True, text=True, check=True)`, `json`, `re`, `datetime`/`timedelta`/`timezone.utc`, `collections.Counter/defaultdict`, `pathlib`, `csv`, `concurrent.futures.ThreadPoolExecutor`, `requests` (3rd party) or `urllib.request`, `yaml` (PyYAML), `tempfile`, `shutil`, `time.sleep` with backoff, `dataclasses`, type hints, f-strings, context managers, generators for big files.

## 4. Classic Python tasks

### 4a. Parse log, count 5xx per endpoint, output top N as JSON

```python
import re, sys, json
from collections import Counter

LINE = re.compile(r'"(?P<method>[A-Z]+) (?P<path>\S+) HTTP/[\d.]+" (?P<status>\d{3}) (?P<bytes>\d+) (?P<latency>[\d.]+)')

def parse(path):
    with open(path, encoding="utf-8", errors="replace") as fh:
        for line in fh:                      # streams; fine for GB files
            m = LINE.search(line)
            if m:
                yield m.groupdict()

def main(path, top=10):
    errors = Counter()
    latencies = []
    for rec in parse(path):
        if rec["status"].startswith("5"):
            errors[rec["path"]] += 1
        latencies.append(float(rec["latency"]))
    latencies.sort()
    p95 = latencies[int(len(latencies) * 0.95) - 1] if latencies else None
    print(json.dumps({"top_5xx": errors.most_common(top), "p95_latency": p95}, indent=2))

if __name__ == "__main__":
    main(sys.argv[1])
```

### 4b. Find security groups open to the world on sensitive ports

```python
SENSITIVE = {22, 3389, 3306, 5432, 1433, 6379, 27017}

def open_to_world(perm):
    return any(r.get("CidrIp") == "0.0.0.0/0" for r in perm.get("IpRanges", [])) or \
           any(r.get("CidrIpv6") == "::/0" for r in perm.get("Ipv6Ranges", []))

def port_hits(perm):
    if perm.get("IpProtocol") == "-1":
        return SENSITIVE
    lo, hi = perm.get("FromPort", 0), perm.get("ToPort", 65535)
    return {p for p in SENSITIVE if lo <= p <= hi}

findings = []
for page in ec2.get_paginator("describe_security_groups").paginate():
    for sg in page["SecurityGroups"]:
        for perm in sg["IpPermissions"]:
            if open_to_world(perm) and (hits := port_hits(perm)):
                findings.append({"sg": sg["GroupId"], "vpc": sg.get("VpcId"), "ports": sorted(hits)})
print(json.dumps(findings, indent=2))
```

### 4c. Rotate/delete old AMIs keeping the last N per Role tag

```python
from datetime import datetime, timezone
from collections import defaultdict

def cleanup_amis(ec2, keep=3, apply=False):
    by_role = defaultdict(list)
    for img in ec2.describe_images(Owners=["self"])["Images"]:
        tags = {t["Key"]: t["Value"] for t in img.get("Tags", [])}
        if "Role" in tags:
            by_role[tags["Role"]].append(img)
    for role, imgs in by_role.items():
        imgs.sort(key=lambda i: i["CreationDate"], reverse=True)
        for img in imgs[keep:]:
            snaps = [m["Ebs"]["SnapshotId"] for m in img["BlockDeviceMappings"] if "Ebs" in m]
            print(f"{'DELETE' if apply else 'DRY'} {role} {img['ImageId']} {img['CreationDate']} snaps={snaps}")
            if apply:
                ec2.deregister_image(ImageId=img["ImageId"])
                for s in snaps:
                    ec2.delete_snapshot(SnapshotId=s)
```

### 4d. Check a list of URLs concurrently, exit non-zero if any fails

```python
import sys, concurrent.futures as cf, urllib.request

def check(url, timeout=5):
    try:
        with urllib.request.urlopen(url, timeout=timeout) as r:
            return url, r.status, None
    except Exception as e:                 # noqa: BLE001 – report everything
        return url, None, str(e)

urls = [l.strip() for l in open(sys.argv[1]) if l.strip()]
with cf.ThreadPoolExecutor(max_workers=16) as ex:
    results = list(ex.map(check, urls))
bad = [r for r in results if r[1] != 200]
for url, status, err in results:
    print(f"{status or 'ERR':>4} {url} {err or ''}")
sys.exit(1 if bad else 0)
```

### 4e. Kubernetes: list pods not Ready and why (via kubectl JSON)

```python
import json, subprocess

out = subprocess.run(["kubectl", "get", "pods", "-A", "-o", "json"], capture_output=True, text=True, check=True).stdout
for pod in json.loads(out)["items"]:
    conds = {c["type"]: c for c in pod["status"].get("conditions", [])}
    ready = conds.get("Ready", {}).get("status") == "True"
    phase = pod["status"]["phase"]
    if not ready and phase != "Succeeded":
        reasons = []
        for cs in pod["status"].get("containerStatuses", []):
            st = cs["state"]
            if "waiting" in st:   reasons.append(f"{cs['name']}: {st['waiting'].get('reason')}")
            if "terminated" in st: reasons.append(f"{cs['name']}: {st['terminated'].get('reason')}")
            if cs.get("restartCount", 0) > 3: reasons.append(f"{cs['name']}: restarts={cs['restartCount']}")
        print(f"{pod['metadata']['namespace']}/{pod['metadata']['name']} {phase} {'; '.join(reasons) or conds.get('Ready',{}).get('message','')}")
```

### 4f. Lambda handler pattern (serverless question)

```python
import json, os, logging, boto3
log = logging.getLogger(); log.setLevel(logging.INFO)
TABLE = os.environ["TABLE_NAME"]
ddb = boto3.resource("dynamodb").Table(TABLE)    # init outside handler = reused across warm invocations

def handler(event, context):
    log.info(json.dumps(event))
    for rec in event.get("Records", []):          # SQS/SNS/S3 batch
        body = json.loads(rec["body"]) if "body" in rec else rec
        ddb.put_item(Item={"pk": body["id"], "payload": json.dumps(body)})
    return {"statusCode": 200, "body": json.dumps({"processed": len(event.get("Records", []))})}
```

## 5. Talking points on "how you automate"

- Idempotent by design: check current state before acting; tags as markers; dry-run default, `--apply` to execute.
- Observability of the script itself: structured logs, exit codes, metrics to CloudWatch, failure → SNS.
- Credentials: never embedded; role assumption; least privilege for the script's role.
- Where it runs: Lambda on EventBridge schedule for cloud hygiene; SSM Automation/Run Command for fleet tasks; CronJob in K8s; CI job for repo tasks.
- Testing: `pytest` with `moto` for boto3 mocks, `bats` for Bash, `shellcheck` + `ruff` in pre-commit.
- Bash vs Python rule: Bash for <50 lines glue around CLI tools; Python when you need data structures, error handling, JSON, concurrency, or tests.
