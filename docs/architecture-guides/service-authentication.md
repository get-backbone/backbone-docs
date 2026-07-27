---
title: "Service authentication"
summary: "How backend services authenticate to each other and propagate user context without trusting the network alone."
---

Every backend service runs with its own **service identity**. When one service calls another, it presents a **service JWT** that identifies the caller. When the call chain started from a logged-in user, **user context** can travel alongside service identity on downstream requests.

Receiving services validate tokens on every call and can restrict sensitive operations to named callers.

This guide is for architecture review, procurement, and security diligence. See [ADR-0011](/docs/0011-stateless-jwt-authentication) and [ADR-0005](/docs/0005-implement-service-auth-sts-assumerolewithwebidentity) for technical records.

## What problem this solves

Internal network location is not proof of trust. In a microservice deployment:

1. **Any compromised task could call any endpoint** if only IP rules protected APIs.
2. **Background jobs** (scheduled tasks, async processing) need identity without a human session.
3. **Sensitive operations** (audit ingest, admin actions) should accept calls only from known services.

Backbone treats **service identity** as a first-class JWT trust domain, separate from user tokens.

## What's in place

| Capability                     | Description                                                                                                     |
|--------------------------------|-----------------------------------------------------------------------------------------------------------------|
| **Dedicated service accounts** | One Cognito service-pool identity per ECS backend service                                                       |
| **Credential storage**         | Per-service secrets in AWS Secrets Manager, provisioned and rotated by infrastructure                           |
| **Automatic token lifecycle**  | Services acquire, cache, and refresh service JWTs without manual intervention                                   |
| **Caller allowlists**          | Sensitive endpoints accept only named callers (e.g. audit ingest limited to auth and notification services)     |
| **Dual context propagation**   | User-initiated chains carry both service and user identity; system-initiated chains carry service identity only |

## How internal calls work

```
  actor-bff (user request)
        │
        │  service JWT + user JWT
        ▼
  auth-service
        │
        │  service JWT (+ user context when applicable)
        ▼
  actor-service / notification-service / …
        │
        ▼
  Receiver validates JWT, checks caller allowlist
```

**User-initiated flows** through the BFF present both identities on downstream calls. **Scheduled tasks and system flows** present service identity only.

Bootstrap endpoints (login, registration, token refresh) do not forward identity — credential exchange stays separate from authenticated call chains.

## Zero-trust alignment

1. **Verify explicitly** — Protected service endpoints require a valid JWT; service-only routes reject user-only tokens.
2. **Least privilege** — Caller allowlists name the only services permitted for sensitive operations.
3. **Assume breach** — Stolen user credentials alone cannot satisfy service-only endpoints; service credentials are a separate trust domain.

Transport between services today uses HTTP on the internal load balancer with **application-layer JWT validation**. Optional internal TLS is documented in [ADR-0024](/docs/0024-internal-alb-tls-east-west-optional).

## Relationship to user authentication

| Identity type   | Used for                 | Issued by               |
|-----------------|--------------------------|-------------------------|
| **User JWT**    | End-user API access      | Cognito actor user pool |
| **Service JWT** | Service-to-service calls | Cognito service pool    |

These are **separate trust domains**. See [User authentication](/docs/user-authentication) for the human identity model.

## Operator expectations

| Topic                | Expectation                                                                       |
|----------------------|-----------------------------------------------------------------------------------|
| **Deploy**           | Service accounts and secrets are created automatically when runtime stacks deploy |
| **Rotation**         | Infrastructure manages password rotation for service pool credentials             |
| **New services**     | Adding a deployable ECS service includes provisioning its service account         |
| **Future hardening** | mTLS and service mesh identity are optional enhancements beyond the baseline      |

## Further reading

- [User authentication](/docs/user-authentication) — human identity (separate trust domain)
- [ADR-0005: Service auth with STS AssumeRoleWithWebIdentity](/docs/0005-implement-service-auth-sts-assumerolewithwebidentity)
- [IAM OIDC federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html) — AWS Documentation
