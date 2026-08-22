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
  Audit dispatch  ──SQS ──▶  audit-service consumer  ──▶  PostgreSQL
        │
        └── EventBridge (roadmap) ──▶  central consumer / SIEM
```

**Today:** emitting services queue events on a dedicated **SQS** queue. **audit-service** consumes the queue and stores events in PostgreSQL (append-only).

**Roadmap:** the same interceptor and publisher can also fan out to **Amazon EventBridge** for SIEM / multi-account central logging. SQS covers delivery guarantees for application audit ingest.

## What gets audited

Typical audited actions include:

| Area               | Examples                                                  |
|--------------------|-----------------------------------------------------------|
| **Authentication** | Login, registration                                       |
| **Notifications**  | Welcome email, password reset                             |
| **Data access**    | Sensitive reads or changes (as annotated by each service) |

Events carry **event type**, **severity**, **actor identity** (resolved from request context or auth results), and **message templates** with argument paths for structured detail.

## Security model

- **Caller authorization** — Emitters need IAM permission to send to the audit queue; arbitrary clients cannot append to the audit log via the public API surface.
- **Structured records** — Events are schema-shaped, not free-text log lines, supporting downstream compliance tooling.
- **PII awareness** — Operators should review which fields are captured in event metadata; sensitive values should be masked or omitted at emission time.
- **Async delivery** — Dispatch is designed not to block user-facing latency; failed sends and invalid messages (DLQ) are visible for operator investigation.

## Compliance relevance

Audit logging supports common diligence questions:

- Can you show **who authenticated** and when?
- Can you trace **security-relevant actions** across services?
- Is there a **central store** queryable for investigations?

Backbone provides the **technical baseline**; retention policies, access controls on audit data, and legal hold processes remain **operator responsibilities**. See [Compliance](/docs/compliance) for control mapping.

## Operator expectations

| Topic              | Expectation                                                                       |
|--------------------|-----------------------------------------------------------------------------------|
| **Retention**      | Define Postgres backup and retention to match your framework (SOC 2, HIPAA, etc.) |
| **Centralization** | Plan EventBridge or SIEM fan-out when multi-account logging is required           |

## Further reading

- [Observability architecture](/docs/observability) — how application audit differs from CloudTrail and operational telemetry
- [Security best practices for AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html) — AWS Documentation
- [AWS centralized audit alerting](https://awsfundamentals.com/blog/build-centralized-alerting-across-your-organization-with-cloudwatch-eventbridge-lambda-and-cdk) — AWS Fundamentals
