---
title: "AWS platform infrastructure"
summary: "How Backbone is deployed on AWS — networking, compute, data, edge delivery, and environment isolation."
---

Backbone is deployed into **AWS accounts owned by the operator**. Infrastructure is defined as code (AWS CDK in TypeScript), provisioned deterministically per environment, and designed for container-native workloads on ECS Fargate.

Backbone is **not a hosted SaaS control plane**. The operator owns the account, networking, data, and runtime configuration.

This guide is for architecture review, procurement, and technical due diligence. Day-to-day CDK commands live in [Operations](/docs/operations) and [Runbook](/docs/runbook) documentation.

## Design goals

The infrastructure model prioritizes:

- **Predictable deployments** — repeatable stacks per environment (DEV, INT, TEST, PROD)
- **Horizontal scale** — stateless ECS services behind load balancers
- **Private-by-default** — datastores and tasks in private subnets; controlled public edge
- **Least privilege** — scoped IAM, security groups, and secret access
- **Progressive hardening** — baseline suitable for mature deployments; room to add enterprise controls

Backbone intentionally avoids mandating a specific AWS Organizations layout. Operators can integrate the platform into existing landing zones.

> **Single region today:** Each environment deploys to one AWS region with multi-AZ patterns inside that region. Multi-region failover is a roadmap consideration, not the current baseline.

## High-level architecture

![Backbone platform architecture - edge delivery, the VPC public/private subnet boundary, data tier, identity, and observability](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/forge-architecture.png)

Browser traffic hits **CloudFront** at the environment apex hostname. Static UI is served from **S3**. API paths forward to a **regional internet-facing ALB** (CloudFront origin only) with origin verification. Application tasks run in **private subnets** and reach datastores over internal networking.

## Environment composition

Each Backbone environment (e.g. INT, PROD) is a set of coordinated CDK stacks:

| Stack (conceptual)         | Purpose                                                                      |
|----------------------------|------------------------------------------------------------------------------|
| **Network**                | VPC, public/private subnets, optional NAT, VPC endpoints                     |
| **Domain & DNS**           | Route 53 hosted zone, certificates, email domain identity                    |
| **Static edge (global)**   | CloudFront certificate and edge WAF (us-east-1)                              |
| **Static edge (regional)** | S3 UI bucket, CloudFront distribution, API origin to ALB                     |
| **Identity**               | Cognito user pools (human + service accounts)                                |
| **Datastores**             | RDS PostgreSQL, DynamoDB, S3 buckets, ElastiCache Redis                      |
| **Security**               | WAF on ALB where applicable, security groups, IAM roles, secrets integration |
| **Runtime**                | ECS cluster, Fargate services, load balancers, routing rules                 |
| **CI/CD role (INT)**       | GitHub Actions OIDC role for prescribed integration environment              |

Stateful resources run inside **private networking boundaries**. Stateless application services scale horizontally by adding ECS tasks.

## Container and release model

- **One shared container registry** per account/region holds versioned/tagged images for all services and environments.
- **Immutable build tags** trace each deployment to source; **mutable service tags** reference what ECS runs.
- **Native or JVM runtime** — operators can switch ECS task runtime mode (GraalVM native vs JVM) via deployment configuration without infrastructure redesign. See [Runbook](/docs/runbook).

Images are built and published through GitHub Actions; deployment workflows update ECS task definitions.

## Shared platform services

| Service                 | Role                                                       |
|-------------------------|------------------------------------------------------------|
| **PostgreSQL (RDS)**    | Relational data — actors, notifications, audit events      |
| **DynamoDB**            | Document metadata, notification templates, delivery state  |
| **S3**                  | Document objects, static UI assets, access logs            |
| **Redis (ElastiCache)** | Distributed cache and rate-limit counters across ECS tasks |
| **Cognito**             | Human and service identity                                 |
| **SES**                 | Outbound email (via notification service)                  |

See [Application caching and distributed scale](/docs/caching) and [Rate limiting](/docs/rate-limiting) for how Redis is used.

## Networking and security

- **Public edge:** CloudFront with WAF (hostname allowlist, rate limits, geo controls).
- **Origin protection:** CloudFront-to-ALB verification header; ALB security groups restrict source to CloudFront edge ranges.
- **Internal traffic:** Service-to-service calls use an internal load balancer; JWT validation at the application layer (not network trust alone).
- **Optional internal TLS:** Documented as a hardening step in [ADR-0024](/docs/0024-internal-alb-tls-east-west-optional).

Full edge and IAM posture: [Platform security posture](/docs/security).

## Baseline cost expectations (non-production)

Representative **idle** monthly cost in us-west-2 for a single-replica, minimal-endpoint footprint is on the order of **~USD 150/month**, dominated by Fargate tasks, interface VPC endpoints, and load balancers. Actual costs vary with traffic, NAT usage, RDS storage, and endpoint AZ mode.

| Component (indicative)            | Approx. monthly |
|-----------------------------------|----------------:|
| Fargate workloads (6 small tasks) |            ~$54 |
| Interface VPC endpoints           |            ~$50 |
| Two ALBs                          |            ~$32 |
| Edge WAF + CloudFront + S3 (idle) |             ~$9 |
| Logging, secrets, DNS             |             ~$4 |

Validate production assumptions with the AWS Pricing Calculator. Gateway endpoints (S3, DynamoDB) do not incur hourly charges; **interface endpoint AZ spread** is often the largest hidden cost lever in smaller environments.

## Availability posture

| Model                                 | Indicative availability | Notes                                 |
|---------------------------------------|-------------------------|---------------------------------------|
| Single AZ footprint                   | ~99.5–99.7%             | Lower cost; limited fault tolerance   |
| Multi-AZ Fargate + managed datastores | ~99.99%                 | Aligns with AWS managed service SLAs  |
| Multi-region                          | Roadmap                 | Requires explicit cross-region design |

Backbone supports progressive hardening without redesigning application services.

## Operator expectations

| Topic                 | Expectation                                                                                              |
|-----------------------|----------------------------------------------------------------------------------------------------------|
| **Account ownership** | Operator provisions and pays for all AWS resources                                                       |
| **Environments**      | DEV / INT / TEST / PROD are isolated deployments within operator control                                 |
| **Configuration**     | Domain, capacity, VPC topology, and feature toggles are operator-maintained                              |
| **Extensibility**     | New deployable services follow the platform service registry pattern in the private codebase             |
| **Operations**        | CDK deploy/destroy, runtime switching, and self-hosted CI runners documented in [Runbook](/docs/runbook) |

## Further reading

- [ADR-0022: Public ALB edge and origin protection](/docs/0022-public-alb-edge-and-origin-protection)
- [Runbook](/docs/runbook) — JVM/native runtime and operational procedures
- [Amazon ECS best practices guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html) — AWS Documentation
