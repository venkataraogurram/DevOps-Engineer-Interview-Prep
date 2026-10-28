# Monitoring queries cheatsheet – CloudWatch Logs Insights, PromQL, Grafana, X-Ray

[← Cheatsheet index](README.md)

## CloudWatch Logs Insights

```
fields @timestamp, @message | sort @timestamp desc | limit 100
fields @timestamp, @message | filter @message like /(?i)error|exception/ | sort @timestamp desc
filter @message like /AccessDenied/ | stats count() by bin(5m)
fields @timestamp, @logStream | filter @message like "OOMKilled"
# JSON logs
fields @timestamp, level, msg, requestId | filter level = "ERROR" and service = "payments"
stats count(*) as errors by path | sort errors desc | limit 20
stats avg(latency), pct(latency, 50), pct(latency, 95), pct(latency, 99), max(latency) by bin(1m)
stats count() by status | sort status
stats count_distinct(userId) as users by bin(1h)
# parse unstructured
parse @message /(?<ip>\d+\.\d+\.\d+\.\d+) .* "(?<method>\w+) (?<path>\S+) .*" (?<status>\d{3}) (?<bytes>\d+) (?<latency>[\d.]+)/
| filter status >= 500 | stats count() as c by path | sort c desc
parse @message "* [*] *" as ts, level, rest
# Lambda
filter @type = "REPORT" | stats max(@maxMemoryUsed/1024/1024) as maxMB, avg(@duration), pct(@duration, 99), sum(@initDuration > 0) as coldStarts by bin(5m)
filter @message like /Task timed out/ | stats count() by bin(1h)
# ALB access logs (if shipped to CW) / VPC Flow Logs
fields @timestamp, srcAddr, dstAddr, dstPort, action | filter action = "REJECT" and dstPort = 443 | stats count() by srcAddr | sort count() desc
# EKS control plane audit
fields @timestamp, user.username, verb, objectRef.resource, objectRef.name, responseStatus.code | filter verb = "delete" and objectRef.resource = "secrets"
fields @timestamp, @message | filter @logStream like /authenticator/ | filter @message like /access denied/
# CloudTrail (if in CW Logs)
fields eventTime, eventName, userIdentity.arn, errorCode, errorMessage | filter errorCode like /Denied|Unauthorized/ | stats count() by eventName, userIdentity.arn
# dedupe / display
dedup requestId | display @timestamp, msg
unmask @message        # data-protection masked fields
```

Metric filter patterns: `{ $.level = "ERROR" }`, `{ $.status >= 500 }`, `"ERROR" -"ERROR_IGNORE"`, `[ip, user, username, timestamp, request, status_code = 5*, bytes]`.

Alarm JSON essentials: `--comparison-operator GreaterThanThreshold --threshold 5 --evaluation-periods 3 --datapoints-to-alarm 2 --period 60 --statistic Sum --treat-missing-data notBreaching`.

Metric math: `m1/m2*100` (error rate), `FILL(m1, 0)`, `RATE(m1)`, `ANOMALY_DETECTION_BAND(m1, 2)`, `SEARCH('{AWS/EC2,InstanceId} MetricName="CPUUtilization"', 'Average', 300)`, `INSIGHT_RULE_METRIC`, `SUM(METRICS())`, `IF(m1 > 100, 1, 0)`.

## PromQL

```promql
# instant / range
up == 0
rate(http_requests_total[5m])                                        # per-second rate
sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total[5m]))   # error ratio
histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))                        # p95
increase(http_requests_total[1h]); irate(...)  (spiky); changes(process_start_time_seconds[1h]) > 0  (restarts)
avg_over_time(cpu_usage[10m]); max_over_time(...); predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0   # disk full in 24h
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))); bottomk; count by (namespace) (kube_pod_info)
absent(up{job="api"}); label_replace(); sort_desc()
# node
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}
rate(node_network_receive_bytes_total[5m])
# kubernetes (kube-state-metrics + cadvisor)
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m])) / sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})   # cpu vs request
container_memory_working_set_bytes{container!=""} / on (namespace, pod, container) kube_pod_container_resource_limits{resource="memory"} > 0.9
sum by (namespace) (kube_pod_container_status_restarts_total) ; increase(kube_pod_container_status_restarts_total[1h]) > 3
kube_pod_status_phase{phase="Pending"} == 1 ; kube_deployment_status_replicas_unavailable > 0
kube_node_status_condition{condition="Ready", status="true"} == 0
rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m]) > 0.25   # throttling
kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas    # HPA maxed
# SLO burn rate (99.9%, 30d): fast window
(sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h]))) > (14.4 * 0.001)
```

Alert rule:
```yaml
groups:
  - name: api
    rules:
      - alert: HighErrorRate
        expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels: { severity: page }
        annotations: { summary: "{{ $labels.service }} error rate {{ $value | humanizePercentage }}", runbook_url: "https://runbooks/api-errors" }
```

Alertmanager: `route` (group_by, group_wait, repeat_interval, receivers), `inhibit_rules`, `silences`. `amtool alert query`, `promtool check rules`, `promtool query instant`.

Prometheus ops: `/-/healthy`, `/-/reload`, `/targets`, `/api/v1/query?query=up`, `curl -s localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health!="up")'`. Scrape via `ServiceMonitor`/`PodMonitor` (Prometheus Operator) or annotations `prometheus.io/scrape: "true"`, `prometheus.io/port`.

## Grafana

- Variables: `label_values(up, job)`, `$__rate_interval`, `$__range`, `$__interval`; chained variables; `=~ "$ns"` multi-value.
- Panel tips: units, thresholds, legend `{{pod}}`, transformations, `Stat` for SLO, `State timeline` for up/down; annotations from deploy events.
- Provisioning: `provisioning/datasources/*.yaml`, `provisioning/dashboards/*.yaml` + JSON; `grafana-cli`, API `POST /api/dashboards/db`; Terraform `grafana_dashboard`.
- Alerting: unified alerting, contact points, notification policies; Amazon Managed Grafana uses SigV4 datasources (AMP, CloudWatch).

## X-Ray / tracing

```
# X-Ray filter expressions
service("payments-api") AND fault
responsetime > 2 AND http.status = 200
annotation.customer_tier = "gold" AND duration > 1
edge("payments-api", "DynamoDB") { error }
http.url CONTAINS "/pay"  ;  id(trace_id)  ;  group("prod-payments")
```
`aws xray get-trace-summaries --start-time ... --end-time ... --filter-expression 'fault'`; `aws xray batch-get-traces --trace-ids ...`; sampling rules via `aws xray create-sampling-rule`. ADOT collector config: receivers (otlp, prometheus) → processors (batch, resourcedetection) → exporters (awsxray, awsemf, prometheusremotewrite to AMP).

## Golden-signal dashboard template (per service)

Row 1: request rate · error rate (%) · p50/p95/p99 latency · saturation (CPU vs request, memory vs limit).
Row 2: pods ready/desired · restarts · HPA current/max · throttling.
Row 3: dependencies: DB connections/latency, queue depth/age, downstream error rate.
Row 4: deploy annotations, SLO burn rate, error budget remaining.
