---
title: "Application telemetry"
summary: "Metrics, traces, and logs that Backbone services export for operational troubleshooting — for architecture review and technical diligence."
---

Backbone instruments application services so operators can see **what the software is doing** — not only whether containers are running, but whether login, document processing, notifications, and data access are succeeding at acceptable rates.

This guide explains the **application telemetry** Backbone emits: metrics, traces, and structured logs. It complements [Infrastructure monitoring](/docs/monitoring), which covers CloudWatch dashboards and alarms, and [Observability architecture](/docs/observability), which explains how the operational concerns fit together.

## What problem this solves

Running services on ECS without application-level telemetry leaves blind spots:

1. **Health checks say “up” but users see errors** — the process is alive while business operations fail.
2. **Capacity planning is guesswork** — CPU alone does not reveal hot endpoints or dependency slowdowns.
3. **Abuse and overload are hard to detect early** — rate-limit and error spikes need dedicated signals.

Backbone records metrics at the HTTP layer, on domain operations, on data access, on circuit breakers, and on rate limiting so operators can build meaningful dashboards and alarms.

## Architectural overview

```
  Quarkus service (ECS Fargate)
        │
        ├─ HTTP requests (latency, status codes)
        ├─ Domain operations (login, register, document parse, …)
        ├─ Data access (database, object store)
        ├─ Circuit breakers (open / reject counts)
        └─ Rate limiting (requests, violations, utilization)
                │
                ├─ Metrics ──► AMP (STAGE / PROD)  ──► AMG dashboards
                ├─ Traces  ──► X-Ray (STAGE / PROD) ──► AMG trace views
                └─ Logs    ──► CloudWatch Logs

  Platform infrastructure (ALB, RDS, WAF, …)
        │
        └─ CloudWatch metrics + SNS alarms
```

Each service is instrumented with Micrometer and OpenTelemetry. In deployed STAGE and PROD environments, metrics and traces export in-process from the application task.

## What is measured

| Layer                 | Typical signals                       | Why it matters                                        |
|-----------------------|---------------------------------------|-------------------------------------------------------|
| **HTTP**              | Request counts, latency, status codes | End-user experience and error rates                   |
| **Domain operations** | Success/failure per business action   | Detect failing login, registration, or document flows |
| **Data access**       | Query timing, pool usage              | Catch database or storage pressure early              |
| **Circuit breakers**  | Open state, rejections                | Spot dependency outages before user-visible cascades  |
| **Rate limiting**     | Requests, violations, utilization     | Abuse detection and fair-use enforcement              |

Metrics are consistent across services so dashboards can compare behaviour between auth, actor, document, notification, and audit workloads.

## Environments

| Environment     | Application metrics       | Traces        | Dashboards             |
|-----------------|---------------------------|---------------|------------------------|
| **Local dev**   | Prometheus (Compose)      | Jaeger        | Grafana (Compose)      |
| **INT**         | Exporters off             | Exporters off | Not deployed           |
| **STAGE / PROD** | Amazon Managed Prometheus | AWS X-Ray     | Amazon Managed Grafana |

INT deliberately does not deploy the managed observability stack or enable exporters — integration testing does not require production observability infrastructure.

## Bundled Grafana dashboards

| Dashboard                | Role                                                                 |
|--------------------------|----------------------------------------------------------------------|
| Platform Overview        | Golden signals: rate, 5xx %, p95, throttle violations                |
| Quarkus HTTP             | Request rate, latency, status (service filter)                       |
| Domain Operations        | Auth, actor, document domain counters                                |
| Rate Limiting            | Throttle / rate-limit views                                          |
| Database                 | DB ops and Agroal pool (by service)                                  |
| Redis & Circuit Breakers | Redis client metrics, circuit breakers                               |
| Notification Delivery    | Delivery by channel, provider, reason, failure ratio                 |
| Audit Delivery           | Ingest/dispatch lifecycle by event type, severity, reason            |
| Platform Ops             | ALB / ECS / RDS CloudWatch SEARCH + thin X-Ray Trace Statistics (AMG) |
| Process Runtime          | Process/CPU/uptime for native ECS (primary in STAGE/PROD)             |
| Quarkus JVM              | Heap/GC for local `quarkus:dev` and `BACKBONE_ECS_RUNTIME_MODE=jvm`  |

Infrastructure alarms (load balancers, databases, WAF, and similar) use **CloudWatch and SNS**, independent of AMP. See [Infrastructure monitoring](/docs/monitoring).

## Security and access

- Metrics endpoints are **not protected by user JWT** in the default configuration — they are intended for infrastructure scraping, not browser access.
- In deployed environments, tasks run in **private subnets**. Only load balancers and explicitly allowed network paths should reach application ports.
- Protecting scrape targets (security groups, internal-only access) is an **operator responsibility**.

Edge WAF rules do not replace network-level protection of metrics endpoints.

## Operator expectations

| Topic                 | Expectation                                                                                      |
|-----------------------|--------------------------------------------------------------------------------------------------|
| **Dashboards**        | Use HTTP + domain + dependency metrics together; health alone is insufficient                    |
| **Alerting**          | Rate-limit violations, circuit breaker opens, and elevated 5xx rates are strong alarm candidates |
| **Traces**            | Cross-service flows visible in X-Ray via AMG; correlate logs by trace ID in CloudWatch Logs      |
| **Capacity**          | Combine metrics with load testing envelopes documented under [Performance](/docs/performance)    |

## Further reading

- [Observability architecture](/docs/observability) — four-concern model
- [Infrastructure monitoring](/docs/monitoring) — CloudWatch dashboards and alarms
- [ADR-0026: Observability backend strategy](/docs/0026-observability-backend-strategy)
- [What is Amazon Managed Service for Prometheus?](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html) — AWS Documentation
