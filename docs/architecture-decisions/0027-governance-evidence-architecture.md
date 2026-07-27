---
title: "0027. Governance evidence architecture"
summary: "Backbone treats governance evidence as a distinct concern from application observability and infrastructure monitoring — durable, protected platform proof for investigation and attribution, not operational dashboards."
---

**Status:** Accepted  
**Date:** 2026-06-30  
**Related:** [0026. Observability backend strategy](/docs/0026-observability-backend-strategy)

## **Context**

Production AWS platforms must satisfy several operational concerns that look similar from a distance but serve different goals. They have different consumers, retention requirements, privacy implications, and query patterns. Blurring them leads to the wrong tool for investigations, compliance discussions, and cost control.

Backbone already addresses three of these concerns:

| Concern                       | Question                                 | Backbone capability                                                                                                        |
|-------------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| **Application observability** | How is my software behaving?             | Micrometer, OpenTelemetry, AMP, X-Ray, CloudWatch Logs, AMG — see [ADR-0026](/docs/0026-observability-backend-strategy) |
| **Infrastructure monitoring** | Is the infrastructure healthy?           | Per-stack CloudWatch alarms and SNS notifications                                                                       |
| **Application audit**         | What business action did a user perform? | Declarative audit interceptor and audit-service — see [Audit logging](/docs/audit)                                      |

A fourth concern was not yet architecturally defined:

| Concern                 | Question                                                                                    |
|-------------------------|---------------------------------------------------------------------------------------------|
| **Governance evidence** | What happened on the AWS control plane and at the platform edge — and can I prove it later? |

Without an explicit boundary, future contributors may reasonably ask:

- Why not send CloudTrail into Grafana or AMG?
- Why is governance in a different stack from observability?
- Why both audit-service and CloudTrail?
- Why is Security Hub not part of Backbone?
- Why are ALB access logs in S3 instead of CloudWatch Logs?

This ADR records the architectural decision. Implementation (BackboneGovernanceStack) follows from it.

Platform governance requires **durable evidence** of AWS API activity and edge HTTP traffic that is **independent of operational telemetry**. That evidence must remain trustworthy if operators, auditors, or incident responders question a change or access pattern months later. **This distinction forms the architectural boundary defined by this ADR.**

## **Decision**

Backbone treats **governance evidence** as a **distinct architectural concern** from application observability and infrastructure monitoring.

### Responsibility split

| Concern                   | Backends / stores                                           | Stack ownership                                  |
|---------------------------|-------------------------------------------------------------|--------------------------------------------------|
| Application observability | AMP, AMG, X-Ray, CloudWatch Logs                            | BackboneObservabilityStack (+ in-process exporters) |
| Infrastructure monitoring | CloudWatch metrics, SNS                                     | Per-stack monitoring providers                   |
| Application audit         | audit-service (PostgreSQL)                                  | Application tier                                 |
| **Governance evidence**   | CloudTrail, ALB access logs, **protected evidence storage** | **BackboneGovernanceStack**                         |

Governance evidence is collected for **investigation, attribution, and long-term proof** — not for operational monitoring or dashboards. The deliverable is **immutable, tamper-resistant proof**.

### BackboneGovernanceStack — initial scope

| Capability           | Role                                                                                                                                                                                                                                                    |
|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **CloudTrail**       | Backbone baseline capability when `governanceEvidenceEnabled` is true. Records AWS API activity in the **deployment region** (not multi-region) plus global IAM/STS events. Log file integrity validation enabled.                                         |
| **ALB access logs**  | Backbone baseline capability when governance evidence is enabled. Per-request edge evidence (client IP, TLS, status, latency, target group). Supports incident response and origin-bypass analysis. Volume can be large — lifecycle policies are required. |
| **Evidence storage** | First-class concern: dedicated evidence repository (implementation: S3 buckets separate from application data), SSE-KMS encryption, versioning, restrictive access policies, lifecycle / retention / archival transition.                               |

BackboneGovernanceStack is the **golden path** for forked deployments: operators receive governance evidence by default and can set `governanceEvidenceEnabled: false` in `platform-config.yml` when an existing landing zone or org-wide trail already provides control-plane evidence. Register the stack in platform `STACK_LIFECYCLE` like other long-lived foundation stacks.

### Account scope — not Backbone-only evidence

