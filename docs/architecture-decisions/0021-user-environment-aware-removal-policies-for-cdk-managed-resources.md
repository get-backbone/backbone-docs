---
title: "0021. Environment-aware removal policies for CDK-managed resources"
summary: "AWS CDK resources support a RemovalPolicy that determines whether resources are deleted or retained when a stack is destroyed or a resource is…"
---

**Status:** Accepted  
**Date:** 2026-03-27  
**Context:** CDK `RemovalPolicy` must balance destroy vs retain by environment and resource type without orphans or data loss.

## Context

AWS CDK resources support a `RemovalPolicy` that determines whether resources are deleted or retained when a stack is destroyed or a resource is replaced.

A blanket policy (e.g. `RETAIN` for all production resources) can lead to:

* Orphaned infrastructure
* Increased operational overhead
* Naming conflicts on redeploy

Conversely, using `DESTROY` universally risks irreversible data loss for stateful resources.

## Decision

Adopt an **environment-sensitive and resource-type-aware removal policy strategy**:

### 1. Environment Rules

* **Non-production (dev/test):**

  * Default: `DESTROY` *
* **Production:**

  * Use `RETAIN` **only for stateful resources**
  * Use `DESTROY` for stateless infrastructure

### 2. Resource Classification

**Stateful (RETAIN in production):**

* Databases (RDS, DynamoDB)
* Object storage (S3)
* Identity systems (Cognito User Pools)
* Persistent volumes (EFS)
* Search/indexing (OpenSearch)
* Encryption keys (KMS)

**Stateless (DESTROY in production):**

* Compute (Lambda, ECS)
* API layers (API Gateway, ALB)
* IAM roles/policies
* Eventing (EventBridge)
* Networking components

### 3. Additional Safeguards

* Enable **CloudFormation termination protection** on production stacks
* Restrict deletion via IAM policies where appropriate

## Consequences

### Positive

* Protects critical production data from accidental deletion
* Keeps non-prod environments disposable and cost-efficient
* Avoids unnecessary retention of reproducible infrastructure

### Negative

* Retained resources become **orphaned** if stacks are deleted
* Manual cleanup or import may be required
* Requires discipline in correctly classifying resources

## Notes

Retention is a **data protection mechanism**, not a substitute for:

* Backups
* Disaster recovery
* Access control

Stack deletion in production should be considered an exceptional operation.

---
