---
title: "Health checks"
summary: "How Backbone signals readiness to load balancers and operators when dependencies are healthy."
---

Container orchestration needs more than “the JVM started.” Backbone services report **readiness** — whether the task can safely receive traffic given the state of databases, identity providers, object storage, and other dependencies.

This guide is for architecture review, procurement, and operations diligence.

## What problem this solves

Without dependency-aware health checks, load balancers may route traffic to tasks that:

1. **Cannot reach PostgreSQL or DynamoDB** — every request fails until the task is replaced.
2. **Cannot validate tokens against Cognito** — authentication appears broken cluster-wide.
3. **Cannot enforce rate limits when Redis is required** — abuse controls silently weaken (rate limiting uses a fail-closed readiness probe; see [Rate limiting](/docs/rate-limiting)).

Backbone readiness probes give the load balancer a accurate signal: **ready to serve** vs **remove from rotation**.

## How health checks work

```
  Application Load Balancer
           │
           │  GET /q/health/ready  (every ~30s)
           ▼
      ECS task (service)
           │
           ├─ Process alive?        → liveness (/q/health/live)
           └─ Dependencies OK?    → readiness (/q/health/ready)
                    │
                    ├─ PostgreSQL (where used)
                    ├─ DynamoDB / S3 (where used)
                    ├─ Cognito pools (auth-service)
                    ├─ Redis (when rate-limit store = redis)
                    └─ Email provider reachability (notification-service)
```

**Liveness** confirms the process should not be restarted. **Readiness** confirms the task should receive traffic. ECS target groups use **readiness** as the health check path so unhealthy tasks are drained automatically.

## Per-service readiness scope

| Service                  | Readiness verifies (high level)                                    |
|--------------------------|--------------------------------------------------------------------|
| **auth-service**         | Human and service Cognito user pools reachable                     |
| **actor-service**        | Actor profile database reachable                                   |
| **document-service**     | Document object storage and metadata store reachable               |
| **notification-service** | Notification database, template store, and outbound email provider |
| **audit-service**        | Audit event database reachable                                     |
| **actor-bff**            | Core platform dependencies required for browser-facing aggregation |

Exact checks evolve with each service’s datastore footprint; the pattern is consistent: **prove critical dependencies before accepting traffic**.

## Fail-open vs fail-closed

| Concern                       | Readiness behaviour                                                                               |
|-------------------------------|---------------------------------------------------------------------------------------------------|
| **Application cache (Redis)** | Fail-open — cache unavailability does not fail readiness                                          |
| **Rate limiting (Redis)**     | Fail-closed — tasks go not-ready if Redis is unreachable when distributed rate limits are enabled |

This matches the security posture described in [Application caching and distributed scale](/docs/caching) and [Rate limiting](/docs/rate-limiting).

## Operator expectations

| Topic                  | Expectation                                                                                         |
|------------------------|-----------------------------------------------------------------------------------------------------|
| **Deploys**            | New tasks must pass readiness before old tasks drain; failed readiness blocks rollout progress      |
| **Dependency outages** | A datastore or Cognito incident removes affected tasks from rotation rather than serving errors     |
| **Monitoring**         | Alert on sustained unhealthy target counts in the load balancer                                     |
| **Local dev**          | Health endpoints are available in Quarkus dev mode; dependency checks reflect local or test doubles |

## Further reading

- [Observability architecture](/docs/observability) — readiness is one signal among several operational concerns
- [Infrastructure monitoring](/docs/monitoring) — alert on sustained unhealthy target counts
- [Application telemetry](/docs/application-telemetry) — application health signals beyond load-balancer probes
- [Determine Amazon ECS task health using container health checks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/healthcheck.html) — AWS Documentation
