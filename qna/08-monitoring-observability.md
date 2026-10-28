# 8. Monitoring and observability

[← Back to index](README.md)

**Q69. How do you monitor a production service?**
Start with the golden signals: latency, traffic, errors, saturation. CloudWatch metrics and alarms for infrastructure, custom metrics from the app via EMF or Prometheus, structured JSON logs to CloudWatch Logs with retention set, distributed tracing with X-Ray or OpenTelemetry, synthetic canaries on key journeys. Define SLOs and alert on error-budget burn rather than CPU. Every alert routes to an on-call rotation with a runbook link. Dashboards per service for humans; alerts for pages.

---

**Q70. CloudWatch vs ELK?**
CloudWatch is native and zero-ops with Logs Insights, metric filters and alarms, priced per GB ingested and scanned. ELK or OpenSearch gives full-text search, rich dashboards and cheaper long retention but needs operating or the managed OpenSearch Service. Many banks ship CloudWatch logs through Kinesis Firehose to OpenSearch or Splunk for central SIEM. I'd use CloudWatch for operational alerting and OpenSearch for cross-source investigation and compliance retention.

---

**Q71. What is the difference between monitoring and observability?**
Monitoring tells you a known condition happened: CPU above 80%, 5xx above 1%. Observability lets you ask new questions about the system from its outputs: metrics, logs and traces with enough context, like correlation IDs, to explain an incident you didn't predict. In practice: structured logs, trace propagation, high-cardinality labels, and tools that join them.

---

**Q72. How do you reduce alert fatigue?**
Alert on symptoms, not causes; use composite alarms so one incident is one page; set `treat missing data` correctly; use anomaly detection for seasonal metrics; tune thresholds from historical data; route informational alerts to a channel not a pager; review every page weekly and delete or fix noisy ones; every alert must be actionable with a runbook.

---

**Q73. Describe your incident response process.**
Detect through alarms, triage severity, open an incident channel, assign an incident commander and a communicator who updates stakeholders every 30 minutes, mitigate first (rollback, scale, failover) and root-cause later, resolve, then a blameless post-mortem within a few days with action items tracked to completion. Measure MTTD and MTTR.

---

[← Back to index](README.md)
