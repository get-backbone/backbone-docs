---
title: "Compliance"
summary: "How Backbone maps to common security and compliance expectations — and what operators must still implement and evidence."
---

<!-- markdownlint-disable-file MD013 -->

Backbone cannot represent a production deployment as "compliant" on its own. Backbone is not operated as a centralized SaaS platform. Every deployment is forked, extended, configured, and operated inside **AWS accounts owned by the operator**. Compliance outcomes depend on the deployed system, operational controls, organizational processes, and legal agreements — not on the source repository alone.

This guide is for architecture review, procurement, and compliance diligence. It maps platform capabilities to control objectives commonly discussed under frameworks such as SOC 2, GDPR, and HIPAA-aligned environments. It is **not** an audit determination, certification, attestation, or legal interpretation. Start with [Platform security posture](/docs/security) for the control baseline this guide maps against.

## Scope and operating model

Backbone ships a security-focused technical baseline designed to support common compliance objectives. The baseline includes identity and access patterns, private networking, edge protection, secrets handling, audit event foundations, and repeatable infrastructure deployment.

Progress indicators in the tables below reflect **what the platform implements** versus what operators must wire, policy, or evidence in their environment.

Backbone operates at the **application platform layer**, not as an AWS landing zone. It intentionally avoids mandating a specific AWS Organizations layout or provisioning account- and organization-level security services — for example AWS Organizations security OUs, AWS Config, Security Hub, GuardDuty, or Shield Advanced. Those controls need org-wide delegated administration, account topology, and often already exist in an enterprise landing zone; Backbone cannot assume that shape without conflicting with operator foundations. Operators may run single-account sandboxes or multi-account landing zones. Both paths can satisfy control objectives when required org-level controls are implemented and evidenced in the operator environment.

## Shared responsibility

| Party        | Role                                                                                                                                                                                                                                           |
|--------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Backbone** | Reference services, libraries, and infrastructure-as-code operators deploy. Security-relevant patterns — authentication, authorization, audit emission, network defaults — are implemented where stated in this guide.                         |
| **Operator** | Owns AWS accounts, data classification, workforce policies, vendor agreements (for example BAAs for HIPAA, DPAs for GDPR), SOC 2 control operation and evidence, monitoring of the full account, and gap closure beyond the platform baseline. |

Deploying Backbone does not make an operator compliant. Compliance requires the deployed system, operating processes,
and legal agreements to work together.

## How to read control coverage

The tables below separate three layers auditors typically ask about:

1. **What Backbone implements** in the standard deploy path.
2. **What frameworks commonly expect** at a high level.
3. **What the operator must configure, operate, and evidence** in their environment.

| Symbol | Meaning                                                                                |
|:------:|----------------------------------------------------------------------------------------|
|   ✅   | Implemented in the platform baseline for a standard AWS deployment.                    |
|   🟠   | Partial — building blocks exist; operator wiring, policy, or extension still required. |
|   ❌   | Not in the platform baseline — operator-owned, roadmap item, or integration topic.     |

## Architectural planes

Auditors often partition systems into planes. The table maps that vocabulary to Backbone without prescribing a single
account model.

| Plane                          | Meaning in Backbone                                                                  | Where to read more                                                                                                                             |
|--------------------------------|--------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| **Control plane**              | Human and service identity, authorization, configuration, notification orchestration | [User authentication](/docs/user-authentication), [Service authentication](/docs/service-authentication), [Notifications](/docs/notifications) |
| **Audit and governance**       | Immutable-style event capture and action traceability across actors and services     | [Audit](/docs/audit)                                                                                                                           |
| **Data plane**                 | Domain data in PostgreSQL, DynamoDB, and S3 as provisioned per environment           | [Infrastructure](/docs/infrastructure)                                                                                                         |
| **Observability and security** | Logging, metrics, tracing hooks; edge and network controls; governance evidence      | [Observability architecture](/docs/observability), [Platform security posture](/docs/security)                                                 |

## Control coverage by domain

### Identity and access management

