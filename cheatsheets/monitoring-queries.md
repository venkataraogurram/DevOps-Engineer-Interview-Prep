# Monitoring queries cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: query → what it does and when you reach for it.

## CloudWatch Logs Insights

- `fields @timestamp, @message | sort @timestamp desc | limit 100`

  Latest 100 lines. The starting point in any log group.

- `fields @timestamp, @message | filter @message like /(?i)error|exception/ | sort @timestamp desc`

  Case-insensitive regex filter for errors.

- `filter @message like /AccessDenied/ | stats count() by bin(5m)`

  Error count per 5-minute bucket: see when it started.

- `fields @timestamp, @logStream | filter @message like "OOMKilled"`

  Which stream (pod/instance) hit a condition.

- `fields @timestamp, level, msg, requestId | filter level = "ERROR" and service = "payments"`

  JSON logs are auto-parsed: filter on fields directly.

- `stats count(*) as errors by path | sort errors desc | limit 20`

  Top endpoints by error count.

- `stats avg(latency), pct(latency, 50), pct(latency, 95), pct(latency, 99), max(latency) by bin(1m)`

  Latency percentiles over time. Use p95/p99, not average.

- `stats count() by status | sort status`

  Status code distribution.

- `stats count_distinct(userId) as users by bin(1h)`

  Unique users per hour.

- `parse @message /(?<ip>\d+\.\d+\.\d+\.\d+) .* "(?<method>\w+) (?<path>\S+) .*" (?<status>\d{3}) (?<bytes>\d+) (?<latency>[\d.]+)/ | filter status >= 500 | stats count() as c by path | sort c desc`

  Extract fields from unstructured access logs with a named-group regex, then aggregate.

- `parse @message "* [*] *" as ts, level, rest`

  Glob-style parse when the format is simple.

- `filter @type = "REPORT" | stats max(@maxMemoryUsed/1024/1024) as maxMB, avg(@duration), pct(@duration, 99), sum(@initDuration > 0) as coldStarts by bin(5m)`

  Lambda: memory headroom, duration, and cold-start count from REPORT lines. Right-size memory from `maxMB`.

- `filter @message like /Task timed out/ | stats count() by bin(1h)`

  Lambda timeouts over time.

- `fields @timestamp, srcAddr, dstAddr, dstPort, action | filter action = "REJECT" and dstPort = 443 | stats count() by srcAddr | sort count() desc`

  VPC Flow Logs: who is being blocked on 443. Confirms a security group/NACL problem.

- `fields @timestamp, user.username, verb, objectRef.resource, objectRef.name, responseStatus.code | filter verb = "delete" and objectRef.resource = "secrets"`

  EKS audit log: who deleted secrets.

- `fields @timestamp, @message | filter @logStream like /authenticator/ | filter @message like /access denied/`

  EKS authenticator: IAM principals being refused (access entry missing).

- `fields eventTime, eventName, userIdentity.arn, errorCode, errorMessage | filter errorCode like /Denied|Unauthorized/ | stats count() by eventName, userIdentity.arn`

  CloudTrail in CloudWatch: AccessDenied by action and principal. The IAM debugging query.

- `dedup requestId | display @timestamp, msg`

  One line per request; `display` picks output columns.

- `unmask @message`

  Reveal fields hidden by a data-protection policy (needs permission).

Metric filter patterns (log → metric): `{ $.level = "ERROR" }` JSON field · `{ $.status >= 500 }` · `"ERROR" -"ERROR_IGNORE"` plain text include/exclude · `[ip, user, username, timestamp, request, status_code = 5*, bytes]` space-delimited columns.

Alarm flags: `--comparison-operator GreaterThanThreshold --threshold 5 --evaluation-periods 3 --datapoints-to-alarm 2 --period 60 --statistic Sum --treat-missing-data notBreaching` → alarm when 2 of 3 one-minute datapoints exceed 5; missing data counts as OK (right for sparse error metrics).

- `m1/m2*100`

  Error rate as a percentage of requests.

- `FILL(m1, 0)`

  Treat missing datapoints as zero so sparse metrics graph and alarm sensibly.

- `RATE(m1)`

  Per-second rate of a counter.

- `ANOMALY_DETECTION_BAND(m1, 2)`

  Band of expected values (2 standard deviations); alarm outside it for seasonal metrics.

- `SEARCH('{AWS/EC2,InstanceId} MetricName="CPUUtilization"', 'Average', 300)`

  Dynamic set of metrics by search; dashboards that auto-include new instances.

- `SUM(METRICS())`

  Aggregate all metrics in the expression.

- `IF(m1 > 100, 1, 0)`

  Boolean series for composite logic.

## PromQL

- `up == 0`

  Targets Prometheus can't scrape. First check when metrics are missing.

- `rate(http_requests_total[5m])`

  Per-second request rate over a 5-minute window. Always wrap counters in `rate()`.

- `sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total[5m]))`

  Error ratio per service. The core SLI.

- `histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))`

  p95 latency from a histogram. Must aggregate by `le`.

- `increase(http_requests_total[1h])`

  Total increase over an hour (requests served).

- `irate(...)`

  Instant rate from the last two samples; spiky, for fast-moving graphs only.

- `changes(process_start_time_seconds[1h]) > 0`

  Processes that restarted in the last hour.

