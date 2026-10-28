# 21. Scripting – advanced Python and Bash

[← Back to index](README.md)

**Q208. Write a Python function that returns all EC2 instances across all regions missing a required tag.**
```python
import boto3
from concurrent.futures import ThreadPoolExecutor

REQUIRED = {"Owner", "Env", "CostCenter"}

def scan_region(region):
    ec2 = boto3.client("ec2", region_name=region)
    out = []
    for page in ec2.get_paginator("describe_instances").paginate(
            Filters=[{"Name": "instance-state-name", "Values": ["running", "stopped"]}]):
        for r in page["Reservations"]:
            for i in r["Instances"]:
                tags = {t["Key"] for t in i.get("Tags", [])}
                missing = REQUIRED - tags
                if missing:
                    out.append({"region": region, "id": i["InstanceId"], "missing": sorted(missing)})
    return out

def main():
    regions = [r["RegionName"] for r in boto3.client("ec2", region_name="us-east-1").describe_regions()["Regions"]]
    with ThreadPoolExecutor(max_workers=8) as ex:
        results = [x for batch in ex.map(scan_region, regions) for x in batch]
    for r in results:
        print(f'{r["region"]}\t{r["id"]}\tmissing={",".join(r["missing"])}')
    return 1 if results else 0
```
Talk through: pagination, set arithmetic for missing tags, threads because the work is I/O bound, non-zero exit so a pipeline can gate on it, and that the role needs `ec2:DescribeInstances` and `ec2:DescribeRegions` only.

---

**Q209. Bash: rotate a log directory keeping the 7 newest files, safely.**
```bash
#!/usr/bin/env bash
set -euo pipefail
dir=${1:?usage: $0 <dir>}; keep=${2:-7}
[[ -d "$dir" ]] || { echo "not a dir: $dir" >&2; exit 2; }
# newest first, NUL-safe for odd filenames, skip the first $keep
find "$dir" -maxdepth 1 -type f -name '*.log' -printf '%T@ %p\0' \
  | sort -z -rn \
  | tail -z -n +"$((keep + 1))" \
  | cut -z -d' ' -f2- \
  | xargs -0 -r rm -v --
```
Points: `set -euo pipefail`, parameter validation with `${1:?}`, NUL-delimited pipeline so spaces or newlines in names can't break it, `xargs -r` to do nothing on empty input, `--` to stop option parsing. On macOS `find -printf` doesn't exist; use `stat -f` or Python.

---

**Q210. Parse a Kubernetes `kubectl get pods -o json` and report containers restarting more than N times, grouped by namespace.**
```python
import json, subprocess, sys
from collections import defaultdict

threshold = int(sys.argv[1]) if len(sys.argv) > 1 else 3
raw = subprocess.run(["kubectl", "get", "pods", "-A", "-o", "json"], check=True, capture_output=True, text=True).stdout
by_ns = defaultdict(list)
for pod in json.loads(raw)["items"]:
    for cs in pod["status"].get("containerStatuses", []):
        if cs.get("restartCount", 0) > threshold:
            reason = (cs.get("lastState", {}).get("terminated") or {}).get("reason", "?")
            by_ns[pod["metadata"]["namespace"]].append((pod["metadata"]["name"], cs["name"], cs["restartCount"], reason))
for ns, rows in sorted(by_ns.items()):
    print(f"== {ns}")
    for name, c, n, reason in sorted(rows, key=lambda r: -r[2]):
        print(f"  {name}/{c}: {n} restarts, last reason {reason}")
sys.exit(1 if by_ns else 0)
```
Mention the alternative with the official `kubernetes` Python client, and that `lastState.terminated.reason` gives OOMKilled vs Error, which decides the next step.

---

**Q211. Write a Bash health-check loop with timeout and exponential backoff.**
```bash
wait_for_url() {
  local url=$1 timeout=${2:-120} delay=1 start=$SECONDS code
  while (( SECONDS - start < timeout )); do
    code=$(curl -fsS -o /dev/null -w '%{http_code}' --max-time 5 "$url" || true)
    [[ "$code" == "200" ]] && { echo "healthy after $((SECONDS - start))s"; return 0; }
    echo "got '${code:-timeout}', retrying in ${delay}s" >&2
    sleep "$delay"; (( delay = delay < 16 ? delay * 2 : 16 ))
  done
  echo "timeout after ${timeout}s" >&2; return 1
}
wait_for_url https://app.internal/health 180 || kubectl rollout undo deploy/app
```
Points: `$SECONDS` builtin for elapsed time, `|| true` so a failed curl doesn't trip `set -e`, capped backoff, return codes used by the caller.

---