|    | Capability                                                | Platform provides                                                                  | Operator still owns                                           |
|:--:|-----------------------------------------------------------|------------------------------------------------------------------------------------|---------------------------------------------------------------|
| ✅ | Human authentication (JWT via Amazon Cognito)             | User pools, password policies, stateless token model                               | Access reviews, administrator MFA, workforce IdP integration  |
| ✅ | Service-to-service identity and caller authorization      | Dedicated service pool, per-request validation, explicit caller restrictions       | IAM role alignment per environment, service account lifecycle |
| ✅ | AWS API access from workloads via short-lived credentials | ECS task roles issue temporary credentials; no long-lived IAM user keys at runtime | Role scoping reviews and evidence per account                 |
| ❌ | Enterprise SSO (SAML or OIDC IdP federation)              | Intentionally remains a client decision                                            | IdP federation design and operation                           |
| ✅ | CI/CD deployment without long-lived AWS keys              | GitHub Actions OIDC to constrained deployment roles                                | Production approval, segregation of duties, release records   |

### Network and edge

|    | Capability                                           | Platform provides                                                                 | Operator still owns                                 |
|:--:|------------------------------------------------------|-----------------------------------------------------------------------------------|-----------------------------------------------------|
| ✅ | VPC per environment with public/private segmentation | Dedicated VPC, restricted default security groups                                 | Account layout and peering decisions                |
| ✅ | Workloads in private subnets                         | ECS Fargate tasks without public IPs                                              | Capacity and scaling policy                         |
| ✅ | CloudFront edge with WAF                             | TLS termination, host allowlist, rate limits, geo controls                        | WAF tuning, false-positive review, monitoring       |
| ✅ | Restricted API origin (not open internet ALB)        | Defense in depth: edge WAF, network allowlisting, origin verification             | Integration testing through the CloudFront hostname |
| ✅ | Internal service-to-service transport encryption     | Optional HTTPS on the internal ALB (ACM + private DNS);                           | mTLS/mesh if hop-to-task encryption is required     |
| ✅ | Private connectivity to AWS APIs                     | VPC interface endpoints for secrets, identity, container registry, logging, email | Endpoint policy review                              |

### Data protection

|    | Capability                                              | Platform provides                                                                                                                                          | Operator still owns                                                                                     |
|:--:|---------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| ✅ | PostgreSQL encryption at rest                           | Encrypted RDS in private subnets                                                                                                                           | Data classification, key custody if CMK required                                                        |
| ✅ | Network isolation for relational data                   | Security groups limit database access to application tasks                                                                                                 | Tenant boundary design in application layer                                                             |
| ✅ | Automated relational backups with stage-based retention | Backup retention varies by environment stage                                                                                                               | RPO/RTO targets and restore testing                                                                     |
| ✅ | DynamoDB and S3 for platform data                       | Default encryption on managed datastores                                                                                                                   | Per-table and per-bucket classification                                                                 |
| ✅ | Customer-managed KMS keys (configurable)                | When enabled, CMKs (provisioned or BYOK) for RDS, Redis, DynamoDB, application S3, Secrets Manager, audit SQS, static-edge assets, and governance evidence | BYOK key policies and rotation evidence                                                                 |
| ✅ | Single-tenant deployment isolation                      | One client environment per owned AWS account; VPC and data plane scoped to that deploy                                                                     | If the fork later becomes multi-tenant in-process, row/org isolation in domain services and data models |

### Secrets and configuration

|    | Capability                               | Platform provides                                                         | Operator still owns                               |
|:--:|------------------------------------------|---------------------------------------------------------------------------|---------------------------------------------------|
| ✅ | Runtime secrets from AWS Secrets Manager | Database, identity, OAuth, and service credentials injected at task start | Secret rotation policy and access reviews         |
| ✅ | Service account password rotation        | Automated rotation for Cognito service accounts                           | Break-glass and emergency access procedures       |
| ✅ | No secrets in application source         | Secrets supplied via environment and managed stores                       | Developer workstation and CI secret hygiene       |
| ✅ | Centralized configuration                | Shared configuration module across services                               | Environment-specific overrides and change control |

