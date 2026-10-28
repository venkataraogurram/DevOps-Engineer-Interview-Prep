# Refresher – Monitoring, Docker, Microservices, Serverless

## 1. Monitoring and observability

### Vocabulary
- **Three pillars**: metrics (numbers over time), logs (events), traces (request path across services). Plus profiles and synthetics.
- **Golden signals** (Google SRE): latency, traffic, errors, saturation. **RED** for services (Rate, Errors, Duration), **USE** for resources (Utilization, Saturation, Errors).
- **SLI/SLO/SLA**: indicator (p99 latency < 300 ms), objective (99.9% of requests), agreement (contractual). Error budget = 1 − SLO; burn-rate alerts (fast 2 % in 1 h, slow 5 % in 6 h) instead of static thresholds.
- Alert on symptoms (user-facing SLO breach), not causes (CPU 80 %). Every alert has a runbook and an owner. Page only on actionable.

### CloudWatch
- **Metrics**: namespaces, dimensions, 1-min standard, high-resolution 1 s custom, 15 months retention (downsampled). Math expressions, `SEARCH()`, anomaly detection bands, Metrics Insights (SQL-like).
- **Alarms**: static, anomaly, composite (AND/OR to cut noise), `treat missing data` setting matters (`notBreaching` for sparse metrics), actions → SNS / ASG / EC2 / Systems Manager OpsItem.
- **Logs**: log groups/streams, retention per group (set it! default never expires = cost), **metric filters** (count `ERROR`), **Logs Insights** queries, subscription filters → Lambda/Kinesis/OpenSearch, Live Tail, data protection (mask PII), cross-account observability.
- **Agent**: unified CloudWatch agent (`amazon-cloudwatch-agent`) for OS metrics (mem/disk – not in default EC2 metrics) + log files; config in SSM Parameter Store; bake into Packer AMI.
- **EMF** (Embedded Metric Format): log JSON → metrics automatically; great for Lambda.
- **Container Insights** (EKS/ECS, with enhanced observability), **Lambda Insights**, **Application Signals** (SLOs + service map via ADOT), **X-Ray**, **Synthetics** canaries, **RUM**, **Evidently**, **Internet Monitor**.
- **Dashboards**: per service; cross-account; share read-only.

Logs Insights snippets:
```
fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 50
stats count(*) by bin(5m)
filter status >= 500 | stats count() as errors by path | sort errors desc
parse @message /(?<ip>\d+\.\d+\.\d+\.\d+)/ | stats count() by ip
fields @duration | stats pct(@duration, 95) as p95 by bin(1m)       # Lambda
```

### ELK / OpenSearch
- **Elasticsearch/OpenSearch** (store+search, indices, shards/replicas, ILM/ISM for hot-warm-cold-delete), **Logstash** (heavy ETL; grok filters) or **Fluent Bit/Fluentd** (light shippers; Fluent Bit as DaemonSet on EKS), **Kibana/OpenSearch Dashboards**, **Beats** (Filebeat, Metricbeat).
- AWS: Amazon OpenSearch Service (managed, fine-grained access, UltraWarm/cold), OpenSearch Serverless, Kinesis Data Firehose → OpenSearch with CloudWatch subscription filter.
- Gotchas: index mapping explosions, shard sizing (20–50 GB), retention costs, cluster going yellow/red (unassigned shards), JVM heap pressure.
- When to pick: need full-text search across many sources, long retention analytics, central SIEM. CloudWatch for AWS-native simplicity; many run both (CloudWatch → Firehose → OpenSearch/Splunk).

### Prometheus / Grafana (K8s standard)
- Pull model, `/metrics` scraping, PromQL (`rate(http_requests_total{status=~"5.."}[5m])`, `histogram_quantile(0.95, sum(rate(bucket[5m])) by (le))`), exporters (node, kube-state-metrics, blackbox), Alertmanager (grouping, inhibition, routes), long-term storage (Thanos, Cortex, **Amazon Managed Prometheus**), **Amazon Managed Grafana**.
- kube-prometheus-stack Helm chart; ServiceMonitor CRDs. OpenTelemetry (ADOT collector) as the vendor-neutral pipeline for traces/metrics/logs.

### Tracing
- X-Ray (segments/subsegments, sampling rules, service map, SDK or ADOT), OpenTelemetry → X-Ray/Jaeger/Tempo. Trace ID propagation via headers (`X-Amzn-Trace-Id`, W3C `traceparent`); correlate logs with trace ID.

### Incident basics to mention
- Detection → triage (severity) → comms (status channel, stakeholder updates every 30 min) → mitigation (rollback first, root-cause later) → resolution → blameless post-mortem with action items tracked. MTTD/MTTR as metrics.

## 2. Docker

