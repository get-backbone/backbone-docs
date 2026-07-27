---
title: "0004. Use AWS Cognito Across All Environments (Production and Development)"
summary: "We need a unified authentication strategy for a web application that supports:"
---

**Status:** Accepted  
**Date:** 2025-10-27  
**Context:** Continuation of ADR-0003 (Auth); unified Cognito across environments after development tooling and cost analysis.

## **Context**

We need a unified authentication strategy for a web application that supports:

1. **User login with username/password**
2. **Social login via Google, Facebook, etc.**
3. **Service-to-service authentication between backend services using JWTs**

Constraints and considerations:

* **Development environment:** Fast local iteration, cost-effective, minimal external dependencies.
* **Production environment:** Secure, highly available, fully managed, and scalable.
* **Cost considerations:** Development tools should not exceed production costs.
* Existing options include **Keycloak**, **AWS Cognito**, **LocalStack Pro**, and **cognito-local** for local development.

---

## **Options Considered**

### **Option 1: Keycloak (Self-hosted)**

**Pros:**

* Flexible identity broker: can combine social logins, corporate IdPs, and internal users.
* Supports OIDC and SAML.
* Full control over auth flows, MFA, and custom claims.

**Cons:**

* Requires self-hosting and operational overhead (HA, backups, security updates).
* Adds complexity for service-to-service JWT validation.
* Redundant if using AWS Cognito for production.

**Verdict:** Not ideal — adds operational burden and duplicates functionality available via AWS services.

---

### **Option 2: jagregory/cognito-local (Local Development Emulator)**

**Pros:**

* Lightweight emulator of AWS Cognito User Pools.
* Supports local username/password login and token issuance (JWTs).
* Easy to spin up in dev environments.

**Cons:**

* **Does NOT support social login flows** via Google, Facebook, etc.
* Limited support for Identity Pools and federated credentials.
* May diverge from production behavior in complex scenarios.

**Verdict:** Useful for local JWT-based testing, but insufficient for social login emulation.

---

### **Option 3: AWS Cognito (Managed Service)**

**Pros:**

* Fully managed and production-grade.
* Supports **User Pools** (username/password) and **Identity Pools** (social login).
* Issues JWTs for both user sessions and service-to-service authentication.
* Integrates with AWS ecosystem and scales automatically.
* **Cost-effective:** Uses AWS free tier (50,000 monthly active users) for development.
* **Full feature parity:** Identical to production environment across all environments.
* **No mocking required:** Real social login flows, Identity Pools, and STS integration work in development.
* **Consistent behavior:** Eliminates environment-specific bugs and differences.

**Cons:**

* Requires internet connectivity for development (vs local emulation).
* Slightly more complex setup than local emulation.

**Verdict:** Best fit for both production and development - provides full functionality at minimal cost using AWS free tier.

---

## **Decision**

We will adopt the following strategy:

| Environment              | Auth Provider | Features                                                                                   |
| ------------------------ | ------------- | ------------------------------------------------------------------------------------------ |
| **Production / Staging** | AWS Cognito   | User Pools (username/password), Identity Pools (social login), JWTs for service-to-service |
| **Development / Local**  | AWS Cognito   | User Pools (username/password), Identity Pools (social login), JWTs for service-to-service |

**Key Notes:**

* **Cost optimization:** AWS Cognito free tier (50,000 monthly active users) is significantly cheaper than LocalStack Pro ($40/user/month).
* **Environment parity:** Same authentication service across all environments eliminates environment-specific issues.
* **Full functionality:** No mocking required - real social login flows and Identity Pools work in development.
* **LocalStack Community:** Still used for other AWS services (S3, DynamoDB, etc.) that are fully supported in the free version.
* **Simplified architecture:** Single authentication provider across all environments reduces complexity.

---

## **Consequences**

* **Cost savings:** AWS Cognito free tier provides 50,000 monthly active users vs LocalStack Pro at $40/user/month.
* **Environment parity:** Identical authentication behavior across all environments eliminates development/production differences.
* **Simplified architecture:** No mocking or emulation required - real AWS services in all environments.
* **Full feature support:** Complete social login flows and Identity Pools work in development.
* **Hybrid approach:** LocalStack Community still used for other AWS services (S3, DynamoDB, etc.) that are fully supported.

---

## **References / Investigations**

* **AWS Cognito Documentation:** User Pools, Identity Pools, JWTs
* **AWS Cognito Pricing:** Free tier vs paid tiers
* **LocalStack Pro Pricing:** $40/user/month cost analysis
* **Keycloak:** Flexible but operationally heavy; not chosen
* **jagregory/cognito-local:** Supports basic JWT issuance, but **does not support social login or Identity Pools**

---

This ADR ensures that authentication is consistent, secure, and cost-effective across all environments, while providing full functionality without expensive development tools.

---
