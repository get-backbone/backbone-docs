---
title: "0005. Implement Service-to-Service Authentication Using AWS STS AssumeRoleWithWebIdentity"
summary: "Our authentication design established AWS Cognito as the identity provider across all environments"
---

**Status:** Accepted  
**Date:** 2025-10-27  
**Context:** Continuation of ADR-0004; service-to-service authentication and secure AWS IAM access via Cognito and STS.

## **Context**

Our authentication design established AWS Cognito as the identity provider across all environments (production, staging, and development). Both environments use Cognito **User Pools** for username/password login and social identity integration via **Identity Pools**.

As we move toward full integration, it became clear that our backend services will require **direct access to AWS resources** such as **DynamoDB, S3, and SQS**. This introduces an additional requirement beyond simple service-to-service authentication: **services must be able to obtain AWS IAM credentials securely**.

The previous approach using **User Pool JWTs** suffices for service-to-service identity verification, but does not allow AWS API access since JWTs are not IAM credentials.

---

## **Decision**

We will implement **service-to-service authentication and AWS access** using **Cognito User Pool tokens** combined with **AWS STS AssumeRoleWithWebIdentity**.

### **Implementation Overview**

1. **JWT Issuance (Identity Layer)**

    * Each service authenticates with the configured Cognito User Pool (real AWS Cognito in all environments).
    * The service receives a standard Cognito **access token (JWT)**.

2. **Credential Exchange (Authorization Layer)**

    * The service exchanges its JWT for **temporary IAM credentials** using the AWS Security Token Service (STS) API: `AssumeRoleWithWebIdentity`.
    * STS validates the JWT against the corresponding Cognito User Pool.
    * STS issues temporary credentials (`AccessKeyId`, `SecretAccessKey`, `SessionToken`) for the assumed IAM role.

3. **Resource Access**

    * The service uses these credentials to call AWS APIs (DynamoDB, S3, etc.) or other internal services.
    * Temporary credentials are cached and refreshed before expiry.

4. **Environment Parity**

    * **Production/Staging/Development:** Real AWS Cognito + STS across all environments.
    * The same adapter logic applies across all environments; only configuration differs.

---

## **Implementation Detail: CognitoTokenValidator**

The `CognitoTokenValidator` will:

* Authenticate against AWS Cognito in all environments.
* Retrieve a valid access token (JWT).
* Expose the token for local validation (service-to-service) or for STS exchange.

A new component, `StsWebIdentityAdapter`, will:

* Call `AssumeRoleWithWebIdentity` with the Cognito JWT.
* Return an AWS credentials object (temporary IAM credentials).
* Handle token caching and expiration.

---

## **Alternatives Considered**

### **Option 1: Simple User Pool JWTs for Service-to-Service Auth (no STS)**

**Pros:**

* Simpler architecture, no IAM roles required.
* Works well for purely internal API-to-API calls.

**Cons:**

* Cannot access AWS services (DynamoDB, S3, etc.).
* Services would need static AWS credentials or other workarounds for AWS API access.

**Verdict:** Rejected for now — insufficient for AWS service access but could be revisited if architecture moves away from AWS-hosted workloads or if IAM access becomes unnecessary.

---

## **Consequences**

✅ **Benefits**

* Unified identity model for both human users and service principals across all environments.
* No static credentials embedded in services.
* Fully AWS-native security model with temporary IAM credentials.
* Same authentication flow across all environments (Cognito ↔ STS ↔ IAM).
* **Cost-effective:** Uses AWS free tier for development instead of expensive LocalStack Pro.

⚠️ **Trade-offs**

* Slightly higher complexity (two-step token + STS exchange).
* Requires maintaining service identities and role mappings in Cognito/Identity Pools.
* Additional latency from STS exchange (mitigated via credential caching).
* Requires internet connectivity for development (vs local emulation).

---

## **Future Considerations**

If the platform evolves to where:

* Services no longer require AWS API access, or
* Service-to-service calls are isolated within a trusted internal mesh (e.g., via mTLS or service mesh identity),

then the system could revert to a **simpler JWT-only model** (Option 1). This would reduce complexity while preserving inter-service trust.

Such a refactor would involve:

* Removing the STS exchange step.
* Validating JWTs directly against Cognito JWKS.
* Maintaining internal service roles via claims rather than IAM roles.

---

## **References**

* [AWS STS AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html)
* [AWS Cognito Identity Pools and Role Mapping](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html)
* [AWS Cognito Pricing](https://aws.amazon.com/cognito/pricing/)

---