CloudTrail records **account-level** API activity. Backbone cannot filter trails to CloudFormation stacks or Backbone resources only. With governance enabled, the trail captures management events in the stack **deployment region** (plus global IAM/STS via `includeGlobalServiceEvents`); it does **not** follow a multi-region deployment model Backbone has not implemented yet.

| Operator pattern                                           | Recommendation                                                                                                   |
|------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Dedicated workload account (greenfield / sandbox)          | Keep `governanceEvidenceEnabled: true` — trail noise stays low.                                                  |
| Shared account with unrelated workloads in the same region | Accept broader logs or use a dedicated account.                                                                  |
| Enterprise landing zone with org CloudTrail → log archive  | Set `governanceEvidenceEnabled: false`; use centralized evidence and re-point ALB logging in a fork if required. |

Backbone operates at the application platform layer, not as a landing zone. See [Operations — Governance evidence](/docs/operations#3-governance-evidence-backbonegovernancestack).

### Three independent evidence streams (do not merge)

| Stream                    | Answers                                    |
|---------------------------|--------------------------------------------|
| Application audit service | What business action did the user perform? |
| CloudTrail                | What AWS API activity occurred?            |
| ALB access logs           | What raw HTTP reached the load balancer?   |

These streams serve different investigative purposes and therefore remain architecturally independent.

### Common operator extensions

Backbone operates at the **application platform layer**, not as an AWS landing zone. The following are **out of Backbone Core scope**. Operators may add them in their own accounts or security foundations:

- AWS Config (configuration history — complements CloudTrail; answers *what changed*, not only *who changed*)
- GuardDuty, Security Hub, Inspector, Detective, IAM Access Analyzer
- Organization-wide log aggregation, central security accounts, SIEM integration
- VPC Flow Logs and CloudFront / WAF full access log archives (WAF *monitoring* via alarms remains in infrastructure monitoring)

### Explicit non-goals

- Backbone Core will not grow into a catalogue of account-level security services.
- Governance evidence ADRs and guides will not anchor on specific compliance frameworks (SOC 2, ISO 27001, PCI, etc.). Requirements evolve; the boundary above remains valid regardless of which questionnaire a customer uses.

## **Consequences**

### Positive

- Clear separation of responsibilities across observability, infrastructure monitoring, audit, and governance.
- Easier diligence conversations: each concern maps to a named capability and stack.
- Observability and governance can evolve independently (backends, retention, cost).
- Evidence protection (encryption, versioning, integrity validation) is a deliberate design goal, not an afterthought on generic storage.
- Reduces risk of operational tooling (dashboards, alarms) being mistaken for forensic or control-plane proof.

### Negative

- Additional stack and evidence storage to operate.
- Investigations may require correlating across multiple evidence sources (traces, audit events, CloudTrail, ALB logs).
- Governance evidence storage has ongoing cost; ALB log volume in particular requires lifecycle and archival discipline.

## **Alternatives considered**

### Unified "observability" stack for all signals

Rejected. Operational telemetry (metrics, traces, live logs, dashboards) and governance evidence (immutable API and edge archives) have different lifecycles, consumers, and protection models. A single stack encourages the wrong queries (e.g. CloudTrail in Grafana) and complicates ownership.

### CloudTrail or ALB logs to CloudWatch Logs / AMG

Rejected for governance evidence. CloudWatch Logs and AMG suit **operational** application and infra log analysis. Governance evidence requires **durable, independently protected evidence storage** with integrity validation (CloudTrail) and cost-aware lifecycle for high-volume HTTP access logs (ALB). Operators may still use Logs Insights for application logs without conflating that with control-plane evidence.

### AWS Config, Security Hub, or GuardDuty inside Backbone Core

Rejected. These are account- or organization-level security services. Document them as common operator extensions; do not provision them as part of the application platform baseline.

### VPC Flow Logs and CloudFront / WAF full access logs in baseline

Deferred. Useful for mature security programmes but expensive, noisy, or redundant with existing WAF *monitoring*. Operators can enable when threat model requires full archive, not only alarms.

### Application audit folded into CloudTrail

Rejected. Business audit events (login, registration, sensitive actions) are application-domain records with their own schema, retention, and consumers. CloudTrail records AWS API calls. Both are required; they are not substitutes.

## **References**

- [ADR-0026: Observability backend strategy](/docs/0026-observability-backend-strategy)
- [Audit logging](/docs/audit) — application audit
- [Platform security posture](/docs/security) — governance vocabulary
- Production observability phases plan — Phase 6 (BackboneGovernanceStack implementation)
