---
title: "0028. Governance evidence Object Lock retention"
summary: "CloudTrail evidence buckets use S3 Object Lock in COMPLIANCE mode with stage-aware default retention periods; this follow-up to ADR-0027 closes the same-account immutable archive gap without changing the governance evidence boundary."
---

**Status:** Accepted  
**Date:** 2026-08-21  
**Related:** [0027. Governance evidence architecture](/docs/0027-governance-evidence-architecture)

## **Context**

[ADR-0027](/docs/0027-governance-evidence-architecture) established governance evidence as a distinct concern and scoped `BackboneGovernanceStack` to CloudTrail, ALB access logs, and protected evidence storage (KMS, versioning, lifecycle).

Diligence reviewers still expect a **WORM-style** archive: objects that cannot be deleted or overwritten for a defined retention window. Versioning and encryption alone do not satisfy that ask. Application audit events in PostgreSQL are a different stream and are not this control.

## **Decision**

When `governanceEvidenceEnabled` is true, the **CloudTrail evidence bucket**:

- Enables **S3 Object Lock** with **COMPLIANCE** default retention in **every** stage (same behaviour)
- Uses a **stage-aware retention period** only:
  - Non-production: **1 day** (aligned with non-prod evidence lifecycle expiry)
  - Production: **365 days**
- Disables CDK `autoDeleteObjects` on that bucket (COMPLIANCE cannot be bypassed by the emptier)

The ALB access-logs companion bucket remains SSE-S3 without Object Lock (ALB cannot deliver to KMS destinations; locking that sink is deferred).

Dedicated security / log-archive accounts, counsel-driven legal hold workflows, and SIEM routing remain **operator-owned** extensions.

## **Consequences**

### Positive

- Same-account immutable archive is an automated baseline capability for CloudTrail evidence
- COMPLIANCE mode matches common reviewer expectations for non-bypassable retention
- Stage-aware duration keeps non-prod teardown possible after short expiry without changing lock mode

### Negative

- Locked object versions cannot be force-deleted before retention expires (including by account root)
- Forced non-prod `GovernanceStack` destroy may fail until retention expires if evidence objects exist
- Object Lock is sticky for the bucket lifetime; changing mode later requires bucket replacement

## **Alternatives considered**

### GOVERNANCE mode (bypassable)

Rejected for baseline. Useful for experiments, but reviewers asking for immutable archive typically expect COMPLIANCE. Bypass permissions would weaken the diligence claim.

### Object Lock only in production

Rejected. Behaviour would differ by stage; the chosen approach keeps COMPLIANCE everywhere and varies only duration.

### Object Lock on the ALB access-logs bucket

Deferred. ALB delivery constraints and log volume make this a separate decision.

### Cross-account log archive as baseline

Rejected for Backbone Core. Landing-zone / security OU patterns stay operator extensions per ADR-0027.

## **References**

- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Compliance](/docs/compliance)
- [Platform features — Licence tiers](/docs/features#licence-tiers-and-platform-configuration)
