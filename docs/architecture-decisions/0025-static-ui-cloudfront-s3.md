---
title: "0025. Static UI hosting on CloudFront and S3"
summary: "The web UI is hosted on S3 and CloudFront rather than ECS, enabling independent frontend deployments and edge delivery while retaining same-origin API access."
---

**Status:** Accepted  
**Date:** 2026-06-13

## **Context**

The web UI is a static HTML, CSS and JavaScript application.

Historically it was deployed as an ECS service alongside backend services. This introduced unnecessary runtime infrastructure, operational cost, and deployment coupling between frontend and backend releases.

The platform requires:

- independent frontend deployments
- edge caching and global delivery
- support for same-origin API access
- retention of the public UI when backend runtime infrastructure is temporarily unavailable

---

## **Decision**

The web UI will be hosted using AWS static hosting patterns:

- Static assets are stored in S3.
- CloudFront serves the application globally.
- API requests continue to route to backend services through the platform's existing ingress architecture.
- Frontend deployments are independent of backend deployments.
- Static hosting infrastructure remains available even when runtime compute infrastructure is removed.

The UI will no longer be deployed as an ECS service.

---

## **Consequences**

### Positive

- Reduced infrastructure cost.
- Eliminates a dedicated ECS service used only for static content delivery.
- Independent frontend release cadence.
- Improved edge caching and global performance.
- Static UI remains available during runtime environment teardown.

### Negative

- Cache invalidation is required when deploying updated assets until immutable asset versioning is adopted.
- Frontend delivery now depends on CloudFront and S3 operational characteristics.

---

## **Alternatives considered**

### Continue hosting on ECS

Rejected because the application does not require server-side execution and gains no benefit from dedicated compute resources.

### Static hosting on S3 without CloudFront

Rejected because edge caching, performance, and security controls are important platform requirements.
