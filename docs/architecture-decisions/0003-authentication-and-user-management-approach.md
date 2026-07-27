---
title: "0003. Authentication and User Management Approach"
summary: "The platform requires secure, standards-based authentication and user management for both web and API clients."
---

**Status:** Accepted  
**Date:** 2025-10-08  
**Context:** Continuation of ADR-0002 (Quarkus adoption); defines authentication, authorization, and user data management.

## **Context**

The platform requires secure, standards-based authentication and user management for both web and API clients. Key requirements include:

* Support for OAuth 2.0 / OIDC flows (for future integration with third-party login providers).
* Secure token-based access for API consumers.
* Minimal operational burden for the small engineering team.
* Compatibility with Quarkus security extensions and AWS deployment model (ECS + Aurora PostgreSQL).
* Flexibility to evolve toward multi-tenant and enterprise scenarios.

Two primary models were considered:

1. **Application-managed identity** using **Quarkus Security JPA**, storing credentials and roles in our own database.
2. **Delegated identity management** using a managed OIDC provider, specifically **AWS Cognito**.

---

## **Decision**

We will adopt **AWS Cognito** as the primary authentication and user-management service. Quarkus will integrate via the **`quarkus-oidc`** extension, treating Cognito as an external OpenID Connect provider.

---

## **Rationale**

| Consideration          | Cognito (Chosen)                                                         | Quarkus Security JPA (Not Chosen for now)                                 |
| ---------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| **Operations**         | Fully managed service, handles signup, password reset, MFA, social login | Requires us to store and hash passwords, build account flows |
| **Security posture**   | Offloads credential handling and compliance to AWS                       | Higher risk surface; must implement secure storage and policies ourselves |
| **Scalability**        | Auto-scales with user base; integrates with AWS IAM                      | Scales with our Aurora DB; more tuning required |
| **Integration**        | Works natively with Quarkus OIDC and ALB OIDC auth                       | Tight coupling to DB schema; limited federation options |
| **Future flexibility** | Supports enterprise federation (SAML, OIDC, AD)                          | Limited to in-app users unless refactored later |

Given limited operational capacity at this stage, Cognito provides a secure, standards-compliant baseline while allowing later customization if business needs demand internal user management.

---

## **Implementation Outline**

### **In Quarkus**

```properties
quarkus.oidc.auth-server-url=https://cognito-idp.<region>.amazonaws.com/<user-pool-id>
quarkus.oidc.client-id=<client-id>
quarkus.oidc.credentials.secret=<client-secret>
quarkus.oidc.application-type=web-app
quarkus.oidc.authentication.scopes=openid,profile,email
```

* Tokens (JWTs) are verified automatically by Quarkus OIDC.
* Role/claim mapping handled via Cognito groups or custom attributes.
* Application endpoints protected via `@RolesAllowed` and `SecurityIdentity`.

### **Persistence Layer**

* **Aurora PostgreSQL** remains the primary store for domain data.
* Minimal user references (e.g. Cognito `sub` UUID, profile metadata) stored as foreign keys — no passwords.

### **Local Development**

* Use **AWS Cognito** in a development sandbox environment using AWS free tier.
* Quarkus's `quarkus.oidc.devservices.enabled=true` can spin up a local OIDC provider automatically when needed for unit testing.
* **Cost-effective:** AWS Cognito free tier (50,000 monthly active users) is significantly cheaper than LocalStack Pro ($40/user/month).

---

## **Future Alternatives**

If business needs require:

* **Tighter integration or internal credential control**, migrate selectively to **Quarkus Security JPA** (passwords stored in Aurora).
* **Cross-platform login** (Google, Apple, LinkedIn), leverage Cognito’s Identity Federation features.
* **Enterprise SSO**, integrate existing IdPs via Cognito Federation or replace Cognito with Keycloak.

---

## **Consequences**

**Positive**

* Removes responsibility for credential lifecycle, MFA, and compliance.
* Simplifies Quarkus configuration (OIDC instead of JPA-based security).
* Scales seamlessly within AWS ecosystem.

**Negative / Trade-offs**

* Introduces external dependency (Cognito availability, cost model).
* Requires internet connectivity for development (vs local emulation).
* Less direct control over user-table schema and credential policies.

---
