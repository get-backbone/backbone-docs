---
title: "Platform security posture"
summary: "Backbone is a production-oriented platform baseline for teams running containerized services on AWS. This guide explains the security controls that are…"
---

<!-- markdownlint-disable-file MD013 -->
<!-- MD013 off: policy prose, long URLs, and table rows exceed the repository line length limit without harm to readability. -->

Backbone is a production-oriented platform baseline for teams running containerized services on AWS. This guide explains the security controls that are implemented today, the risks that are intentionally accepted, and the controls that remain the responsibility of each client team.

This document is public and meant for architecture review, procurement review, and security review. User login details are in [User authentication](/docs/user-authentication). Service identity is in [Service authentication](/docs/service-authentication).

## Scope and operating model

Backbone provisions and operates infrastructure inside AWS accounts owned by the client organization. The platform ships opinionated defaults for network topology, identity, workload runtime, and CI access patterns.

Backbone is not a managed SaaS control plane. Client teams retain operational ownership of the AWS account, service code, data classification, key management decisions, and any controls required by their regulatory framework.

Backbone follows a single-tenant deployment model. Each client environment is isolated in the client-owned AWS account boundary.

## Threat model baseline

Backbone baseline assumptions:

1. Public endpoints are internet-exposed and should be treated as reachable by commodity attack traffic, including scanning and request flooding.
2. Internal network location is not sufficient proof of trust for protected operations.
3. Compromise of one service should not automatically authorize access to another service without explicit identity and authorization checks.

Backbone baseline non-assumptions:

1. Full service-mesh zero-trust controls such as mTLS and workload identity federation are not mandatory in the baseline.
2. Deep detection and response controls such as SIEM correlation and SOC operations are organization-level concerns outside platform defaults.

## Security principles

Backbone applies these principles by default:

1. Verify identity for user and service requests through JWT validation and explicit authorization checks.
2. Enforce least privilege in IAM policy design through narrow action sets and resource scoping.
3. Keep stateful infrastructure private by default and expose only explicit edge entry points.
4. Store sensitive configuration in managed secret stores instead of source control.
5. Apply a progressive hardening model where identity controls are always required and transport hardening is added per environment risk profile.

Progressive hardening means teams can layer stronger transport, key management, and policy controls without redesigning application-level identity and authorization logic.

No repository-authored baseline IAM policy or construct intentionally grants `Action: "*"`, and wildcard resource usage is explicitly documented and constrained.

## Network and edge security posture

### Trust boundaries

Backbone uses explicit trust boundaries:

1. Internet to CloudFront: untrusted traffic enters edge controls (WAF, TLS termination, caching).
2. CloudFront to internet-facing ALB: origin boundary gated by defense in depth (edge WAF, network allowlisting, and a shared origin-verify secret).
3. Internet-facing ALB to ECS services: application-layer authentication and authorization boundary (JWT and service token validation in Quarkus services, not at the ALB listener).
4. Service to service: identity-based trust through token validation, not network location.
5. AWS account boundary: primary tenant isolation boundary between client environments.

### VPC segmentation

Backbone deploys a dedicated VPC per environment with:

- public subnets for internet-facing load balancers (CloudFront origin targets)
- private subnets for application workloads and data-plane dependencies
- `restrictDefaultSecurityGroup: true` to avoid permissive default security group behavior

Traffic between network zones is controlled through explicit security group rules. In PROD, reject-only VPC flow logs are enabled and retained for one year to support operational and forensic review.

### Security group posture

Current ingress posture is explicit and narrow:

- Internet-facing ALB security group allows inbound TCP `443` from the AWS-managed CloudFront origin-facing **IPv4** prefix list only (the internet-facing ALB is IPv4-only).
- ECS service security group allows inbound application traffic only from ALB security groups.
- Internal ALB security group accepts traffic from ECS service security group only.

