---
title: "User authentication"
summary: "How end users sign in, register, and access the platform with stateless JWTs through a browser-facing API."
---

Human users authenticate with **stateless JWTs** issued by Amazon Cognito. After sign-in or registration, clients store tokens locally and send them on each API request. There is **no server-side session store** — any service instance can validate tokens independently, which supports horizontal scaling behind CloudFront and load balancers.

This guide is for architecture review, procurement, and security diligence.

## What problem this solves

Session-based authentication ties users to specific server instances or shared session clusters. That complicates ECS scaling, increases operational overhead, and creates affinity assumptions at the load balancer.

Backbone’s model:

1. **Cognito** is the identity provider (user pools, password policies, token issuance).
2. **Clients hold tokens** — browsers use local storage; mobile clients use platform secure storage.
3. **Services validate on every request** — no sticky sessions required.

## What's in place

| Capability                    | Description                                                                                      |
|-------------------------------|--------------------------------------------------------------------------------------------------|
| **Email/password login**      | First-party UI; server-side authentication against Cognito (no redirect to Cognito hosted pages) |
| **Self-service registration** | Creates Cognito user and actor profile in one flow                                               |
| **Google sign-in**            | OAuth2 with short-lived temporary token handoff to the browser                                   |
| **LinkedIn sign-in**          | OAuth2 with short-lived temporary token handoff to the browser                                   |
| **Token refresh**             | Long-lived sessions without re-entering credentials on every visit                               |
| **Browser entry point**       | CloudFront → backend-for-frontend (BFF) → domain services                                        |

## Request path (deployed environments)

```
Browser  →  CloudFront (apex hostname)
        →  Internet-facing ALB (API paths only)
        →  actor-bff
        →  auth-service / actor-service / document-service
```

Static UI assets are served from S3 via CloudFront. API paths (`/auth/*`, `/actors/*`, `/documents/*`, etc.) forward to the application load balancer with origin verification. See [Platform security posture](/docs/security) and [Infrastructure](/docs/infrastructure).

Typical user-facing operations:

| User action         | Entry path                                                    |
|---------------------|---------------------------------------------------------------|
| Sign in             | `POST /auth/login`                                            |
| Register            | `POST /auth/register`                                         |
| Refresh session     | `POST /auth/refresh-user-token`                               |
| Google sign-in      | Redirect flow via `/auth/google/login`, then token exchange   |
| LinkedIn sign-in    | Redirect flow via `/auth/linkedin/login`, then token exchange |
| Profile / documents | Authenticated calls through the BFF                           |

## Authentication flows

### Email and password (primary)

```
Browser → BFF → auth-service → Cognito
       ← JWTs (access, id, refresh) ←
Client stores tokens; subsequent calls use Authorization: Bearer
```

This keeps branding and UX in your product while Cognito still issues standard OIDC-shaped JWTs.

### Google and LinkedIn OAuth2

1. User starts Google or LinkedIn sign-in from the product UI. auth-service redirects to the identity provider with an opaque one-time **`state`** nonce and sets an HttpOnly `Path=/auth` CSRF cookie that binds that nonce to the browser.
2. After the provider callback, auth-service matches `state` to the cookie, consumes the nonce, then issues a **temporary, single-use token** to the browser (not a long-lived JWT in the URL).
3. Browser exchanges the temporary token for Cognito JWTs via a dedicated endpoint.

LinkedIn **account linking** uses a one-time server-side nonce issued only after the user is authenticated.

Temporary tokens expire quickly and are invalidated on first use — reducing exposure in redirect URLs.

### Registration

```
Browser → BFF → auth-service → Cognito (create user)
                           → actor-service (create profile)
                           → notification-service (welcome email, queued)
       ← JWTs ←
```

Registration is a multi-step backend orchestration.

## Security principles

These apply to **human identity** specifically. Platform-wide edge and network assumptions are in [Platform security posture](/docs/security).

1. **Verify explicitly** — Protected APIs require a valid user JWT; anonymous access is denied by default on secured routes.
2. **Least privilege** — Authentication proves identity; authorization rules apply at the resource layer.
3. **Assume breach** — Tokens are short-lived and refreshable; a compromised user token does not grant service-to-service identity (see [Service authentication](/docs/service-authentication)).
4. **No network trust** — Source IP and VPC location are not treated as proof of user identity.
5. **CSRF model** — API credentials are Bearer JWTs from `localStorage`, not cookies, so classic cookie CSRF does not apply to BFF/API calls. Google and LinkedIn **login** bind OAuth `state` to an HttpOnly `Path=/auth` CSRF cookie plus Redis consume-once. LinkedIn **linking** uses a one-time server-side nonce issued only after the user is authenticated. XSS (token theft) is the primary browser threat and is addressed with edge CSP; see [Platform security posture](/docs/security#csrf-posture).

User JWT validation results are cached in Redis across ECS tasks for performance; see [Application caching and distributed scale](/docs/caching).

## Operator expectations

| Topic                   | Expectation                                                                                                                                            |
|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Identity provider**   | Cognito user pool per environment; password policy configured in CDK                                                                                   |
| **Social login**        | Google and LinkedIn developer app credentials supplied as deployment secrets                                                                           |
| **Token storage**       | Browser `localStorage` for JWTs; XSS mitigated with CSP (not cookie CSRF tokens). OAuth login uses a `Path=/auth` CSRF nonce cookie, not session auth. |
| **Future enhancements** | MFA, passkeys, and centralized revocation are roadmap items, not baseline                                                                              |

## Further reading

- [Service authentication](/docs/service-authentication) — workload identity (separate trust domain)
- [ADR-0011: Stateless JWT authentication](/docs/0011-stateless-jwt-authentication)
- [ADR-0004: Use AWS Cognito across all environments](/docs/0004-use-aws-cognito-across-all-environments)
- [Amazon Cognito user pool security best practices](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-security-best-practices.html) — AWS Documentation
