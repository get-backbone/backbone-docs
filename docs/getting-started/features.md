---
title: "Platform features"
summary: "Discrete inventory of Backbone platform capabilities across identity, operations, domain services, delivery, and governance."
---

See [Licence tiers and platform configuration](#licence-tiers-and-platform-configuration) for capability configuration.

## Identity and security

- Service-to-service authentication (JWT / Cognito service pool)
- User authentication (Cognito JWT, email/password, first-party UI)
- User authentication (Google OAuth2)
- User authentication (LinkedIn OAuth2 + account linking)
- Registration / token refresh / password reset
- Service-level authorization (named caller allowlists)
- Zero-trust request model (stateless JWT, no server sessions)
- Secrets management (Secrets Manager / SSM Parameter Store; no long-lived AWS keys in CI)
- GitHub Actions OIDC to AWS deploy roles
- Edge protection (CloudFront + WAF: host allowlist, geo, rate rules)
- Origin protection (ALB locked to CloudFront + verify secret)
- TLS-only viewers and HSTS at the public edge
- Private ECS networking / VPC per environment
- Runtime isolation (ECS Fargate, private subnets, non-root container user)
- Encryption at rest with AWS-managed keys (AMK)
- DNSSEC on Route53 hosted zones
- CSP / XSS posture for browser token storage
- [Growth + Enterprise licence tier] Customer-managed KMS (platform-provisioned CMKs per encryption domain)
- [Enterprise-only licence tier] Bring-your-own-key (BYOK) ARNs per encryption domain
- [Enterprise-only licence tier] Optional internal ALB TLS (east-west HTTPS on the internal listener)

## Traffic control and resilience

- Application throttling / rate limiting (Redis-backed, distributed)
- Edge throttling (CloudFront WAF)
- Distributed caching (ElastiCache Redis; cache-aside)
- Circuit breakers / fault tolerance on persistence paths
- Dependency-aware health / readiness (Postgres, DynamoDB, S3, Cognito, SES, Redis)
- [Enterprise-only licence tier] HA endpoint AZ mode (ALB and VPC interface endpoints spanning AZs)

## Observability

- Prometheus metrics (HTTP, JVM, domain ops, throttle, DB, circuit breakers)
- Grafana dashboards (local/lab)
- Distributed tracing instrumentation (OpenTelemetry)
- Structured / correlated logging (MDC, `X-Correlation-Id`)
- PII / sensitive-data log masking
- CloudWatch + SNS infrastructure monitoring / alerting
- [Growth + Enterprise licence tier] Optional managed observability (AMP remote write + AMG + X-Ray OTLP export)

## Domain services

- auth-service (identity lifecycle)
- actor-service (user profiles + caching)
- audit-service (central ingest + Postgres persistence)
- Declarative cross-cutting audit emission
- notification-service (async email via SES, templates, retries, unsubscribe tokens)
- document-service (upload/storage + parsing across SQL / DynamoDB / S3)
- template-service scaffolding enables RAD
- BFF orchestration (actor / admin persona APIs)
- Stateless reference UI (no client-side state; HATEOAS-aware; static CloudFront/S3 path)

## Data and messaging

- PostgreSQL (profiles, audit, notifications, relational workloads)
- DynamoDB (high-throughput / document-oriented shapes)
- S3 object storage
- SES email delivery
- Guaranteed audit message delivery
- EventBridge audit fan-out with SQS ingest (SIEM rules attach as operator extension)

## Platform delivery and infra (AWS)

- AWS CDK IaC (VPC, DNS, certs, WAF, ALB, ECS Fargate, ECR, datastores)
- ECS/Fargate deploy model + autoscaling
- GraalVM native image builds (with JVM fast-jar escape hatch)
- GitHub Actions CI/CD spine (hygiene → build/test → static analysis → coverage → version → ECR → env deploy)
- Multi-env deploy workflows (int / stage / prod)
- Non-prod infra hibernate (FinOps)
- Optional NAT Gateway for private-subnet egress
- Optional self-hosted GHA runners (native builds)
- [k6 performance test framework](/docs/perf-test-plan)

## Governance and compliance foundations

- Application audit trails
- Compliance control mapping (SOC 2 / GDPR / HIPAA-aligned objectives; operator-owned certification)
- Compliance roadmap / shared-responsibility model
- Least-privilege IAM posture documentation; exceptions documented (nag)
- Edge access logging (CloudFront and static-content stores)
- VPC reject flow logs (PROD; one-year retention)
- [Enterprise-only licence tier] Regional CloudTrail (deployment region plus global IAM/STS) with log-file integrity validation
- [Enterprise-only licence tier] Protected evidence storage (versioning, lifecycle, S3 Object Lock COMPLIANCE, stage-aware retention)
- [Enterprise-only licence tier] ALB access logs into a companion evidence bucket

Disable the CloudTrail, Object Lock, and ALB access-log capabilities when your organization already operates org-wide trails and log archives.

## Developer experience

- Floci-powered local development
- Taskfile ops automation plane and [cheatsheet](/docs/cheatsheet)
- Opinionated Quarkus/Java service layout (clean architecture)
- Maven dependency management (Quarkus/AWS BOM imports, aggregator-scoped pins, versionless leaf POMs)
- Optimised build and test wall-clock (Maven build cache, parallel reactor, parallel Surefire/Failsafe; CI typically under 10 minutes)
- OpenAPI (e.g. actor-bff)
- Renovate / dependency hygiene workflows
- [Architecture guides](/docs/security)
- [Architecture Decision Records (ADRs)](/docs/adrs)
- [Published performance envelopes](/docs/performance)

## Shared platform ([backbone-kit](https://github.com/get-backbone/backbone-kit))

Open-source libraries for observability, throttling, health, and small cross-cutting APIs - so product repos stay about domain, not plumbing:

| Module                                                                                                                                                                                                                                                       | What it brings                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| [backbone-metrics](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-metrics) / [backbone-metrics-api](https://github.com/get-backbone/backbone-kit/tree/main/backbone-api/backbone-metrics-api)                                 | Declarative application metrics (operations, DB, resilience, throttles)          |
| [backbone-health-aws](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-health-aws)                                                                                                                                              | Readiness for AWS-shaped dependencies (Postgres, DynamoDB, S3, Cognito)          |
| [backbone-http-aws](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-http-aws)                                                                                                                                                  | SigV4-signed HTTP transport (AMP, X-Ray OTLP, other AWS endpoints)               |
| [backbone-observability-api](https://github.com/get-backbone/backbone-kit/tree/main/backbone-api/backbone-observability-api) / [backbone-observability-aws](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-observability-aws) | AMP metrics push (Prometheus remote write) and X-Ray trace export (OTLP + SigV4) |
| [backbone-throttle](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-throttle)                                                                                                                                                  | Rate limiting primitives and metrics-friendly integration                        |
| [backbone-security](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-security)                                                                                                                                                  | Reusable JWT building blocks (Cognito claims extractors ordered in Backbone)     |
| [backbone-logging](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-logging)                                                                                                                                                    | Method-entry logging, correlation IDs, PII / sensitive-data masking              |
| [backbone-common](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-common)                                                                                                                                                      | Shared REST, validation, and error-handling utilities                            |

---

## Licence tiers and platform configuration

Your signed licence tier (`foundation`, `growth`, or `enterprise`) determines which platform configuration options the bootstrap procedure allows. Settings apply **per environment** (`DEV`, `INT`, `STAGE`, `PROD`) so integration stages can stay lean and production-like stages can take a fuller posture, keeping FinOps predictable.

| Configuration               | Foundation        | Growth            | Enterprise        |
|-----------------------------|-------------------|-------------------|-------------------|
| `vpcAzCount`                | max 2             | max 2             | unrestricted      |
| `endpointAzMode`            | `minimal`         | `minimal`         | `minimal` \| `ha` |
| `natEnabled`                | `true` \| `false` | `true` \| `false` | `true` \| `false` |
| `ecsDesiredCount`           | 1–2               | 1–3               | min 1, no max     |
| `internalHttps`             | `false`           | `false`           | `true` \| `false` |
| `customerManagedKms`        | AMK               | AMK, CMK          | AMK, CMK, BYOK    |
| `observabilityEnabled`      | `false`           | `true` \| `false` | `true` \| `false` |
| `governanceEvidenceEnabled` | `false`           | `false`           | `true` \| `false` |

**Managed observability** (`observabilityEnabled`) gates the AMP/AMG stack and application export to Prometheus remote write and X-Ray.

**Infrastructure monitoring** (CloudWatch dashboards and alarms to SNS) and **application logs** in CloudWatch Logs ship on every tier regardless.

Addition of a **NAT Gateway** for outbound internet access is a configurable option for all tiers (with cost-saving implications).

Encryption at rest defaults to AWS-managed keys (AMK). Growth adds Backbone-provisioned CMKs (CMK); Enterprise adds bring-your-own-key (BYOK) per domain.