### Dockerfile best practices
```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install --prefix=/install -r requirements.txt

FROM python:3.12-slim
RUN addgroup --system app && adduser --system --ingroup app --uid 10001 app
WORKDIR /app
COPY --from=build /install /usr/local
COPY --chown=app:app . .
USER app
EXPOSE 8080
ENV PYTHONUNBUFFERED=1
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request;urllib.request.urlopen('http://127.0.0.1:8080/healthz')" || exit 1
ENTRYPOINT ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]   # exec form → PID 1 gets SIGTERM
```
- Multi-stage builds (small final image, no build tools), pin base image (digest for prod), slim/distroless/alpine, `.dockerignore`, order layers from least to most frequently changing (deps before code), one process per container, non-root user, no secrets in layers (use BuildKit `--mount=type=secret`), labels (`org.opencontainers.image.revision`).
- `ENTRYPOINT` vs `CMD`: ENTRYPOINT fixed executable, CMD default args (overridable). Exec form (JSON array) vs shell form (wraps in `sh -c`, breaks signal handling).
- `COPY` vs `ADD`: ADD auto-extracts tars and fetches URLs; prefer COPY.
- Image = layers (read-only) + container writable layer. `docker history`, `dive` to inspect size.
- Networking: bridge (default), host, none, user-defined bridge gives DNS by container name. Volumes vs bind mounts vs tmpfs. Compose for local multi-service (`docker compose up -d`, `depends_on` with `condition: service_healthy`).
- Security: scan (Trivy/ECR), rootless, read-only FS, drop capabilities, no `--privileged`, don't mount docker.sock, sign (cosign), SBOM.
- Registries: ECR (lifecycle policies to expire untagged, tag immutability, replication, pull-through cache for Docker Hub rate limits).
- Commands: `docker build -t img:tag --platform linux/amd64 .`, `docker run -d -p 8080:8080 --env-file .env --name x img`, `docker exec -it x sh`, `docker logs -f x`, `docker inspect`, `docker stats`, `docker system prune -af`, `docker buildx build --platform linux/amd64,linux/arm64 --push`.
- Troubleshooting: container exits immediately (CMD finished / wrong entrypoint), `exec format error` (arch mismatch – M1 Mac building arm64 for x86 nodes → `--platform`), permission denied on volume (uid mismatch), OOM (`--memory`), can't reach other container (use compose network/service name, not localhost).

## 3. Microservices architecture

- Each service owns its data; communicate via API (REST/gRPC) or events (SNS/SQS/EventBridge/Kafka-MSK). Avoid shared DBs and distributed transactions; use sagas/outbox pattern, idempotency keys, eventual consistency.
- Cross-cutting: API gateway (API Gateway/ALB/Kong), service discovery (K8s DNS, Cloud Map), config (ConfigMaps/AppConfig/SSM), secrets, centralized logging with correlation IDs, distributed tracing, circuit breakers/retries with backoff + jitter, bulkheads, timeouts everywhere, rate limiting.
- Deployment independence: one pipeline per service, contract tests (Pact), versioned APIs, backwards-compatible changes.
- Patterns: strangler fig for monolith migration, BFF, CQRS, event sourcing (use sparingly), sidecar (Envoy), service mesh for mTLS/traffic policy.
- Trade-offs to state: operational complexity, network latency, debugging across services, data consistency. Don't start with microservices for a small team.
- Twelve-factor app: config in env, stateless processes, logs to stdout, disposability, dev/prod parity, backing services as attached resources.

## 4. Serverless

- Compute: Lambda (see `02`), Fargate (serverless containers for ECS/EKS), App Runner.
- Integration: API Gateway (REST: usage plans, WAF, caching, request validation; HTTP: cheaper, JWT authorizer), EventBridge (buses, rules, schemas, Scheduler, Pipes), SQS (standard vs FIFO, visibility timeout ≥ 6× Lambda timeout, DLQ with redrive, long polling), SNS (fan-out, filter policies), Step Functions (Standard: long-running, exactly-once; Express: high-volume, at-least-once; JSONata/JSONPath; Choice/Map/Parallel/Wait; retries/catch per state), DynamoDB, S3, Cognito, AppSync.
- Design: idempotent handlers (SQS at-least-once), small single-purpose functions, async where possible, DLQs + alarms on `DeadLetterErrors`/`Throttles`/`ConcurrentExecutions`, structured logs with request IDs (Lambda Powertools: logger, tracer, metrics, idempotency), least-privilege execution role per function, env vars via SSM/Secrets with caching, VPC only when needed (ENIs, cold start), reserved concurrency to protect downstreams, provisioned concurrency for latency-critical.
- IaC: SAM (`sam build/deploy`, `sam local invoke`), CDK, Serverless Framework, Terraform `aws_lambda_function` + `archive_file` / container image. Lambda versions + aliases + CodeDeploy canary (`Canary10Percent5Minutes`).
- Cost model: per request + GB-second; cheap at low/variable load; at steady high load containers win. Observe with Lambda Insights + CloudWatch Logs retention set.
- Common failure questions: throttling (account concurrency 1,000 default → request increase, reserved concurrency), timeouts (15 min max; increase or redesign to Step Functions), cold starts, SQS messages reappearing (visibility timeout < processing time), S3 → Lambda recursion loops (write to a different prefix/bucket), VPC Lambda can't reach internet (NAT) / AWS APIs (endpoints).
