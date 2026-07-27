---
title: "Audit logging"
summary: "How Backbone captures security-relevant actions for compliance traceability without embedding logging in business code."
---

Regulated and security-conscious operators need a **durable trail** of who did what, when — login, registration, sensitive data access, and notification events — without scattering log statements through application code.

Backbone provides **cross-cutting audit logging**: services declare auditable actions declaratively; the platform captures structured events and persists them centrally.

This guide is for architecture review, procurement, and compliance diligence.

## What problem this solves

Ad-hoc application logging is insufficient for audit and compliance use cases:

1. **Inconsistent coverage** — Some teams log, others forget; formats differ.
2. **No central query surface** — Logs trapped per service are hard to correlate.
3. **Business logic pollution** — Audit concerns mixed into domain code.

Backbone separates **business behaviour** from **audit emission**. Annotated operations produce structured audit records with actor identity, event type, severity, and contextual metadata.

## Architectural overview

```
  Domain service (auth, notification, …)
        │
        │  auditable action completes
        ▼
  Audit interceptor (platform library)
        │
        │  structured AuditEventRequest
        ▼
  Audit dispatch  ──HTTP (today) ──▶  audit-service  ──▶  PostgreSQL
        │
        └── EventBridge (roadmap) ──▶  central consumer / SIEM
```

**Today:** emitting services send events to a dedicated **audit-service** over HTTP. Events are stored in PostgreSQL with a stable schema.

**Roadmap:** the same interceptor and publisher can dispatch to **Amazon EventBridge** for centralized ingestion in a security or logging account without changing business services.

## What gets audited

Typical audited actions include:

| Area               | Examples                                                  |
|--------------------|-----------------------------------------------------------|
| **Authentication** | Login, registration                                       |
| **Notifications**  | Welcome email, password reset                             |
| **Data access**    | Sensitive reads or changes (as annotated by each service) |

Events carry **event type**, **severity**, **actor identity** (resolved from request context or auth results), and **message templates** with argument paths for structured detail.

## Security model

- **Caller authorization** — Only designated platform services may ingest audit events; arbitrary clients cannot append to the audit log.
- **Structured records** — Events are schema-shaped, not free-text log lines, supporting downstream compliance tooling.
- **PII awareness** — Operators should review which fields are captured in event metadata; sensitive values should be masked or omitted at emission time.
- **Async delivery** — Dispatch is designed not to block user-facing latency; delivery failures are logged for operator investigation.

## Compliance relevance

Audit logging supports common diligence questions:

- Can you show **who authenticated** and when?
- Can you trace **security-relevant actions** across services?
- Is there a **central store** queryable for investigations?

Backbone provides the **technical baseline**; retention policies, access controls on audit data, and legal hold processes remain **operator responsibilities**. See [Compliance](/docs/compliance) for control mapping.

## Operator expectations

| Topic              | Expectation                                                                                 |
|--------------------|---------------------------------------------------------------------------------------------|
| **Availability**   | audit-service must be reachable from emitting services; failed dispatch should be monitored |
| **Retention**      | Define Postgres backup and retention to match your framework (SOC 2, HIPAA, etc.)           |
| **Centralization** | Plan EventBridge or SIEM integration when multi-account central logging is required         |
| **Local dev**      | Audit can be disabled or pointed at a local audit-service instance for development          |

## Further reading

- [Observability architecture](/docs/observability) — how application audit differs from CloudTrail and operational telemetry
- [Security best practices for AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html) — AWS Documentation
- [AWS centralized audit alerting](https://awsfundamentals.com/blog/build-centralized-alerting-across-your-organization-with-cloudwatch-eventbridge-lambda-and-cdk) — AWS Fundamentals
