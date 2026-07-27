---
title: "Welcome"
summary: "What Backbone is, what you get out of the box, and where to start."
---

<!-- markdownlint-disable-file MD033 -->
<!-- MD033 off: inline HTML is used for spacing and image gallery layout where pure Markdown is insufficient. -->

The Backbone platform provides a pre-engineered operational foundation for secure, scalable distributed systems on AWS, dramatically reducing the time, cost, and risk of reaching production-grade operational maturity.

It encodes years of software and platform engineering decisions into a deployable operating model built on AWS and Quarkus.

Backbone is not a framework. It is a pre-engineered operational production system.

### What you get at a glance

- Security model: zero-trust and identity-first request verification (JSON Web Token, OpenID Connect).
- Runtime model: stateless, horizontally scalable services deployed as containers on ECS Fargate.
- Observability model: metrics, dashboards, health endpoints, tracing support.
- Delivery and infrastructure model: repeatable pipelines and environments (IaC, CI/CD, complete GitHub Actions workflow pipeline).
- Developer experience: a complete local development environment that emulates AWS via LocalStack.

For the full capability breakdown across engineering, security, observability, data, and cloud, see [Platform features](/docs/features).

<br />

---

## Who Backbone is for

Backbone is designed for engineering teams building production systems where security, scalability, and operational maturity must be correct from day one.

Typical adopters include:

- Teams building backend platforms or SaaS (Software as a Service) products on AWS that need security and operational maturity from day one.
- Organizations standardizing on Java, Maven, and Quarkus for service development.
- Founders shaping engineering organisations around proven platform patterns, without needing to build a large internal platform engineering function.
- Product teams accelerating time-to-market while reducing architectural uncertainty and operational risk.

Backbone is a strong fit when the cost of getting platform decisions wrong is high: security posture, deployment model, observability model, and identity boundaries need to be coherent from the outset.

<br />

---

## How Backbone works