**Q212. Python: safely update a JSON config file in place with a backup and atomic write.**
```python
import json, os, shutil, tempfile
from pathlib import Path

def update_config(path: Path, **changes):
    data = json.loads(path.read_text())
    data.update(changes)
    shutil.copy2(path, path.with_suffix(path.suffix + ".bak"))
    fd, tmp = tempfile.mkstemp(dir=path.parent, prefix=".tmp-")
    with os.fdopen(fd, "w") as fh:
        json.dump(data, fh, indent=2, sort_keys=True); fh.write("\n")
        fh.flush(); os.fsync(fh.fileno())
    os.replace(tmp, path)      # atomic on POSIX
```
Why atomic: a crash mid-write otherwise leaves a truncated config that breaks the next service start. Same idea applies to writing kubeconfigs, Terraform tfvars, or anything a daemon reads.

---

**Q213. How do you structure a Python CLI tool your team will maintain?**
`argparse` or `click` for subcommands; a `main()` returning an exit code; logging via `logging` with a `--verbose` flag, never `print` for diagnostics; config from env vars with a `--profile`/`--region` override; boto3 session created once and passed in; pure functions for logic so they're testable with `moto` or stub clients; `pyproject.toml` with pinned dependencies and an entry point; `ruff` and `mypy` in pre-commit; `pytest` with fixtures; a Dockerfile so it runs the same in CI; semantic versioning and a changelog.

---

**Q214. Bash pitfalls you check for in code review?**
Unquoted variables (`rm -rf $dir/` with empty dir), `for f in $(ls)` instead of globs, parsing `ls`, missing `set -euo pipefail`, `cd` without `|| exit`, `[ ]` with unquoted strings, using `echo` for data with backslashes (use `printf`), `sudo` inside scripts without need, `eval`, temp files without `mktemp` and cleanup trap, assuming GNU tools on macOS, backticks instead of `$()`, `==` inside `[ ]`, not handling `IFS`, and anything that `shellcheck` would flag. Also: scripts that aren't idempotent and scripts with embedded credentials.

---

**Q215. Write a Python Lambda that reacts to an EC2 state-change event and tags the instance with the launcher's identity.**
```python
import boto3, logging
log = logging.getLogger(); log.setLevel("INFO")
ec2 = boto3.client("ec2")

def handler(event, context):
    detail = event["detail"]
    if detail.get("eventName") != "RunInstances" or detail.get("errorCode"):
        return
    user = detail["userIdentity"].get("arn", "unknown").split("/")[-1]
    ids = [i["instanceId"] for i in detail["responseElements"]["instancesSet"]["items"]]
    ec2.create_tags(Resources=ids, Tags=[{"Key": "CreatedBy", "Value": user}, {"Key": "CreatedAt", "Value": detail["eventTime"]}])
    log.info("tagged %s with CreatedBy=%s", ids, user)
```
Trigger: EventBridge rule on CloudTrail `RunInstances` via `source: aws.ec2, detail-type: AWS API Call via CloudTrail`. Role: `ec2:CreateTags` only. Talk about idempotency (re-tagging is harmless) and the DLQ for failed invocations.

---

**Q216. Compare `subprocess.run`, `os.system`, and the AWS SDK for automation.**
`os.system` runs through a shell with no output capture and injection risk; avoid. `subprocess.run([...], check=True, capture_output=True, text=True)` with a list avoids shell parsing, captures stdout/stderr, and raises on failure; use for wrapping CLIs like `kubectl` or `terraform`. For AWS, boto3 beats shelling to the CLI: typed responses, pagination, retries, no parsing of text. Use `shell=True` only when you need pipes and the command string is fully under your control.

---

**Q217. How would you write a script to find unused security groups and why is "unused" tricky?**
Collect all SG IDs, then subtract SGs referenced by ENIs (`describe-network-interfaces` covers EC2, RDS, Lambda, ELB, EKS pods), by launch templates and ASG launch configurations (future use), by other SGs' rules (referenced SGs), and the default SG per VPC which can't be deleted. "Unused" by ENI alone would delete SGs that templates depend on. Output a dry-run list with the reason it's considered unused, and delete only with `--apply` after a review period, logging each deletion.

---

**Q218. Explain Python generators and why they matter for log processing.**
A generator yields items lazily instead of building a list, so processing a 20 GB log uses constant memory: `for line in open(path)` is already a generator; chaining `(parse(l) for l in lines if "ERROR" in l)` keeps it lazy. Combine with `itertools.islice` for sampling, `collections.Counter` for aggregation, and `gzip.open` for compressed files. The same principle applies to boto3 paginators, which are generators of pages.

[← Back to index](README.md)
