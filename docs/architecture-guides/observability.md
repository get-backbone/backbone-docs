---
title: "Observability architecture"
summary: "How Backbone separates application observability, infrastructure monitoring, application audit, and governance evidence — for architecture review and technical diligence."
---

Backbone platforms answer four distinct operational questions. Although they all involve collecting operational data, they serve different purposes, have different consumers, and require different retention and protection models. Backbone deliberately keeps these concerns separate — especially governance evidence and operational observability.

This guide is the architectural map. It explains how Backbone thinks about operational capabilities — not how to deploy or configure them. Depth guides: [Application telemetry](/docs/application-telemetry), [Infrastructure monitoring](/docs/monitoring), [Audit](/docs/audit). Deployment defaults are in [Operations](/docs/operations).

## Four concerns

| Concern                       | Question                                                                                    | Backbone provides                                                          | Typical implementation (AWS)                     |
|-------------------------------|---------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|--------------------------------------------------|
| **Application observability** | How is my software behaving?                                                                | Instrumentation, export, and dashboards across all services             | AMP, X-Ray, CloudWatch Logs, AMG                 |
| **Infrastructure monitoring** | Is the infrastructure healthy?                                                              | Per-stack CloudWatch dashboards and alarms across the deployed platform | CloudWatch metrics → SNS                         |
| **Application audit**         | What business action did a user perform?                                                    | Declarative audit capture to a central audit service                    | PostgreSQL (audit-service store)                 |
| **Governance evidence**       | What happened on the AWS control plane and at the platform edge — and can I prove it later? | Protected evidence collection for control-plane and edge HTTP traffic   | Protected object storage (encryption, lifecycle) |

**Governance evidence is separate from operational dashboards.** It is durable, tamper-resistant proof stored for investigations — not another datasource in AMG.

### Application observability

Deployed services export metrics, traces, and structured logs to managed AWS backends. Logs, traces, and metrics share correlation identifiers so operators can move between dashboards, traces, and log lines during live investigations — **operational troubleshooting**, not long-term governance archives. Signal depth is in [Application telemetry](/docs/application-telemetry).

### Infrastructure monitoring

Backbone deliberately separates application telemetry from infrastructure health monitoring. Dashboards and alarms cover runtime, datastores, edge delivery, identity, and email — delivered via CloudWatch and SNS, not through the application metrics backend. Coverage and scope boundaries are in [Infrastructure monitoring](/docs/monitoring).

### Application audit

Application audit records business-domain events such as authentication, registration, sensitive data access, and notification activity. Unlike infrastructure monitoring or governance evidence, these records describe application behaviour from a business perspective and are consumed by compliance, customer support, and investigations. See [Audit](/docs/audit).

### Governance evidence

Backbone's governance evidence capability uses CloudTrail, load-balancer access logs, and protected evidence storage. Deployments may disable this capability when equivalent organization-wide evidence stores already exist. Deployment scope, account boundaries, and flag defaults are in [Operations — Governance evidence](/docs/operations#4-governance-evidence-backbonegovernancestack) and [ADR-0027](/docs/0027-governance-evidence-architecture).

### Three independent audit and evidence streams (do not conflate)

| Stream                    | Answers                                                         | Consumers                                   |
|---------------------------|-----------------------------------------------------------------|---------------------------------------------|
| Application audit service | Login, registration, sensitive data access, notification events | Compliance, product support, investigations |
| CloudTrail                | Who changed IAM, RDS, security groups? CDK vs console?          | Security, platform ops                      |
| ALB access logs           | What raw HTTP reached the load balancer?                        | Incident response, abuse analysis           |

## Environment posture

| Environment     | Application observability         | Infrastructure monitoring | Governance evidence          |
|-----------------|-----------------------------------|---------------------------|------------------------------|
| **Local**       | Local metrics and tracing tooling | n/a                       | n/a                          |
| **INT**         | Lightweight logging only          | Alarms deploy             | Off by default               |
| **TEST / PROD** | Full managed observability        | Alarms deploy             | On by default (configurable) |

INT is optimized for integration testing rather than production-scale operational visibility. Deployment defaults and environment-specific behaviour are documented in [Operations](/docs/operations). See [ADR-0026](/docs/0026-observability-backend-strategy) for backend detail.

## Governance evidence vs application observability

|                   | Application observability            | Governance evidence                                |
|-------------------|--------------------------------------|----------------------------------------------------|
| **Purpose**       | Live health, latency, errors, traces | Immutable proof for investigations and attribution |
| **Consumers**     | On-call, SRE, engineering            | Security, audit, incident response                 |
| **Typical query** | "Why is p99 high?"                   | "Who changed this security group?"                 |

## Common operator extensions

Backbone operates at the **application platform layer**, not as an AWS landing zone. Operators may add:

| Extension                                         | Complements                                                        |
|---------------------------------------------------|--------------------------------------------------------------------|
| **AWS Config**                                    | CloudTrail (configuration history: *what* changed, not only *who*) |
| **Org-wide CloudTrail → log archive**             | Replace or supplement Backbone governance evidence                    |
| **GuardDuty, Security Hub, Inspector, Detective** | Account-level threat detection                                     |
| **SIEM / central security account**               | Cross-account correlation                                          |
| **VPC Flow Logs**                                 | Network forensics (volume/cost sensitive)                          |
| **CloudFront / WAF full access log archives**     | Beyond WAF alarms already in infrastructure monitoring             |

## Further reading

- [Application telemetry](/docs/application-telemetry) — metrics, traces, and logs
- [Infrastructure monitoring](/docs/monitoring) — CloudWatch dashboards and alarms
- [ADR-0026: Observability backend strategy](/docs/0026-observability-backend-strategy)
- [ADR-0027: Governance evidence architecture](/docs/0027-governance-evidence-architecture)
- [AWS Well-Architected Framework — Operational Excellence pillar](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html) — AWS Documentation
- [Security best practices for AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html) — AWS Documentation
