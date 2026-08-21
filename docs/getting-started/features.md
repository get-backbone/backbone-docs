---
title: "Platform features"
summary: "Discrete inventory of Backbone platform capabilities across identity, operations, domain services, delivery, and governance."
---

## Identity and security

- service-to-service authentication (JWT / Cognito service pool)
- user authentication (Cognito JWT, email/password, first-party UI)
- user authentication (Google OAuth2)
- user authentication (LinkedIn OAuth2 + account linking)
- registration / token refresh / password reset
- service-level authorization (named caller allowlists)
- zero-trust request model (stateless JWT, no server sessions)
- secrets management (SSM / encrypted config; no long-lived AWS keys in CI)
- GitHub Actions OIDC to AWS deploy roles
- edge protection (CloudFront + WAF: host allowlist, geo, rate rules)
- origin protection (ALB locked to CloudFront + verify secret)
- private ECS networking / VPC per environment
- optional internal ALB TLS (east-west)
- DNSSEC on Route53 hosted zones
- CSP / XSS posture for browser token storage

## Traffic control and resilience

- application throttling / rate limiting (Redis-backed, distributed)
- edge throttling (CloudFront WAF)
- distributed caching (ElastiCache Redis; cache-aside)
- circuit breakers / fault tolerance on persistence paths
- dependency-aware health / readiness (Postgres, DynamoDB, S3, Cognito, SES, Redis)

## Observability

- Prometheus metrics (HTTP, JVM, domain ops, throttle, DB, circuit breakers)
- Grafana dashboards (local/lab)
- Amazon Managed Prometheus remote write (SigV4)
- distributed tracing (OpenTelemetry to AWS X-Ray OTLP, SigV4)
- structured / correlated logging (MDC, `X-Correlation-Id`)
- PII / sensitive-data log masking
- CloudWatch + SNS infrastructure monitoring / alerting
- governance evidence hooks (edge/ALB access logs when enabled)

## Domain services

- auth-service (identity lifecycle)
- actor-service (user profiles + caching)
- audit-service (central ingest + Postgres persistence)
- declarative cross-cutting audit emission
- notification-service (async email via SES, templates, retries, unsubscribe tokens)
- document-service (upload/storage + parsing across SQL / DynamoDB / S3)
- template-service
- BFF orchestration (actor / admin persona APIs)
- stateless reference UI (no client-side state; HATEOAS-aware; static CloudFront/S3 path)

## Data and messaging

- PostgreSQL (profiles, audit, notifications, relational workloads)
- DynamoDB (high-throughput / document-oriented shapes)
- S3 object storage
- SES email delivery
- reactor / messaging integration (shared libs)
- EventBridge audit routing (roadmap path)

## Platform delivery and infra (AWS)

- AWS CDK IaC (VPC, DNS, certs, WAF, ALB, ECS Fargate, ECR, datastores)
- ECS/Fargate deploy model + autoscaling
- GraalVM native image builds (with JVM fast-jar escape hatch)
- GitHub Actions CI/CD spine (hygiene → build/test → static analysis → coverage → version → ECR → env deploy)
- multi-env deploy workflows (int / stage / prod)
- non-prod infra hibernate (cost shed)
- optional self-hosted GHA runners (native builds)
- docs mirror to public ReadMe (`docs.backbonehq.io`)
- [k6 performance test framework](/docs/perf-test-plan) and [published baselines](/docs/performance)

## Governance and compliance foundations

- audit trails
- compliance control mapping (SOC 2 / GDPR / HIPAA-aligned objectives; operator-owned certification)
- compliance roadmap / shared-responsibility model
- governance evidence architecture
- least-privilege IAM posture documentation
- [Architecture Decision Records (ADRs)](/docs/adrs)

## Developer experience

- Floci-powered local development
- Taskfile / ops automation plane
- opinionated Quarkus/Java service layout (clean architecture)
- OpenAPI (e.g. actor-bff)
- Renovate / dependency hygiene workflows
- architecture guides

## Shared platform ([backbone-kit](https://github.com/get-backbone/backbone-kit))

Open-source libraries for observability, throttling, health, and small cross-cutting APIs - so product repos stay about domain, not plumbing:

| Module | What it brings |
|--------|----------------|
| [backbone-metrics](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-metrics) / [backbone-metrics-api](https://github.com/get-backbone/backbone-kit/tree/main/backbone-api/backbone-metrics-api) | Declarative application metrics (operations, DB, resilience, throttles) |
| [backbone-health-aws](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-health-aws) | Readiness for AWS-shaped dependencies (Postgres, DynamoDB, S3, Cognito) |
| [backbone-http-aws](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-http-aws) | SigV4-signed HTTP transport (AMP, X-Ray OTLP, other AWS endpoints) |
| [backbone-observability-api](https://github.com/get-backbone/backbone-kit/tree/main/backbone-api/backbone-observability-api) / [backbone-observability-aws](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-observability-aws) | AMP metrics push (Prometheus remote write) and X-Ray trace export (OTLP + SigV4) |
| [backbone-throttle](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-throttle) | Rate limiting primitives and metrics-friendly integration |
| [backbone-security](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-security) | Reusable JWT building blocks (Cognito claims extractors ordered in Backbone) |
| [backbone-logging](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-logging) | Method-entry logging, correlation IDs, PII / sensitive-data masking |
| [backbone-common](https://github.com/get-backbone/backbone-kit/tree/main/backbone-impl/backbone-common) | Shared REST, validation, and error-handling utilities |