- Purchase a Backbone platform licence file from the [Backbone website](https://backbonehq.io/#pricing).
- Your organization is added as a Contributor, so you can fork the `backbone-platform` repository.
- You own and develop that forked codebase.
- Provision the provided CI/CD workflows in your GitHub account.
- Deploy into your AWS accounts. Development environments can run within AWS free tier limits.
- Receive future platform updates by syncing your fork to the upstream `backbone-platform` repository.

You retain full ownership and control of infrastructure, services, deployments, and data.

<br />

---

## Start here

Most readers want the operating model first. These guides provide the fastest path to understanding how Backbone is built, how it runs, and how to extend it safely.

| Guide                                     | Why read it                                                                                          |
|-------------------------------------------|------------------------------------------------------------------------------------------------------|
| [DEVELOPMENT.md](/docs/development)       | Local development, tooling, Quarkus workflow.                                                        |
| [OPERATIONS.md](/docs/operations)         | Deployments, GitHub OIDC (OpenID Connect), AWS, LocalStack, CDK.                                     |
| [SECURITY.md](/docs/security)             | Platform security posture, least privilege, documented trade-offs.                                   |
| [COMPLIANCE.md](/docs/compliance)         | Security and compliance control mapping for forked deployments, including operator responsibilities. |
| [PERFORMANCE.md](/docs/performance)       | Performance test plan, outputs, phase summaries and conclusions.                                     |
| [INFRASTRUCTURE.md](/docs/infrastructure) | AWS deployment topology — networking, compute, datastores, edge delivery, and cost expectations.     |

If you are evaluating Backbone, start with Security and Operations. Those documents show the platform's decisions in the open, including constraints and trade-offs.

<br />

---

## Security and compliance (overview)

Short summaries only. Full implementation detail, trade-offs, and shared-responsibility boundaries are documented in the linked guides.

### Platform security architecture (summary)

Backbone provides a production-oriented security baseline for running containerized distributed systems on AWS. Deployments run entirely inside AWS accounts owned and controlled by the operator; Backbone is not a managed SaaS control plane ([SECURITY.md](/docs/security)).

The platform is designed around identity-first security and least-privilege infrastructure patterns. Baseline capabilities include:

- JWT-based authentication and authorization for both users and services
- Application-layer authorization enforcement rather than trust-by-network-location
- Least-privilege IAM policies defined in infrastructure code
- ECS Fargate workloads isolated in private subnets
- Public ingress restricted to AWS ALBs protected by WAF controls
- TLS termination at the public edge
- Secrets managed through AWS Secrets Manager and Systems Manager Parameter Store
- GitHub Actions OIDC deployment flows without long-lived AWS deployment keys
- Repeatable AWS infrastructure provisioning through CDK
- Structured logging, metrics, and OpenTelemetry-based observability foundations

Backbone intentionally documents architectural boundaries and progressive hardening paths. Operators can extend the baseline with stronger transport security, centralized audit controls, organization-wide governance tooling, and environment-specific compliance controls without redesigning the underlying platform architecture.

See [SECURITY.md](/docs/security) for detailed coverage of trust boundaries, IAM posture, network architecture, workload isolation, logging strategy, and shared responsibility.

### Security and compliance alignment (summary)

Backbone is designed to support organizations building security-conscious and compliance-aware systems on AWS. [COMPLIANCE.md](/docs/compliance) maps repository capabilities to control objectives commonly associated with frameworks and programs such as SOC 2, GDPR, and HIPAA-aligned environments.

The platform provides a strong technical baseline including identity and access controls, infrastructure isolation, secrets management, audit-event foundations, observability hooks, and repeatable infrastructure deployment patterns.

Backbone itself does not claim compliance certification or attestation for client deployments. Each deployment is forked, extended, configured, and operated independently inside operator-owned AWS environments. Compliance outcomes therefore depend on the deployed system, operational processes, monitoring controls, legal agreements, and governance practices implemented by the operator organization.

Progress states in the compliance guide reflect the current repository implementation posture and are intended as transparent control mappings rather than audit findings or legal interpretations.

<br />

---

## Repository structure

The [Backbone platform](https://backbonehq.io/) consists of the following discrete repositories:

| Repository                                                     | Visibility | Description                                                                                                 |
|----------------------------------------------------------------|------------|-------------------------------------------------------------------------------------------------------------|
| [forge-kit](https://github.com/get-backbone/forge-kit)         | Public     | Reusable operational components for Quarkus services.                                                       |
| `backbone-core`                                                | Private    | Internal upstream platform source.                                                                          |
| `backbone-platform`                                            | Private    | Client-forkable distributable platform, filtered mirror of `backbone-core`.                                 |
| [backbone-docs](https://github.com/get-backbone/backbone-docs) | Public     | Public documentation source and asset host, published at [docs.backbonehq.io](https://docs.backbonehq.io/). |

`forge-kit` is open source and can be adopted independently in existing Quarkus services. It is also a working dependency of `backbone-core`.

<br />

---

## What you get

Out of the box, Backbone provides you with the following:

- A development environment built predominantly on free tier LocalStack that emulates AWS and spins up in seconds.

[![Local services](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-services-local.png)](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-services-local.png)

- An entire GitHub Actions pipeline which includes release automation; ECS deployments (diffed services only); infrastructure deployments (CDK); static code analysis (OWASP, SpotBugs); code coverage, unit/integration test reports, and more.

[![GitHub Actions workflows](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-github-workflows.png)](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-github-workflows.png)

- Full IaC support and repeatable automation for AWS environments, including thoughtful segregation of stateful vs stateless resources.

[![AWS CloudFormation stacks](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-sandbox-aws-cloudformation.png)](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-sandbox-aws-cloudformation.png)

- A clean, well-documented, and well-tested codebase that you can fork and modify.
  <br /><br />
- A stateless reference web application that you can deploy locally and to AWS and use immediately.

<p align="center">
  <a href="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-web-home.png">
    <img src="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-web-home.png" alt="Backbone reference web homepage" width="33%" />
  </a>
  <a href="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-web-login.png">
    <img src="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-web-login.png" alt="Backbone reference web login" width="33%" />
  </a>
  <a href="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-web-dashboard.png">
    <img src="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-web-dashboard.png"
      alt="Backbone reference web dashboard" width="33%" />
  </a>
</p>

- The following foundational services provide the base for you to build domain services on top of:
  - actor-service; canonical user profile and identity-linked domain data
  - audit-service; immutable event and action trail for compliance and observability
  - auth-service; JWT issuance, validation, and user/service authentication workflows
  - document-service; document metadata, storage orchestration, and retrieval APIs
  - notification-service; template-driven outbound messaging and delivery orchestration
      <br /><br />
- The following edge services that provide client-facing composition and delivery layers:
  - actor-bff; BFF (Backend for Frontend) orchestration tier
  - backend-web; disposable reference UI and consumable frontend
      <br /><br />
- Comprehensive Prometheus (metrics) and Grafana (dashboards) for observability.

<p align="center">
  <a href="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-metrics-dashboard.png">
    <img src="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-metrics-dashboard.png" alt="Backbone metrics dashboard" />
  </a>
  <br />
  <br />
  <a href="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-metrics-database.png">
    <img src="https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-metrics-database.png" alt="Backbone database metrics" />
  </a>
</p>

For the complete list of platform features, see the [FEATURES.md](/docs/features) file.

<br />

---

## Build vs. Buy

Backbone exists to remove a class of problems that most teams eventually end up solving themselves. You can build this platform internally. Many teams do. But in practice, that path comes with trade-offs:

### Time

- Building a production-ready foundation like this typically takes multiple years.
- Progress is incremental and often delayed by competing business priorities.

### Focus

Your team splits attention between:

- domain features (what your business actually sells)
- platform engineering (infrastructure, security, reliability, operations)

This dilution slows both tracks.

### Cost

The true cost includes more than engineering time:

- iteration cycles
- operational mistakes
- rework as standards evolve

### Opportunity cost

Every month spent building foundations is a month not spent:

- shipping differentiating features
- validating your market
- generating revenue

### What Backbone changes

Backbone compresses that entire journey into something you can adopt immediately:

- A production-ready foundation from day one
- A clear operational model aligned with modern cloud practices
- A Quarkus-first golden path with flexibility where you need it
- A platform that lets your team stay focused on domain and business value

Instead of building the runway, you start further down it.

### When it makes sense

Backbone is a strong fit, if:

- You want to move quickly without building infrastructure from scratch
- Your team is domain-focused, not platform-heavy
- You value security, consistency, operability, and scale from the outset

If your goal is to invest heavily in building a bespoke internal platform, Backbone may be less relevant.

<br />

---

## How Backbone runs in production

Backbone is designed to run as a container-native platform on AWS, using a small number of well-understood building blocks.

The operating model prioritizes:

- security best-practices
- predictable operations
- clear system reasoning
- alignment with modern service deployment practices

At a high level, Backbone separates edge, services, and infrastructure concerns so each layer can scale and evolve independently.

### Architecture and runtime model

Traffic enters through the edge layer (CloudFront and WAF), is routed to stateless services behind load balancers, and those services rely on managed AWS infrastructure for data, identity, and messaging.

![Backbone platform architecture](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-architecture.png)

Key characteristics:

- **Private by default** — only the public load balancer sits in public subnets; all compute (ECS Fargate) and in-VPC data (RDS, ElastiCache Redis) run in private subnets, and regional services (S3, DynamoDB, Cognito, SES) are reached over VPC endpoints rather than the public internet.
- **Stateless and horizontally scalable** — services are packaged as containers and deployed on ECS Fargate without host management, scaling out by adding tasks.
- **Clear boundaries** — separating edge, service, and data concerns simplifies ownership, security reasoning, and independent evolution.
- **Deployment-shaped health** — standardized health, metrics, and operational endpoints keep "green" in the load balancer aligned with real readiness, on top of managed services that reduce operational overhead.

### What this means in practice

- You do not need to design your deployment model from scratch
- You inherit a setup aligned with enterprise production best practices
- Your team can focus on building services instead of building platform foundations

Backbone gives you a starting point that is usable immediately, secure, and built to scale.

<br />

---

## In-depth architecture guides

| Guide                                                     | Description                                                                |
|-----------------------------------------------------------|----------------------------------------------------------------------------|
| [SECURITY.md](/docs/security)                             | Platform security posture, edge controls, IAM, and shared responsibility.  |
| [COMPLIANCE.md](/docs/compliance)                         | Control mapping for forked deployments and operator responsibilities.      |
| [USER_AUTHENTICATION.md](/docs/user-authentication)       | Human sign-in, registration, and stateless JWT model.                      |
| [SERVICE_AUTHENTICATION.md](/docs/service-authentication) | Service-to-service identity and caller authorization.                      |
| [AUDIT.md](/docs/audit)                                   | Centralized audit logging for compliance traceability.                     |
| [NOTIFICATIONS.md](/docs/notifications)                   | Transactional email, queuing, and delivery lifecycle.                      |
| [INFRASTRUCTURE.md](/docs/infrastructure)                 | AWS stacks, networking, datastores, and baseline cost expectations.        |
| [CACHING.md](/docs/caching)                               | Distributed caching and shared Redis for multi-instance scale.             |
| [RATE_LIMITING.md](/docs/rate-limiting)                   | Edge and application HTTP rate limits and fail-closed posture.             |
| [CIRCUIT_BREAKERS.md](/docs/circuit-breakers)             | Fault tolerance on data-access paths.                                      |
| [Observability architecture](/docs/observability)         | How operational concerns are separated.                                    |
| [Infrastructure monitoring](/docs/monitoring)             | CloudWatch dashboards and alarms for platform health.                      |
| [Application telemetry](/docs/application-telemetry)      | Metrics, traces, and logs for dashboards, alerting, and capacity planning. |
| [HEALTH_CHECKS.md](/docs/health-checks)                   | Readiness and liveness probes for load balancer routing.                   |

<br />

---

## Architecture Decision Records (ADRs)

ADRs (Architecture Decision Records) are historical "why" records: context, trade-offs, and alternatives. The full public index, redaction scope, and links to each decision are in **[architecture/ADRs.md](/docs/adrs)**.

<br />

---

## Operational documentation (post-fork / deployment reference)

- [DEVELOPMENT.md](/docs/development) - local dev, tooling, licence, Quarkus
- [OPERATIONS.md](/docs/operations) - GitHub OIDC, Actions, CDK, AWS/LocalStack
- [CHEATSHEET.md](/docs/cheatsheet) - `task` index and copy-paste
- [RUNBOOK.md](/docs/runbook) - operational runbook