- `avg_over_time(cpu_usage[10m])` / `max_over_time(...)`

  Smoothed or peak values over a window.

- `predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0`

  Disk will be full within 24 hours based on the last 6 hours. Alert before it happens.

- `topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))`

  Five hottest pods. `bottomk` for the opposite.

- `count by (namespace) (kube_pod_info)`

  Pods per namespace.

- `absent(up{job="api"})`

  Fires when a series disappears entirely (job removed/renamed).

- `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

  Node CPU utilisation %.

- `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100`

  Node memory available %.

- `1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}`

  Root disk used fraction.

- `rate(node_network_receive_bytes_total[5m])`

  Network ingress bytes/s.

- `sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m])) / sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})`

  CPU usage vs request: >1 means under-requested, <0.3 means over-requested. Right-sizing input.

- `container_memory_working_set_bytes{container!=""} / on (namespace, pod, container) kube_pod_container_resource_limits{resource="memory"} > 0.9`

  Containers within 10% of their memory limit: OOM candidates.

- `increase(kube_pod_container_status_restarts_total[1h]) > 3`

  Crashlooping containers.

- `kube_pod_status_phase{phase="Pending"} == 1`

  Pods stuck Pending.

- `kube_deployment_status_replicas_unavailable > 0`

  Deployments not fully available.

- `kube_node_status_condition{condition="Ready", status="true"} == 0`

  NotReady nodes.

- `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m]) > 0.25`

  CPU throttling ratio: limits too tight.

- `kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas`

  HPA pinned at max: capacity ceiling reached.

- `(sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h]))) > (14.4 * 0.001)`

  SLO burn-rate fast alert for a 99.9% SLO: burning the 30-day budget 14.4× too fast over 1 hour.

```yaml
# Alert rule – fires only after the condition holds for 10 minutes, with a runbook link
groups:
  - name: api
    rules:
      - alert: HighErrorRate
        expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels: { severity: page }
        annotations: { summary: "{{ $labels.service }} error rate {{ $value | humanizePercentage }}", runbook_url: "https://runbooks/api-errors" }
```

- `promtool check rules rules.yml` / `promtool check config prometheus.yml`

  Validate before reload. CI step.

- `promtool query instant http://prom:9090 'up'`

  Run a query from the shell.

- `curl -s localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health!="up")'`

  Unhealthy scrape targets with the error reason.

- `curl -X POST localhost:9090/-/reload`

  Reload config without restart (needs `--web.enable-lifecycle`).

- `/targets`, `/-/healthy`, `/api/v1/query?query=up`

  UI and API endpoints.

- `amtool alert query` / `amtool silence add alertname=X --duration 2h`

  Alertmanager: current alerts / silence during maintenance.

- **Alertmanager config**

  `route` (group_by, group_wait, repeat_interval, receivers), `inhibit_rules` (suppress child alerts when the parent fires), silences.

- **Scraping targets**

  `ServiceMonitor`/`PodMonitor` CRDs with the Prometheus Operator, or pod annotations `prometheus.io/scrape: "true"`, `prometheus.io/port: "9102"`.

## Grafana

- `label_values(up, job)`

  Dashboard variable populated from a label.

- `$__rate_interval`

  Safe interval for `rate()` that adapts to zoom level; use it instead of a fixed `[5m]`.

- `$__range` / `$__interval`

  Selected time range / graph step.

- `{service=~"$svc"}`

  Multi-value variable in a query (regex match).

- **Panel settings**

  Units, thresholds (colour at SLO breach), legend `{{pod}}`, transformations (join, calculate), `Stat` for SLO numbers, `State timeline` for up/down history.

- **Annotations**

  Mark deploys on graphs from a deploy-events query; correlate regressions with releases.

- **Provisioning**

  `provisioning/datasources/*.yaml`, `provisioning/dashboards/*.yaml` + JSON files, or Terraform `grafana_dashboard`; dashboards as code in git.

- **Alerting**

  Unified alerting with contact points and notification policies; Amazon Managed Grafana authenticates to AMP/CloudWatch with SigV4.

## X-Ray filter expressions

- `service("payments-api") AND fault`

  Traces with 5xx faults in one service.

- `responsetime > 2 AND http.status = 200`

  Slow but successful requests.

- `annotation.customer_tier = "gold" AND duration > 1`

  Filter on custom annotations your code adds (indexed).

- `edge("payments-api", "DynamoDB") { error }`

  Errors on a specific downstream call.

- `http.url CONTAINS "/pay"`

  By path.

- `id(trace_id)`

  One trace by ID (from a log line).

- `aws xray get-trace-summaries --start-time ... --end-time ... --filter-expression 'fault'`

  CLI equivalent. `batch-get-traces --trace-ids` for full segments.

ADOT collector pipeline: receivers (otlp, prometheus) → processors (batch, resourcedetection) → exporters (awsxray, awsemf for CloudWatch metrics, prometheusremotewrite to AMP). One agent, three signals.

## Golden-signal dashboard layout (per service)

1. Request rate · error rate % · p50/p95/p99 latency · saturation (CPU vs request, memory vs limit).
2. Pods ready/desired · restarts · HPA current/max · CPU throttling.
3. Dependencies: DB connections and latency, queue depth and age, downstream error rate.
4. Deploy annotations · SLO burn rate · error budget remaining.
