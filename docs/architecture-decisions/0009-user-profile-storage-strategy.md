---
title: "0009. User Profile Storage Strategy"
summary: "We must choose between:"
---

**Status:** Accepted  
**Date:** 2025-11-25  
**Context:** Cognito handles auth; we must store extended profile attributes beyond basic identity and choose Cognito attributes vs application database.

## **Context**

We must choose between:

1. Storing extended attributes directly in **AWS Cognito** user attributes, or
2. Using **Cognito only for authentication** and storing extended attributes in our **own application database** (e.g., RDS).

Enterprise requirements include: scalability, schema evolution, auditability, portability, integration with domain models, and avoiding vendor lock-in.

---

## **Decision**

Use **Cognito exclusively for identity and authentication** and store all extended user profile attributes in our **application database (RDS)**. Cognito remains the identity provider; the application database becomes the system of record for user profile data.

---

## **Rationale**

### **Why not store extended attributes in Cognito**

* Cognito attributes are limited, rigid, and cannot be deleted or migrated easily.
* Not suitable for structured, relational, or evolving profile data.
* Poor fit for items such as profile photos, social handles, preferences, or audit history.
* Difficult to perform reporting, complex queries, or data analytics.
* Storing extensive domain data in an IdP couples business logic to a replaceable component.

### **Why store extended attributes in our own DB**

* Full control over schema evolution, indexing, constraints, and migrations.
* Supports complex, structured, and rich data models.
* Clean separation between authentication (Cognito) and domain data (RDS).
* Enables future migration from Cognito without losing key user information.
* Supports audit logs, versioning, and relational relationships with other domain entities.

---

## **Hybrid Architecture Pattern (Recommended)**

### **Store the following *in Cognito*** (minimal identity set)

| Attribute                               | Reason                         |
| --------------------------------------- | ------------------------------ |
| **sub**                                 | Unique, stable user identifier |
| **email**                               | Primary login credential       |
| **email_verified**                      | Auth logic / token claims      |
| **phone_number** (optional)             | Required only if used for MFA  |
| **given_name / family_name** (optional) | Convenience, but not mandatory |

### **Store the following *in the application database*** (system of record)

* Date of birth
* Mobile phone (if not used for MFA)
* LinkedIn profile handle / social URLs
* Profile photo (via S3 URL)
* Resumes / media references
* Preferences & notification settings
* Job-seeker profile data
* Roles & permissions (fine-grained)
* Any domain-specific attributes

---

## **Architecture Overview**

```
                    AWS Cognito (IdP)
                    - Authentication
                    - Tokens
                    - MFA
                          |
                          | sub
                          v
           Application Database (RDS)
           - User Profile (system of record)
           - Preferences
           - Media references (S3)
           - Domain model relationships
```

User creation flow:

* User signs up in Cognito → Lambda trigger creates corresponding user row in DB using Cognito `sub`.

Application access flow:

* Backend receives JWT → validates → fetches user profile from RDS.

---

## **Consequences**

### **Positive**

* Clean architectural separation between identity and domain data.
* Lower Coupling → easy to migrate away from Cognito.
* Scalable, flexible user data model.
* Supports rich domain logic and integrations.
* Centralized reporting and analytics.

### **Negative**

* Slightly more complexity due to maintaining two data sources.
* Requires a user creation sync mechanism (Lambda trigger or backend event).

---
