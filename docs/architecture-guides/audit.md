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

## EventBridge bus

Emitters publish once to a custom **EventBridge** bus (`audit-events-bus`). Routing is infrastructure: rules select targets. The **baseline** target is the existing **SQS** ingest queue (`backbone-audit-events`); **audit-service** consumes that queue into PostgreSQL.

**Why this shape**

- Compliance needs event-driven routing (fan-out to SIEM / multi-account) without changing every emitter.
- SQS remains the durable buffer and DLQ path for application audit ingest.
- Additional consumers (SIEM, log archive, partner destinations) attach as new EventBridge rules — Open/Closed at the infrastructure boundary.

HTTP and SQS dispatcher types remain for possible future use; the active producer is EventBridge only.

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
  Audit dispatch  ──EventBridge (audit-events-bus)──▶  rules
        │                                                 │
        │                                                 ├── audit-ingest → SQS → audit-service → PostgreSQL
        │                                                 └── (operator) SIEM / other targets
```

## What gets audited

Typical audited actions include:

| Area               | Examples                                                  |
|--------------------|-----------------------------------------------------------|
| **Authentication** | Login, registration                                       |
| **Notifications**  | Welcome email, password reset                             |
| **Data access**    | Sensitive reads or changes (as annotated by each service) |

Events carry **event type**, **severity**, **actor identity** (resolved from request context or auth results), and **message templates** with argument paths for structured detail.

## Security model

- **Caller authorization** — Emitters need IAM `events:PutEvents` on the audit bus; arbitrary clients cannot append via the public API surface.
- **Structured records** — Events are schema-shaped, not free-text log lines, supporting downstream compliance tooling.
- **PII awareness** — Operators should review which fields are captured in event metadata; sensitive values should be masked or omitted at emission time.
- **Async delivery** — Dispatch is designed not to block user-facing latency; failed puts and invalid queue messages (DLQ) are visible for operator investigation.

## Compliance relevance

Audit logging supports common diligence questions:

- Can you show **who authenticated** and when?
- Can you trace **security-relevant actions** across services?
- Is there a **central store** queryable for investigations?
- Can you **fan out** audit events to SIEM without rewriting emitters?

Backbone provides the **technical baseline**; retention policies, access controls on audit data, legal hold, and additional EventBridge targets remain **operator responsibilities**. See [Compliance](/docs/compliance) for control mapping.

## Operator expectations

| Topic              | Expectation                                                                       |
|--------------------|-----------------------------------------------------------------------------------|
| **Retention**      | Define Postgres backup and retention to match your framework (SOC 2, HIPAA, etc.) |
| **Centralization** | Attach SIEM / cross-account rules to `audit-events-bus` (exported ARN/name)       |

## Further reading

- [Observability architecture](/docs/observability) — how application audit differs from CloudTrail and operational telemetry
- [Security best practices for AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html) — AWS Documentation
- [AWS centralized audit alerting](https://awsfundamentals.com/blog/build-centralized-alerting-across-your-organization-with-cloudwatch-eventbridge-lambda-and-cdk) — AWS Fundamentals
