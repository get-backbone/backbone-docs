---
title: "Application caching and distributed scale"
summary: "How Backbone keeps performance and consistency when multiple service instances run behind a load balancer."
---

Backbone runs containerized services on AWS behind load balancers. When more than one instance serves traffic at the same time, **in-memory caches and rate limits on a single instance are not enough** — each instance would maintain its own counters and cache entries, leading to inconsistent limits and duplicated work.

Backbone addresses this with a **shared Redis layer** (Amazon ElastiCache in deployed environments) used for application caching and distributed rate limiting. Together, these give you horizontal scale without sacrificing predictable behaviour at the edge.

This guide is for architecture review, procurement, and security diligence. Implementation detail lives in [ADR-0014: Application caching strategy](/docs/0014-application-caching-strategy).

## What problem this solves

Without a shared layer, scaling out ECS tasks creates two practical risks:

1. **Cache inconsistency** — The same user or document might be fetched and validated repeatedly across instances, increasing latency and load on databases and identity providers.
2. **Rate limit bypass** — Per-instance counters allow abusive or accidental traffic to exceed intended limits simply by spreading requests across tasks.

Backbone's distributed caching and rate limiting are designed so **all instances share the same view** of cached data and consumption counters.

## Architectural overview

```
                    ┌─────────────────────────────────────┐
                    │   CloudFront / ALB (multi-task)     │
                    └─────────────────┬───────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
        ┌─────▼─────┐           ┌─────▼─────┐           ┌─────▼─────┐
        │ ECS task  │           │ ECS task  │           │ ECS task  │
        │ (service) │           │ (service) │           │ (service) │
        └─────┬─────┘           └─────┬─────┘           └─────┬─────┘
              │                       │                       │
              └───────────────────────┼───────────────────────┘
                                      │
                              ┌───────▼────────┐
                              │ ElastiCache    │
                              │ (Redis)        │
                              │                │
                              │ • cache keys   │
                              │ • rate limits  │
                              └────────────────┘
```

Redis is provisioned as part of the platform datastore stack alongside PostgreSQL, DynamoDB, and S3. A **single cluster** serves both caching and rate limiting to keep operational overhead and cost predictable.

## Application caching

### Purpose

Application caching reduces repeated work on hot paths — especially token validation on authenticated requests and frequently read domain data — while keeping the database as the source of truth.

### What is cached

| Area                 | Typical benefit                                                          |
|----------------------|--------------------------------------------------------------------------|
| **Token validation** | Avoids re-validating the same JWT on every request across all instances. |
| **Actor profiles**   | Speeds up profile reads; entries are refreshed when profiles change.     |
| **Parsed documents** | Avoids re-reading and re-parsing unchanged document content.             |
| **Service tokens**   | Reduces overhead when services authenticate to each other.               |

Caching uses a **cache-aside** pattern: reads populate the cache; writes invalidate stale entries so clients see updated data on the next read.

### Availability posture (fail-open)

If Redis is temporarily unavailable, **requests continue**. Cache misses fall through to the underlying store or validation path. Cache unavailability does not block login or API traffic.

This trades a short period of higher latency and load for continued availability — appropriate for a performance optimization layer.

## Distributed rate limiting

Incoming HTTP traffic is rate-limited per authenticated user, per service identity, per client IP, and for unauthenticated callers. Limits are enforced **before** business logic runs.

Counters are stored in the same Redis cluster so limits apply **across all ECS tasks**, not per instance.

### Availability posture (fail-closed)

If Redis is unavailable for rate limiting, affected services report **unready** on the readiness probe. The load balancer stops sending new traffic to those tasks until Redis recovers.

This is intentional: rate limiting is an abuse-control mechanism; silently disabling it under load would weaken protection.

Edge-level rate limiting (CloudFront WAF) remains in place independently; see [Rate limiting](/docs/rate-limiting) and [Platform security posture](/docs/security).

## Security and compliance considerations

- **Encryption in transit** — Redis connections use TLS in integration and production environments.
- **Network isolation** — ElastiCache runs in private subnets; it is not internet-facing.
- **Sensitive data** — Token cache keys use hashes rather than storing full bearer tokens in Redis key names.
- **Shared infrastructure** — Operators manage one Redis cluster for both caching and rate limiting; capacity planning should account for combined use.

## Operator expectations

| Topic                 | Expectation                                                                                                                      |
|-----------------------|----------------------------------------------------------------------------------------------------------------------------------|
| **Scaling**           | Adding ECS tasks does not require cache or rate-limit configuration changes; shared Redis coordinates behaviour.                 |
| **Monitoring**        | Cache and throttle metrics are exported for observability dashboards (see [Application telemetry](/docs/application-telemetry)). |
| **Failure modes**     | Cache degradation increases backend load; rate-limit Redis failure removes tasks from rotation until healthy.                    |
| **Local development** | Developers run a local Redis instance for prod-like behaviour; automated tests use in-memory alternatives for speed.             |

## Further reading

- [ADR-0014: Application caching strategy](/docs/0014-application-caching-strategy)
- [Rate limiting](/docs/rate-limiting) — shared Redis layer for distributed limits
- [Best practices: Valkey and Redis OSS caches](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/BestPractices.html) — AWS Documentation
