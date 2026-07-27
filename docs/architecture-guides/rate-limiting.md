---
title: "Rate limiting"
summary: "How Backbone protects public APIs from abuse and accidental overload at the edge and inside application services."
---

Backbone applies rate limiting at **multiple layers**. Edge controls absorb broad traffic spikes and scanning noise; application-level limits enforce fair use per user, per service identity, and per client IP. Together they protect login flows, authenticated APIs, and backend capacity without requiring operators to build a separate abuse-prevention stack.

This guide is for architecture review, procurement, and security diligence. Outbound provider rate limiting for the notification service (SES, and similar) is a separate concern — see [Notifications](/docs/notifications) and [ADR-0016](/docs/0016-notification-service-rate-limiting-strategy).

## What problem this solves

Public APIs face two distinct risks:

1. **Abuse** — Credential stuffing, scraping, or deliberate flooding of login and unauthenticated endpoints.
2. **Accidental overload** — A single integration, user, or buggy client sending more traffic than the platform should accept.

Without distributed counters, scaling out ECS tasks **weakens** per-instance limits: each task maintains its own budget, so abusive traffic can multiply with replica count. Backbone stores application rate-limit state in a **shared Redis cluster** (Amazon ElastiCache in deployed environments) so limits apply consistently across all tasks.

## Layered model

```
  Internet
      │
      ▼
┌─────────────────┐
│ CloudFront WAF  │  ← IP-based edge limit (coarse, all viewer traffic)
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│  Internet-facing ALB │
└────────┬─────────────┘
         │
         ▼
┌─────────────────┐
│ Application     │  ← Per user / service / IP (fine-grained, before auth)
│ rate limit      │
└────────┬────────┘
         │
         ▼
   Business logic
```

Each layer has a different scope and purpose. Edge limits are not a substitute for application limits, and application limits do not replace edge protection.

## Edge rate limiting (CloudFront WAF)

Viewer traffic passes through an AWS WAF WebACL attached to CloudFront. Among other rules (hostname allowlist, geo controls), a **rate-based rule** caps requests per source IP at the edge.

| Characteristic  | Behaviour                                                               |
|-----------------|-------------------------------------------------------------------------|
| **Scope**       | All requests to the public hostname, before they reach application code |
| **Granularity** | Per IP address                                                          |
| **Intent**      | Block obvious flooding and reduce load on origin infrastructure         |
| **Visibility**  | WAF metrics in CloudWatch                                               |

Edge limits are intentionally coarse. They protect the platform perimeter; they do not distinguish authenticated users or service callers.

Details of the edge and origin protection model are in [Platform security posture](/docs/security) and [ADR-0022](/docs/0022-public-alb-edge-and-origin-protection).

## Application rate limiting

Every service that includes the platform security stack can enforce **incoming HTTP rate limits** before authentication and routing complete. The filter runs early in the request pipeline so that login, registration, and other public endpoints are protected even when no valid token is present.

### How callers are identified

| Traffic type           | Limit key                               | Typical use                       |
|------------------------|-----------------------------------------|-----------------------------------|
| **Authenticated user** | User identity from the JWT payload      | Fair use per human actor          |
| **Service caller**     | Service identity from the service token | Fair use per workload             |
| **Unauthenticated**    | Client IP address                       | Protect public and auth endpoints |

The filter extracts identifiers from request headers where possible without requiring a fully validated token, so invalid or expired tokens still contribute to the correct user or service bucket.

### Default capacity (baseline deploy)

Operators can tune these values per environment. The baseline platform configuration uses token-bucket limits approximately as follows:

| Caller type         | Steady capacity         | Refill rate          |
|---------------------|-------------------------|----------------------|
| **Authenticated**   | 600 requests per minute | 10 tokens per second |
| **Unauthenticated** | 100 requests per minute | 2 tokens per second  |

Integration environments may use higher limits to support performance testing. Automated tests use an in-memory store so CI does not depend on Redis.

### Client-visible behaviour

When a caller exceeds its limit, the service responds with **HTTP 429 Too Many Requests** and standard rate-limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` when applicable). This gives well-behaved clients a clear signal to back off.

### Distributed enforcement

Rate-limit counters live in the same ElastiCache Redis cluster used for application caching. All ECS tasks read and update shared counters, so adding replicas does not multiply allowed throughput for a single abuser.

See [Application caching and distributed scale](/docs/caching) for the shared Redis architecture.

### Availability posture (fail-closed)

Rate limiting is treated as a **protection mechanism**, not an optional optimization. If Redis is unreachable while the Redis-backed store is configured, affected services fail the **readiness** probe (`rate-limit-redis`). The load balancer stops routing new traffic to unhealthy tasks until Redis recovers.

This is deliberate: silently disabling limits under load would weaken abuse controls at exactly the wrong time.

Application caching, by contrast, is fail-open. The two layers have different availability trade-offs by design.

## Observability

Rate-limit decisions emit metrics suitable for dashboards and alerting — request counts, violations, utilization, and failures. Operators can answer questions such as which identities or IP ranges are hitting limits and whether limit enforcement itself is failing.

Local development includes Grafana dashboards for throttle metrics; production export uses AMP when observability is enabled. See [Application telemetry](/docs/application-telemetry).

## Security and compliance considerations

- **Defence in depth** — Edge WAF limits, application limits, and authentication/authorization are independent controls.
- **Login protection** — Limits apply to authentication endpoints, reducing brute-force and credential-stuffing effectiveness.
- **Encryption in transit** — Redis connections use TLS in integration and production environments.
- **Network isolation** — ElastiCache is not internet-facing.

## Operator expectations

| Topic                  | Expectation                                                                                         |
|------------------------|-----------------------------------------------------------------------------------------------------|
| **Tuning**             | Adjust capacity and refill rates per environment; document changes for your security review.        |
| **Scaling**            | Horizontal scale does not require rate-limit reconfiguration; shared Redis coordinates counters.    |
| **Failure modes**      | Redis outage for rate limiting removes tasks from rotation until healthy; edge WAF limits remain.   |
| **Outbound email/SMS** | Notification delivery uses separate per-provider throttling — not covered by this HTTP limit stack. |

## Further reading

- [Application caching and distributed scale](/docs/caching) — shared Redis infrastructure
- [ADR-0016: Notification service rate limiting](/docs/0016-notification-service-rate-limiting-strategy) — outbound provider limits (distinct from HTTP ingress)
- [AWS WAF rate-based rule statements](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based-request-limiting.html) — AWS Documentation