### Audit and logging

|    | Capability                                                         | Platform provides                                                                                                              | Operator still owns                                                         |
|:--:|--------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| ✅ | Application audit events to a central audit service                | Annotation-driven emission, EventBridge bus → SQS ingest → PostgreSQL                                                          | Retention, access control on audit data, investigation runbooks             |
| ✅ | Event-driven audit routing                                         | Custom bus `audit-events-bus`; baseline rule to SQS; additional SIEM targets attach as rules                                   | Operator SIEM / cross-account rule configuration                            |
| ✅ | Edge access logging                                                | CloudFront and static-content access logs; ALB access logs when `governanceEvidenceEnabled` is true                            | Log review, retention, and SIEM integration                                 |
| ✅ | VPC reject flow logs                                               | Enabled by default; one-year retention in production                                                                           | Forensic procedures and alert response                                      |
| ✅ | Immutable central audit archive                                    | CloudTrail evidence bucket with S3 Object Lock (COMPLIANCE) and stage-aware retention when governance is enabled               | Dedicated security / log-archive account, counsel legal hold, SIEM routing  |
| ✅ | Regional CloudTrail and evidence bucket                            | Regional trail + global IAM/STS when governance is enabled (account-scoped, not stack-scoped)                                  | Disable when org-wide trail exists; dedicated workload account recommended  |
| ✅ | Governance evidence protection (encryption, versioning, lifecycle) | Dedicated evidence bucket with SSE-S3 or CMK, versioning, Object Lock, and stage-aware lifecycle when governance stack deploys | Retention policy tuning, legal hold, and SIEM routing remain operator-owned |
| ✅ | PII-safe logging discipline                                        | JSON logging, correlation MDC, `sensitive-data-mask` filter on deployed profiles                                               | Log review, DLP, and data-minimization policy                               |

### Observability

|    | Capability                               | Platform provides                                                                                | Operator still owns                                                |
|:--:|------------------------------------------|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| ✅ | OpenTelemetry instrumentation            | Tracing hooks in Quarkus services                                                                | Sampling policy, AMP/AMG cost governance, and dashboard ownership  |
| ✅ | JSON logging to CloudWatch               | Structured JSON on `%int` / `%test` / `%prod` with correlation and trace MDC                     | Log retention, index policies, and SIEM integration                |
| ✅ | Application telemetry and dashboards     | AMP remote write, AMG workspaces, and local Grafana assets when managed observability is enabled | Dashboard ownership, on-call runbooks, and AMP/AMG cost governance |
| ✅ | Infrastructure monitoring (infra alarms) | Per-stack CloudWatch alarms → SNS                                                                | Alarm routing, on-call, and escalation policies                    |

### Change management and secure engineering

|    | Capability                            | Platform provides                                                                | Operator still owns                             |
|:--:|---------------------------------------|----------------------------------------------------------------------------------|-------------------------------------------------|
| ✅ | Automated build and test on change    | CI pipeline on push and pull request                                             | Production promotion gates                      |
| ✅ | Static analysis in CI                 | Infrastructure and code quality checks                                           | Remediation SLAs and exception tracking         |
| ✅ | Secret scanning on pre-push and in CI | Repository secret detection in development and in pipeline                       | Broader supply-chain tooling as needed          |
| ✅ | Infrastructure policy checks          | CDK Nag with documented suppressions                                             | Review of suppressions and drift detection      |
| ✅ | Dependency update automation          | Renovate configuration in repository (scheduled PRs, grouped updates, automerge) | Dependabot, CodeQL, or equivalent org standards |

### Data subject rights, retention, and communication

