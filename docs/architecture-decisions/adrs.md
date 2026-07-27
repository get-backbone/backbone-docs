---
title: "Architecture Decision Records (ADRs)"
summary: "This repository contains a public, sanitized subset of Architecture Decision Records (ADRs) from Backbone."
---

This repository contains a public, sanitized subset of Architecture Decision Records (ADRs) from Backbone.

ADRs document consequential engineering decisions—context, constraints, alternatives, and rationale—as a durable record of *why* the system is shaped as it is.

## Scope and redaction

Not all ADRs are published. The records included here:

- Represent decisions relevant to Backbone consumers
- Exclude sensitive details (e.g. security mechanisms, internal topology, proprietary workflows)
- May be abstracted or generalized from their original form

## Interpretation notes

- ADRs are **historical artifacts**, and are not always reflective of the current implementation
- Decisions may evolve; superseding ADRs will be linked where applicable
- Trade-offs and rejected alternatives are preserved to aid understanding

## ADR index

- [0001-record-architecture-decisions.md](/docs/0001-record-architecture-decisions)
- [0002-adoption-of-quarkus-for-api-and-controller-tier.md](/docs/0002-adoption-of-quarkus-for-api-and-controller-tier)
- [0003-authentication-and-user-management-approach.md](/docs/0003-authentication-and-user-management-approach)
- [0004-use-aws-cognito-across-all-environments.md](/docs/0004-use-aws-cognito-across-all-environments)
- [0005-implement-service-auth-sts-assumerolewithwebidentity.md](/docs/0005-implement-service-auth-sts-assumerolewithwebidentity)
- [0006-internal-third-party-api-wrapper-standalone-service-vs-library.md](/docs/0006-internal-third-party-api-wrapper-standalone-service-vs-library)
- [0007-observability-strategy.md](/docs/0007-observability-strategy)
- [0008-decoupling-micro-services-transitioning-from-rest-to-sqs.md](/docs/0008-decoupling-micro-services-transitioning-from-rest-to-sqs)
- [0009-user-profile-storage-strategy.md](/docs/0009-user-profile-storage-strategy)
- [0010-rest-api-design-standards.md](/docs/0010-rest-api-design-standards)
- [0011-stateless-jwt-authentication.md](/docs/0011-stateless-jwt-authentication)
- [0012-clean-architecture-package-structure.md](/docs/0012-clean-architecture-package-structure)
- [0013-migration-from-mapstruct-to-manual-mappers.md](/docs/0013-migration-from-mapstruct-to-manual-mappers)
- [0014-application-caching-strategy.md](/docs/0014-application-caching-strategy)
- [0015-notification-service-fire-and-forget-pattern.md](/docs/0015-notification-service-fire-and-forget-pattern)
- [0016-notification-service-rate-limiting-strategy.md](/docs/0016-notification-service-rate-limiting-strategy)
- [0017-notification-service-template-implementation.md](/docs/0017-notification-service-template-implementation)
- [0018-notification-service-template-engine.md](/docs/0018-notification-service-template-engine)
- [0019-notification-service-unsubscribe-token-security.md](/docs/0019-notification-service-unsubscribe-token-security)
- [0020-single-vpc-per-environment.md](/docs/0020-single-vpc-per-environment)
- [0021-user-environment-aware-removal-policies-for-cdk-managed-resources.md](/docs/0021-user-environment-aware-removal-policies-for-cdk-managed-resources)
- [0022-public-alb-edge-and-origin-protection.md](/docs/0022-public-alb-edge-and-origin-protection)
- [0023-dnssec-route53-hosted-zones-baseline.md](/docs/0023-dnssec-route53-hosted-zones-baseline)
- [0024-internal-alb-tls-east-west-optional.md](/docs/0024-internal-alb-tls-east-west-optional)
- [0025-static-ui-cloudfront-s3.md](/docs/0025-static-ui-cloudfront-s3)
- [0026-observability-backend-strategy.md](/docs/0026-observability-backend-strategy)
- [0027-governance-evidence-architecture.md](/docs/0027-governance-evidence-architecture)
