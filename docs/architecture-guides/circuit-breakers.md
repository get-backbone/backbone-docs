---
title: "Circuit breakers"
summary: "How Backbone prevents cascading failures when databases and other dependencies become slow or unavailable."
---

When a downstream dependency — a database, object store, or external API — starts failing or responding slowly, continuing to hammer it makes things worse: threads pile up, errors propagate to callers, and recovery takes longer. **Circuit breakers** detect unhealthy dependencies and **stop calling them temporarily**, giving the dependency time to recover while the service fails fast with a controlled error instead of hanging or retrying indefinitely.

Backbone implements circuit breakers on **data-access paths** inside domain services (persistence and storage operations), not on inbound HTTP traffic. This guide is for architecture review, procurement, and security diligence.

## What problem this solves

Without circuit breakers, a partial outage can cascade:

1. A database connection pool exhausts waiting on slow queries.
2. Every incoming request blocks or times out at unpredictable intervals.
3. Load increases on the already-struggling dependency, delaying recovery.

Circuit breakers break that loop. After enough failures in a sliding window, the breaker **opens**: subsequent calls fail immediately without hitting the dependency. After a cooldown period, a limited number of **trial** calls test whether the dependency has recovered.

## Where circuit breakers apply

Backbone uses the MicroProfile Fault Tolerance model (SmallRye implementation in Quarkus) on **repository and persistence operations** — the boundary between application logic and infrastructure.

```
  HTTP request
       │
       ▼
  Application service  (business rules)
       │
       ▼
  Repository layer     ← circuit breaker + timeout + retry
       │
       ▼
  PostgreSQL / DynamoDB / S3 / …
```

| Dependency type      | Example services           | Typical operations protected                  |
|----------------------|----------------------------|-----------------------------------------------|
| **PostgreSQL (RDS)** | Actor, notification, audit | Profile reads/writes, notification records    |
| **DynamoDB**         | Document, notification     | Document metadata, templates, delivery status |
| **S3**               | Document                   | Source document upload and storage            |

Services that do not touch shared datastores through instrumented repositories may not expose circuit breaker metrics; the pattern is applied where persistence latency or availability directly affects user-facing operations.

Circuit breakers are **not** a substitute for edge rate limiting or authentication. They protect **outbound** calls from services to infrastructure, not **inbound** API abuse. See [Rate limiting](/docs/rate-limiting) for ingress protection.

## How breakers behave

Each protected operation is wrapped with a consistent fault-tolerance stack:

| Mechanism           | Purpose                                                                      |
|---------------------|------------------------------------------------------------------------------|
| **Timeout**         | Cap how long a single call waits for the dependency                          |
| **Retry**           | Retry transient failures a small number of times with backoff                |
| **Circuit breaker** | Open the circuit when failure rate exceeds a threshold; fail fast while open |

When the circuit is **closed** (healthy), calls proceed normally through timeout and retry. When **open**, calls fail immediately without consuming connection pool slots or adding load to the failing dependency. After a configured delay, the breaker moves to **half-open** and allows probe requests to test recovery.

Default thresholds follow MicroProfile Fault Tolerance conventions (minimum request volume before tripping, failure ratio, and cooldown delay). Operators validating production behaviour should review environment-specific settings during load and failure testing.

## Relationship to retries and timeouts

Retries and circuit breakers work together but serve different goals:

- **Retry** handles **transient** blips — a single timeout or connection reset.
- **Circuit breaker** handles **sustained** degradation — when the dependency is genuinely unhealthy.

Combining both avoids giving up on a one-off glitch while still stopping a stampede during an outage.

## Observability

Circuit breaker state transitions and call outcomes are exported as metrics (open/closed/half-open counts, successes, failures, rejections). Local Grafana dashboards include infrastructure views for circuit breaker health alongside database and HTTP metrics.

In production, these metrics support alarms on sustained open circuits or elevated rejection rates — a signal that a datastore or integration needs attention before user-visible error rates spike.

## Availability and user impact

When a circuit opens, affected API operations return errors to callers rather than hanging. The platform favours **predictable failure** over **unbounded waiting**. Exact HTTP status and error bodies depend on the service and operation; clients and BFF layers should treat dependency failures as retryable only when appropriate (idempotent reads, not duplicate writes).

Circuit breakers do **not** automatically fail ECS readiness probes. Readiness reflects whether the service process and its configured dependencies are reachable; an open circuit on a hot path may still warrant operational alerting even while the task is considered ready.

## Operator expectations

| Topic          | Expectation                                                                                                |
|----------------|------------------------------------------------------------------------------------------------------------|
| **Scope**      | Protects persistence and storage calls inside services, not CloudFront or ALB traffic.                     |
| **Tuning**     | Thresholds and delays should be validated against realistic load and failure-injection tests.              |
| **Monitoring** | Watch breaker state metrics; sustained open circuits often precede user-visible outages.                   |
| **Recovery**   | Breakers recover automatically when the dependency heals; no manual reset is required in normal operation. |

## Further reading

- [Application telemetry](/docs/application-telemetry) — circuit breaker metrics and dashboards
- [Rate limiting](/docs/rate-limiting) — ingress protection (complementary, not overlapping)
- [ADR-0006: Third-party API wrapper](/docs/0006-internal-third-party-api-wrapper-standalone-service-vs-library)
- [Circuit breaker pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html) — AWS Prescriptive Guidance