See [CloudFront origin protection (defense in depth)](#cloudfront-origin-protection-defense-in-depth) for how security-group rules combine with listener and edge controls.

The current internal ALB path uses HTTP (`80`) by default (`internalHttps: false`). Identity and authorization remain enforced at the application layer through JWT validation and service authorization checks. HTTPS on the internal listener is an Enterprise-tier opt-in via platform configuration.

Baseline implication: confidentiality of internal service traffic is not guaranteed by default transport settings and should be hardened for environments with stricter compliance or threat requirements.

Baseline traffic expectation: internal HTTP carries authenticated application payloads and is not designed as a transport channel for long-lived plaintext credentials. Client teams remain responsible for payload classification and required transport hardening for their risk profile.

### CloudFront origin protection (defense in depth)

API traffic reaches the internet-facing ALB only as a **CloudFront HTTPS origin** on a dedicated regional hostname. Three complementary controls apply; none replaces the others.

| Control                                                                             | Layer                | What it blocks                                                                                                                                                               |
|-------------------------------------------------------------------------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| CloudFront-scoped AWS WAF                                                           | Edge (viewer)        | Wrong-host, abusive, or geo-blocked **viewer** traffic before it reaches regional infrastructure                                                                             |
| CloudFront origin-facing **IPv4** prefix list on internet-facing ALB security group | Network (origin)     | TCP `443` connections whose source is **not** in the published CloudFront origin-facing ranges (before TLS/HTTP is processed)                                                |
| Shared origin-verification header on ALB listener rules                             | Application (origin) | HTTPS requests that reach the listener **without** valid CloudFront origin authentication (default listener action **403**; path rules forward only when the header matches) |

**Primary origin gate:** **CloudFront origin authentication** enforced at the ALB listener via a shared verification header. CloudFront adds the secret on origin requests; the value lives in Secrets Manager and is not exposed to browsers. This is the control that proves traffic was configured through the distribution’s origin settings.

**Supplementary network filter:** the **IPv4 prefix list** on the internet-facing ALB security group. It reduces direct internet reachability, scanning noise, and load on the ALB but consumes security-group rule capacity (each prefix-list entry counts toward the per-group quota). It is not required for a sound CloudFront-to-ALB design when the origin header is enforced; Backbone uses both for defense in depth.

The internet-facing ALB remains an internet-routable endpoint (required for CloudFront origin connectivity) but is not open to arbitrary internet sources at the security-group layer. Clients that bypass CloudFront IP ranges may still complete TLS to the listener; without valid CloudFront origin authentication they receive **403** and do not reach ECS target groups.

Private ALB plus CloudFront VPC origins remains an optional hardening path for stricter threat models.

### WAF and edge controls

Public traffic enters through Amazon CloudFront at the environment apex hostname (`{env}.{domain}`). A CloudFront-scoped AWS WAF WebACL (provisioned in `us-east-1`) is attached to the distribution with:

- host allowlist rule for the expected application hostname
- request-rate limiting rule
- geo-blocking rule

Host allowlist validation at the edge is a hygiene and traffic-shaping control for viewer requests. It is not cryptographic origin proof on its own.

API path behaviors on the distribution forward to the internet-facing ALB origin over HTTPS. Static UI assets are served from a private S3 bucket through Origin Access Control (OAC); the bucket blocks public access, enforces SSL, and uses server-side encryption.

CloudFront attaches a **Content-Security-Policy** response headers policy to the static default behavior and API path behaviors.
Production enforces CSP; non-production stages emit `Content-Security-Policy-Report-Only` so violations can be observed without breaking the UI.

**CORS** for browser-facing APIs is configured in Quarkus (`actor-bff`, and local `web-actor`). Origins are an explicit allowlist (`BACKBONE_PUBLIC_ALB_URL`, with a localhost fallback for local split-origin dev). Allowed request headers are enumerated (`accept`, `authorization`, `content-type` on the BFF); wildcards are not used. Deployed UI and API share the CloudFront apex hostname (same-origin), so CORS is mainly a guard for local development and misconfiguration. Auth uses Bearer JWT in `Authorization`, not cookies, so CORS credentials are not required.

WAF at the edge is the first layer in the [origin protection model](#cloudfront-origin-protection-defense-in-depth); ALB listener and security-group controls apply on the regional origin path.

## Identity and access management posture

### User and service identity

Backbone separates human and workload identity:

- user authentication flows issue JWTs through Cognito-backed auth services
- service-to-service calls use dedicated service accounts and service JWTs
- receiving services can enforce caller allowlists through `@AllowedServices`

The platform does not rely on a trusted internal network assumption for service authorization. Services validate token identity and claims on each protected request.

### CSRF posture

Cross-site request forgery (CSRF) is the browser attack where a hostile site causes the victim's browser to call your API using credentials the browser attaches automatically (typically session cookies).

Backbone's baseline API auth uses **stateless JWTs** sent as `Authorization: Bearer` from JavaScript (`localStorage`), not HttpOnly session cookies.
Classic cookie-based CSRF therefore does **not** apply to ordinary BFF and service API calls: a cross-origin page cannot read the token from `localStorage`, and the browser will not attach a Bearer header on its own.

Residual risks and controls:

| Concern                                | Baseline control                                                                                                                              |
|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| XSS steals tokens from `localStorage`  | CloudFront **Content-Security-Policy** (and related edge headers); see edge controls above                                                    |
| Hostile origins calling the API        | Explicit Quarkus **CORS** allowlists; same-origin UI+API via CloudFront in deployed envs                                                      |
| OAuth redirect forgery (login)         | Opaque one-time `state` in Redis, bound to an HttpOnly `oauth_login_state` cookie (`Path=/auth`, `SameSite=Lax`) on Google and LinkedIn login |
| OAuth redirect forgery (LinkedIn link) | Opaque one-time Redis nonce carrying actor id/email; linking starts as authenticated XHR so a CSRF cookie is not used                         |

The `oauth_login_state` cookie is a CSRF nonce for OAuth login callbacks only. It is not used for API session authentication and does not store access tokens.

**Explicit non-goals** for the baseline: double-submit CSRF tokens, SameSite cookie session auth, or moving access tokens into HttpOnly cookies. Those patterns belong in a future ADR if the token storage model changes.

### Least-privilege IAM implementation

IAM policies in infrastructure code are scoped to specific actions and resource sets wherever AWS APIs support resource-level constraints. Examples include:

- Cognito user-pool actions scoped to actor and service pool ARNs
- SSM reads scoped to specific Cognito parameter ARNs
- DynamoDB access scoped to known table and index ARNs
- S3 access split between bucket-level and object-level permissions
- Secrets Manager access scoped to required secret prefixes

### Wildcard usage policy and exceptions

Backbone avoids broad wildcard permissions for administrative access. Current wildcard usage is limited to AWS API patterns that require wildcard resources or predictable suffix matching:

- `ecr:GetAuthorizationToken` uses `Resource: "*"` because AWS requires account-level token retrieval.
- SES send and account health actions use `Resource: "*"` because SES actions in use are account-scoped.
- Some Secrets Manager and index ARNs include wildcard suffixes for deterministic name patterns generated by AWS or deployment naming conventions.

No `Action: "*"` or administrator-style privilege grants are part of the baseline platform constructs.

## Secrets and credential handling

Backbone stores application runtime-sensitive values in AWS Secrets Manager and AWS Systems Manager Parameter Store. ECS task and execution roles receive read access only for the secret paths and parameters required for runtime behavior.

GitHub repository and environment secrets are used for CI and CD workflow execution, bootstrap operations, and external integration credentials. GitHub secrets are never consumed directly by ECS runtime workloads and are not the system of record for application runtime secret retrieval.

Credential categories include:

- service account credentials for service authentication
- Cognito pool and client identifiers plus required client secrets
- encryption material and OAuth credentials needed by optional flows

Secret rotation is partially automated. The platform currently records explicit cdk-nag suppressions for selected secrets that do not have automatic rotation attached. Teams with stricter compliance requirements should enable managed rotation policies and key lifecycle controls per environment.

## Data protection controls

Encryption at rest posture in the baseline:

- S3 buckets are provisioned with server-side encryption (`S3_MANAGED`) and SSL enforcement, including static UI, edge access-log, and datastore buckets
- DynamoDB tables use DynamoDB-managed encryption at rest by default
- RDS PostgreSQL instances use storage encryption with an AWS-managed KMS key
- Secrets Manager data is encrypted at rest through AWS KMS-backed service behaviour
- Infrastructure provisioning bootstrap allows AWS-managed keys, platform-provisioned CMKs, or external BYOK ARNs per domain.

Encryption in transit posture in the baseline:

- public browser traffic terminates TLS at CloudFront (minimum TLS 1.2)
- CloudFront viewer policy redirects HTTP to HTTPS (`REDIRECT_TO_HTTPS`)
- CloudFront emits **Strict-Transport-Security** plus `X-Content-Type-Options` and `Referrer-Policy` on static and API behaviors
- CloudFront forwards API traffic to the internet-facing ALB over HTTPS
- the internet-facing ALB forwards to ECS tasks over HTTP within the VPC (ALB TLS termination does not extend to target groups)
- internal service-to-service traffic through the internal ALB uses HTTP by default (`internalHttps: false`) and can be switched to HTTPS on the ALB listener (Enterprise tier)

## Runtime workload isolation and container privilege posture

Current baseline runtime posture:

- services run on ECS Fargate tasks, not on shared EC2 container hosts
- service tasks run in private subnets with no public IP assignment
- shared Quarkus runtime images declare a non-root container user (`USER 1000:1000`) and own runtime paths as UID 1000
- ECS container definitions pin the same non-root user (`user: 1000:1000`) so the task definition enforces non-root even if an image omits `USER`

Current repository posture on privileged runtime flags:

- no repository-authored task definition sets privileged container mode (the field is omitted; Fargate does not support `Privileged` even when false)
- no repository-authored task definition enables host networking for application services
- hygiene CI fails builds when Dockerfiles end as root, or when infra sources introduce `privileged: true` or host `NetworkMode`
- CDK unit tests assert synthesized task definitions use `User: 1000:1000`, `NetworkMode: awsvpc`, and omit `Privileged`

Security interpretation:

- Backbone baseline reduces host-level attack surface by combining Fargate isolation with non-root container execution
- Backbone does not rely on privileged container capabilities for normal service operation

Future hardening options for stricter environments:

- read-only root filesystem where service runtime behavior allows it
- explicit container capability minimization beyond the privileged / host-network block
- account-level AWS Config or SCP controls that reject privileged tasks outside this repository

## CI/CD and GitHub security posture

Backbone uses GitHub Actions OpenID Connect (OIDC) for AWS role assumption. The trust policy constrains issuer, audience, and subject pattern so only approved workflow identities can assume the deployment role.

GitHub workflow permissions are scoped for deployment tasks:

- read-only managed policies for S3 and DynamoDB baseline access
- scoped ECR repository push and pull actions
- scoped ECS deployment actions for target cluster and services
- explicit bootstrap-role assumption permissions for CDK deploy flows

The platform does not require long-lived AWS access keys in GitHub for routine deployments once OIDC bootstrap is complete.

The baseline reduces credential supply chain risk by replacing long-lived CI cloud credentials with constrained, auditable OIDC role assumption.

## Logging and auditability posture

Platform logging controls currently implemented:

- in PROD, VPC flow logs for rejected traffic, retained for one year
- ECS task and application logs through CloudWatch log groups
- CloudFront access logs and S3 server access logs for static edge and datastore buckets
- WAF metrics and sampled requests through AWS WAF visibility configuration on the CloudFront WebACL

Organization-level controls expected outside baseline platform constructs (or when `governanceEvidenceEnabled` is false):

- Org-wide CloudTrail to a central log archive (preferred for enterprise landing zones)
- ALB access logging to operator-owned buckets and lifecycle controls
- SIEM integration, alerting, and incident response workflow

Backbone provisions an optional **regional** CloudTrail trail, protected evidence storage with S3 Object Lock (COMPLIANCE) and stage-aware default retention, and ALB access logs into a companion SSE-S3 bucket (Enterprise tier, when enabled).

Platform logging supports operational observability. Audit-grade retention, aggregation, and cross-system correlation are delegated to client environment configuration.

## Deliberate boundaries and non-goals

Backbone documents security trade-offs so clients can evaluate fit by risk profile.

Current deliberate boundaries include:

1. Internal service traffic encryption is not forced in the baseline. The baseline favors broad compatibility and lower operational friction, while preserving a clear migration path to stronger in-transit controls.
2. The internet-facing ALB retains an internet-routable endpoint for CloudFront origin connectivity. Ingress uses defense in depth: CloudFront origin-facing IPv4 prefix list on the security group plus origin verification at the listener (see [CloudFront origin protection](#cloudfront-origin-protection-defense-in-depth)). Private ALB plus CloudFront VPC origins remains an optional hardening path for stricter threat models.
3. Security hardening features that depend on client-specific policy or compliance context remain configurable rather than mandatory defaults.

These boundaries are intentional design choices, not accidental gaps. The platform supports incremental adoption of stronger controls based on client maturity and threat model.

### Explicit non-goals

Backbone baseline does not:

- mandate mTLS or service mesh adoption
- provide managed SOC, SIEM, or incident response operations
- assert compliance certification coverage for every client workload
- replace application-level authorization design in domain services

## Shared responsibility and client actions

Backbone provides a strong baseline. Client teams should still implement:

- environment-specific data classification and retention policy
- production TLS and certificate lifecycle governance for all endpoints
- organization-level logging, SIEM integration, and alert response runbooks
- key management and rotation controls aligned with compliance requirements
- workload-specific authorization rules beyond platform defaults

## Security maturity roadmap alignment

Security posture evolves through documented design decisions. Remaining enhancements such as SIEM reference patterns are tracked on the roadmap.

## Security control status matrix

This matrix summarizes security-relevant platform controls and maturity state for architecture and procurement review. Tier constraints are in [Platform features](/docs/features#licence-tiers-and-platform-configuration).

Status meaning:

- Implemented: baseline enforced by current platform implementation
- Extensible: hardening supported through code and infrastructure extension in the client-owned fork
- Planned: baseline enhancement tracked on the roadmap

| Control                                                    | Status      | Notes                                                                                                                                                |
|------------------------------------------------------------|-------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| Least-privilege IAM                                        | Implemented | Narrow action and resource scoping by default; bounded wildcards only where AWS APIs require them (see [IAM wildcard policy](#iam-wildcard-policy)). |
| OIDC-based CI and CD deployment identity                   | Implemented | Constrained GitHub Actions role assumption; no long-lived AWS keys for routine deploys.                                                              |
| Public edge protection (WAF and TLS)                       | Implemented | Viewer-facing WAF and TLS at the public edge entry point.                                                                                            |
| Content-Security-Policy at the edge                        | Implemented | CSP on static and API edge behaviors; report-only outside production, enforcing in production.                                                       |
| TLS-only viewers and HSTS at the edge                      | Implemented | HTTPS redirect, modern TLS floor, and browser hardening headers (HSTS, XCTO, Referrer-Policy).                                                       |
| Browser CORS allowlist                                     | Implemented | Explicit origins and headers for browser-facing APIs; no origin wildcards.                                                                           |
| CSRF posture                                               | Implemented | Bearer JWT for API auth (not session cookies); OAuth login uses one-time state binding. See [CSRF posture](#csrf-posture).                           |
| CloudFront origin verification and ALB ingress restriction | Implemented | Defense in depth at edge, network, and origin listener. See [CloudFront origin protection](#cloudfront-origin-protection-defense-in-depth).          |
| Static UI delivery with private origin access              | Implemented | Private encrypted object store; no public object access; served only through the edge distribution.                                                  |
| Internal service authentication and authorization          | Implemented | Application-layer JWT validation and service authorization on protected calls.                                                                       |
| Runtime isolation and non-root container baseline          | Implemented | Private-subnet container runtime with non-root execution baseline.                                                                                   |
| Edge access logging                                        | Implemented | Access logs for the public edge and related content stores.                                                                                          |
| Internal traffic encryption (HTTPS; mTLS optional)         | Implemented | Opt-in HTTPS on the internal service path; mTLS / mesh remain extensible.                                                                            |
| Customer-managed KMS key strategy                          | Implemented | Opt-in customer-managed keys (platform-provisioned or BYOK) with documented scope exceptions.                                                        |
| ALB access logging baseline                                | Implemented | Optional with platform governance evidence; omit when operators use org-wide evidence stores.                                                        |
| CloudTrail baseline                                        | Implemented | Optional regional trail (including global IAM/STS events); omit when a landing-zone org trail already covers the account.                            |
| Immutable CloudTrail evidence archive                      | Implemented | Object Lock COMPLIANCE archive with stage-aware retention; key management aligns with the CMK strategy.                                              |
| SIEM integration reference pattern                         | Documented  | Operator extension; not part of the product baseline.                                                                                                |

## IAM wildcard policy

Audit date: 2026-06-17

### Validation result

- No repository-authored IAM statement uses `Action: "*"` or `NotAction`.
- Wildcards are limited to resource patterns where AWS requires account-scope permissions or where controlled naming patterns are required.

### Exception categories

Bounded wildcard usage appears only where AWS APIs require it:

- **ECR token retrieval** — account-scoped authorization token APIs
- **SES send operations** — account-scoped email send APIs
- **CloudFormation stack exports** — account-scoped listing used by CI/CD workflows
- **Secrets Manager ARN matching** — suffix patterns required by AWS secret ARN format for least-privilege task and CI roles

Each exception is documented with rationale in infrastructure code and cdk-nag suppressions where applicable.

### Ongoing review expectation

Any new IAM wildcard usage must include one of the following:

- an AWS API constraint that prevents narrower resource scoping, or
- a deterministic naming constraint with bounded prefix or suffix matching.

Each case should include an inline rationale and corresponding cdk-nag suppression reason where applicable.

## Further reading

- [Architecture decision records](/docs/adrs)
- [Observability architecture](/docs/observability) — governance evidence and operator extensions
- [Operations](/docs/operations) — deployment defaults and platform configuration
- [Development](/docs/development) — platform configuration bootstrap
- [Runbook §9: Enable internal ALB HTTPS](/docs/runbook#9-enable-internal-alb-https)
- [Platform features — licence tiers](/docs/features#licence-tiers-and-platform-configuration)
- [ADR-0022: Public ALB edge and origin protection](/docs/0022-public-alb-edge-and-origin-protection)
- [ADR-0024: Internal ALB TLS (east-west, optional)](/docs/0024-internal-alb-tls-east-west-optional)
- [ADR-0025: Static UI CloudFront S3](/docs/0025-static-ui-cloudfront-s3)
- [ADR-0027: Governance evidence architecture](/docs/0027-governance-evidence-architecture)
- [ADR-0028: Governance evidence Object Lock](/docs/0028-governance-evidence-object-lock)
- [ADR-0029: Customer-managed KMS per domain](/docs/0029-customer-managed-kms-per-domain)
- [ADR-0011: Stateless JWT authentication](/docs/0011-stateless-jwt-authentication)
- [AWS Well-Architected Framework — Security pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html) — AWS Documentation
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — OWASP Foundation
- [OWASP API Security Top 10](https://owasp.org/API-Security/) — OWASP Foundation