|    | Capability                                                    | Platform provides                                              | Operator still owns                                                                                   |
|:--:|---------------------------------------------------------------|----------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| ❌ | Right to erasure                                              | No account-deletion or cross-store erasure API                 | Operator procedures (and product features) for Art. 17 across Cognito, RDS, DynamoDB, S3, and backups |
| ❌ | Personal data export API                                      | No single cross-store export                                   | Articles 15 and 20 procedures and product features                                                    |
| ❌ | Legal retention and hold                                      | No legal-hold or counsel-driven retention controls in baseline | Retention schedules, legal hold, and counsel review                                                   |
| ✅ | Transactional notifications with templates and rate awareness | Template-based email with provider throttling                  | Content and marketing policy, bounce handling, subprocessor DPAs with notification providers          |
| ✅ | Notification unsubscribe integrity                            | Opaque tokens without PII in URLs                              | Consent records and marketing compliance                                                              |

## SOC 2 (Trust Services Criteria) quick map

Typical mappings auditors discuss. Criteria labels vary by report.

| Theme                                 | How Backbone helps                                                                         | What the operator still proves                                                |
|---------------------------------------|--------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| **CC6 (logical and physical access)** | Cognito, JWT, service authorization, private networking, CloudFront edge WAF               | Access reviews, administrator MFA, break-glass, staff IdP, workforce training |
| **CC7 (system operations)**           | Health checks, observability hooks, audit event pipeline, infrastructure monitoring alarms | Incident response, alerting on AWS and application logs, backup restore tests |
| **CC8 (change management)**           | GitHub Actions, infrastructure as code, automated tests                                    | Production approvals, segregation of duties, release records, change reviews  |

## GDPR-oriented notes

- **Controller vs processor**: Operators are typically controllers for their users' data. Backbone is software they
  operate. Legal roles depend on contracts and facts, not this document.
- **Technical measures**: TLS at the public edge, encryption at rest on managed datastores, secrets handling, and audit
  hooks support Article 32 security-of-processing discussions when operators complete logging, monitoring, and incident
  practices.
- **Data subject rights**: Operators implement procedures — and often product features — for access, rectification,
  erasure, and portability across Cognito, relational stores, DynamoDB, and object storage.

## HIPAA-oriented notes (high level)

HIPAA compliance depends on a BAA, scoped systems, and operational safeguards. Backbone can support technical safeguards
commonly expected in HIPAA-aligned deployments — access control, audit records, encryption, transmission security — when
deployed and operated appropriately. This document does not offer a BAA. Operators must classify PHI, close remaining
gaps such as multi-account log archives and complete monitoring where required, and execute vendor agreements with
subprocessors including AWS.

## Operator expectations

| Topic                 | Expectation                                                                                                              |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------|
| **Control ownership** | Map platform capabilities to your control framework; assign owners for every 🟠 and ❌ row above                         |
| **Evidence**          | Maintain policies, access reviews, backup tests, and incident records independent of Backbone source                     |
| **Extensions**        | Add mTLS/mesh, SIEM, and org-wide security services where required                                                       |
| **Legal agreements**  | Execute BAAs, DPAs, and subprocessors agreements appropriate to your jurisdiction and data types                         |
| **Landing zones**     | If using an enterprise landing zone, map Backbone objectives into that foundation rather than reshaping the organization |

## Further reading

- [Platform security posture](/docs/security)
- [Observability architecture](/docs/observability) — how operational concerns are separated
- [Application telemetry](/docs/application-telemetry)
- [Infrastructure monitoring](/docs/monitoring)
- [Audit logging](/docs/audit)
- [Notifications](/docs/notifications)
- [Architecture decision records](/docs/adrs)
- [ADR-0026: Observability backend strategy](/docs/0026-observability-backend-strategy)
- [ADR-0027: Governance evidence architecture](/docs/0027-governance-evidence-architecture)
- [ADR-0028: Governance evidence Object Lock](/docs/0028-governance-evidence-object-lock)
- [ADR-0016: Notification service rate limiting](/docs/0016-notification-service-rate-limiting-strategy)
- [ADR-0019: Notification unsubscribe token security](/docs/0019-notification-service-unsubscribe-token-security)
- [AWS Compliance Programs](https://aws.amazon.com/compliance/programs/) — AWS
