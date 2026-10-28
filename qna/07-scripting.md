# 7. Scripting

[← Back to index](README.md)

**Q63. When do you choose Bash and when Python?**
Bash for short glue around CLI tools, under about fifty lines, where I'm piping `aws`, `jq`, `grep` and `awk`. Python when I need data structures, real error handling, JSON manipulation, concurrency, unit tests, or anything that will live longer than a week. Both get `shellcheck` or `ruff` in pre-commit.

---

**Q64. What does `set -euo pipefail` do?**
`-e` exits on any command failure, `-u` errors on undefined variables, and `pipefail` makes a pipeline fail if any stage fails instead of just the last. Together they stop scripts from silently continuing after an error. I pair them with a `trap` on ERR to log the line and on EXIT to clean up temp files.

---

**Q65. How would you write a script to stop untagged EC2 instances?**
Describe running instances with a JMESPath filter for those missing the Owner tag, print them, and only stop if a `--apply` flag is passed, so dry-run is the default. Use pagination, handle no results, log what was done, and run it as a Lambda on an EventBridge schedule with a role limited to `ec2:Describe*` and `ec2:StopInstances` with a condition on the tag. Send a summary to SNS.

---

**Q66. Parse an access log and find the top endpoints by 5xx errors.**
In Bash: `awk '$9 ~ /^5/ {c[$7]++} END {for (p in c) print c[p], p}' access.log | sort -rn | head`. In Python: stream the file line by line, a compiled regex with named groups, a `Counter` keyed by path for statuses starting with 5, and print `most_common(10)` as JSON. Streaming matters so it works on multi-GB logs.

---

**Q67. How do you make a script idempotent?**
Check before act: look up the resource by name or tag and skip if it exists; use create-if-missing API semantics or Terraform instead of imperative creation; tag anything you create so re-runs recognise it; make deletes tolerate "not found"; keep a lock for cron jobs so overlapping runs don't double-execute.

---

**Q68. How do you handle AWS API throttling in scripts?**
Use the SDK's built-in retries with adaptive mode in botocore config, paginate instead of pulling everything, add jittered backoff for custom loops, batch where the API allows, and spread scheduled jobs so they don't all hit the API at the top of the hour.

---

[← Back to index](README.md)
