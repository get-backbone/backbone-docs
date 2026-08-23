---
title: "0029. Customer-managed KMS per domain (opt-in, greenfield, BYOK)"
summary: "Opt-in customer-managed KMS for application data, secrets, messaging, static edge, and governance evidence — with optional BYOK and documented AWS-managed exceptions."
---

**Status:** Accepted  
**Date:** 2026-08-22  
**Related:** [0024. Internal ALB TLS (east-west) optional](/docs/0024-internal-alb-tls-east-west-optional), [0027. Governance evidence architecture](/docs/0027-governance-evidence-architecture), [0028. Governance evidence Object Lock retention](/docs/0028-governance-evidence-object-lock)

## Context

By default, Backbone encrypts application data at rest with **AWS-managed** keys (databases, caches, object storage, queues, and secrets). That baseline is appropriate for many deployments.

Some operators need **customer-managed KMS keys (CMKs)** for key-policy control, rotation evidence, or **bring-your-own-key (BYOK)** from an organizational landing zone. Common drivers include contractual key-custody requirements and regulated workloads.

SOC 2, HIPAA, and GDPR require **encryption and sound key management** for sensitive data; they do **not** mandate customer-managed keys by name. On AWS, CMKs are the usual mechanism when the operator must own key policy and custody.

## Decision

1. **Customer-managed KMS is opt-in** via platform configuration (`customerManagedKms`), default **off**:
   - **Off** — AWS-managed encryption (unchanged baseline)
   - **On** — the platform provisions one CMK per encryption domain
   - **On with optional key ARNs** — BYOK where ARNs are supplied; the platform provisions any domain without an ARN

2. **One CMK per domain** — secrets, datastore, messaging, static edge, and governance — so compromise or rotation of one key does not span unrelated surfaces. The same BYOK ARN may be reused across domains when an organization prefers a single landing-zone key.

3. **Keys are provisioned (or imported) centrally**; each workload attaches encryption and access for the resources it owns. Operators enabling BYOK remain responsible for key policies that allow the platform account and services to use those keys.

4. **Bootstrap decision** — choose the posture before first deploy of an environment. There is no automated path to re-encrypt existing data after switching modes.

5. **Governance evidence** follows the same flag: with customer-managed KMS enabled, evidence storage and CloudTrail use the governance-domain CMK; with it disabled, the evidence bucket uses **SSE-S3**. Object Lock and related governance controls are unchanged. This updates the always-SSE-KMS evidence posture described in [ADR-0027](/docs/0027-governance-evidence-architecture); that ADR remains the historical record of the earlier baseline.

## Documented exceptions

| Area                                                              | Stance                                        |
|-------------------------------------------------------------------|-----------------------------------------------|
| Load balancer and CloudFront / S3 **access-log delivery** buckets | AWS constraint — SSE-S3 only                  |
| CloudWatch Logs, VPC flow logs                                    | Platform stance — AWS-managed                 |
| Managed Prometheus / Grafana                                      | Platform stance — AWS-managed                 |
| Cognito user pools                                                | Platform stance — identity plane; AWS-managed |

Operators who require CMKs on those surfaces can implement as required.

## Consequences

- Operators can enable platform CMKs or BYOK for key custody on in-scope data stores.
- Domain boundaries make blast radius and ownership clearer; shared BYOK ARNs remain an explicit organizational choice.
- The choice must be made before first deploy of an environment; flipping later implies a migration the platform does not automate.
- BYOK operators must grant the deploy account and integrated AWS services appropriate use of their keys (including CloudTrail when governance evidence is enabled).
- Provisioned CMKs incur KMS key and API charges.

## Related

- [Platform security posture](/docs/security)
- [Compliance](/docs/compliance)
- [Operations](/docs/operations)
