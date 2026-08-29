---
title: "Notification delivery"
summary: "How Backbone sends transactional email with templates, queuing, delivery tracking, and unsubscribe handling."
---

Applications need reliable **transactional notifications** — welcome email, password reset, and future channels — without each service integrating directly with email providers or duplicating template logic.

Backbone provides a **centralized notification service** that accepts send requests, queues work, renders templates, delivers via AWS SES, and tracks outcomes.

This guide is for architecture review, procurement, and product diligence.

## What problem this solves

Without a shared notification layer:

1. **Every service talks to SES (or Twilio) directly** — duplicated credentials, templates, and retry logic.
2. **Callers block on delivery** — slow or failing email providers delay user-facing API responses.
3. **Bounces and complaints are invisible** — reputation and compliance suffer.

Backbone follows a **fire-and-forget** model: callers receive immediate acceptance with a notification ID; delivery happens asynchronously.

## Architectural overview

```
  Calling service (e.g. actor-service)
        │
        │  send welcome / password reset
        ▼
  notification-service API
        │
        ├─ Validate actor / template
        ├─ Persist queued notification (PostgreSQL)
        └─ Return 201 + notificationId  ──▶  caller continues
                │
                ▼ (async processing)
        Template render (Qute) → AWS SES send
                │
                ▼
        SNS webhooks (bounce / complaint / delivery)
                │
                ▼
        Update delivery status in database
```

Callers **do not wait** for SES delivery. They receive confirmation that the notification was **accepted and queued**.

## Capabilities

| Capability                  | Description                                                                                      |
|-----------------------------|--------------------------------------------------------------------------------------------------|
| **Transactional email**     | Welcome and password-reset flows today; extensible template model                                |
| **Template engine**         | Versioned templates with variable substitution                                                   |
| **Priority queues**         | Critical messages can be prioritized over bulk traffic                                           |
| **Delivery tracking**       | Status transitions: queued → sent → delivered / bounced / complaint                              |
| **Unsubscribe tokens**      | Secure, single-use tokens for marketing/compliance opt-out paths                                 |
| **Provider rate awareness** | Per-provider throttling so SES limits are respected (distinct from HTTP rate limits at the edge) |

## Delivery lifecycle and webhooks

In deployed environments, **Amazon SES** sends delivery, bounce, and complaint events to an **SNS HTTPS endpoint** on the notification service. The service:

1. Verifies SNS message authenticity (signature validation).
2. Links provider message IDs back to queued notifications.
3. Updates delivery status idempotently.

This gives operators visibility into **deliverability** without polling provider APIs.

## Security and compliance

- **Service-to-service auth** — Only authorized platform services may trigger notifications.
- **Unsubscribe integrity** — Tokens are short-lived and single-use where applicable.
- **Audit trail** — Send actions can emit audit events.
- **Operator responsibilities** — SES domain verification, DKIM/SPF, bounce handling policies, and marketing consent rules remain with the deploying organization.

## Operator expectations

| Topic                   | Expectation                                                                                               |
|-------------------------|-----------------------------------------------------------------------------------------------------------|
| **SES setup**           | Domain identity and configuration set provisioned by platform CDK; verify sending limits for your account |
| **Webhooks**            | SNS subscription must confirm successfully after deploy                                                   |
| **Monitoring**          | Track bounce and complaint rates; sustained spikes may require list hygiene                               |
| **User-facing latency** | Callers should not depend on synchronous delivery; queue acceptance is the contract                       |

## Further reading

- [Service authentication](/docs/service-authentication) — service-to-service auth for notification triggers
- [Audit logging](/docs/audit) — audit events for send actions
- [ADR-0015: Fire-and-forget pattern](/docs/0015-notification-service-fire-and-forget-pattern)
- [ADR-0016: Provider rate limiting](/docs/0016-notification-service-rate-limiting-strategy)
- [ADR-0019: Notification unsubscribe token security](/docs/0019-notification-service-unsubscribe-token-security)
- [Amazon SES best practices](https://docs.aws.amazon.com/ses/latest/dg/best-practices.html) — AWS Documentation
