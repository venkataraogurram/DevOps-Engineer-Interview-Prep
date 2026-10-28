# 9. Docker, microservices, serverless

[← Back to index](README.md)

**Q74. Dockerfile best practices?**
Pin the base image, use slim or distroless, multi-stage builds so build tools don't ship, order layers from least to most frequently changing for cache hits, `.dockerignore`, run as a non-root user, one process per container, exec-form ENTRYPOINT so PID 1 gets SIGTERM, no secrets in layers, add a HEALTHCHECK and OCI labels with the git revision, and scan the image in CI.

---

**Q75. ENTRYPOINT vs CMD?**
ENTRYPOINT is the fixed executable; CMD supplies default arguments that `docker run` can override. Use exec form, a JSON array, for both so signals reach the process directly instead of through a shell.

---

**Q76. What makes a good microservice boundary?**
A service owns its data and a business capability; it can be deployed independently; communication is through versioned APIs or events, not shared databases. Cross-cutting concerns such as auth, logging with correlation IDs, tracing, retries with backoff and circuit breakers, and timeouts are handled consistently, often by a gateway or a mesh. The honest trade-off: operational complexity and distributed debugging, so don't split a small team's monolith prematurely.

---

**Q77. When would you choose serverless over containers?**
Event-driven, spiky or low-volume workloads, glue between AWS services, scheduled jobs, and APIs where per-request pricing wins. Containers when you need long-running processes, steady high load where per-request pricing is more expensive, custom runtimes, or strict latency with no cold starts. Many systems mix both.

---

**Q78. SQS message processed twice. Why, and how do you handle it?**
Standard SQS is at-least-once; a message reappears if the consumer doesn't delete it before the visibility timeout expires, for example a Lambda that runs longer than the timeout. Set visibility timeout to at least six times the function timeout, make the handler idempotent with an idempotency key stored in DynamoDB, use a DLQ with a maxReceiveCount, and alarm on DLQ depth. FIFO queues add exactly-once processing within a deduplication window.

---

[← Back to index](README.md)
