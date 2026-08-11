---
title: "0026. Observability backend strategy"
summary: "Backbone adopts OpenTelemetry instrumentation with environment-specific backends: AMP and AMG for application metrics and dashboards, X-Ray for traces, CloudWatch for logs and infrastructure alarms — without collector sidecars on Fargate tasks."
---

**Status:** Accepted  
**Date:** 2026-06-22  
**Supersedes:** [0007. Observability strategy](/docs/0007-observability-strategy)

## **Context**

Backbone runs GraalVM-native Quarkus services on AWS ECS Fargate at deliberately small task sizes. Operators need production-grade visibility into application behaviour, cross-service request flows, and platform health — not only container liveness.

The platform already instruments services with Micrometer and OpenTelemetry for local development. [ADR-0007](/docs/0007-observability-strategy) did not define an observability backend strategy for metrics, traces, dashboards, and infrastructure monitoring. A decision is required before deploying STAGE and PROD.

Application audit logging (business events) remains a separate concern from operational telemetry. Governance evidence (CloudTrail, load-balancer access logs, protected evidence buckets) is defined in [ADR-0027](/docs/0027-governance-evidence-architecture).

## **Decision**

Backbone will use **OpenTelemetry as the instrumentation standard** across all environments. **Backend targets vary by environment and by signal type** — metrics and traces follow separate AWS-recommended pipelines.

### Signal backends (deployed STAGE and PROD)

| Signal                | Backend                                                                        |
|-----------------------|--------------------------------------------------------------------------------|
| Application metrics   | Amazon Managed Prometheus (AMP)                                                |
| Application traces    | AWS X-Ray (via the AWS OpenTelemetry OTLP endpoint)                            |
| Application logs      | Amazon CloudWatch Logs                                                         |
| Dashboards            | Amazon Managed Grafana (AMG), with AMP, CloudWatch Logs, and X-Ray datasources |
| Infrastructure alarms | Amazon CloudWatch metrics and SNS                                              |

### Environment posture

| Environment           | Application telemetry                                                                                                                                           |
|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Local development** | Prometheus, Jaeger, and Grafana (existing Compose stack)                                                                                                        |
| **INT**               | Instrumentation present; **exporters disabled** and **no AMP/AMG stack deployed** — INT is for integration testing, not observability cost or operational noise |
| **STAGE / PROD**       | In-process export to the backends above                                                                                                                         |

### Architectural constraints

- **No OpenTelemetry Collector sidecar** (or shared collector service) on Fargate tasks. Export runs **in-process** from the application container.
- Backbone's observability architecture is designed to operate within the existing Fargate task sizing, avoiding additional resource reservations for collector sidecars.
- **Infrastructure metrics and alarms** (load balancers, databases, WAF, Lambdas, network flow logs) use **CloudWatch and SNS**, not AMP. AMP is reserved for application metrics.

Configuration — endpoints, credentials, and profile switches — is environment-specific. Application code does not fork between local and deployed runs.

## **Consequences**

### Positive

- Aligns with AWS guidance: AMP for metrics, X-Ray for traces, AMG as the unified operator surface.
- Preserves minimal Fargate footprint and cost profile.
- INT stays lean: no managed observability stack or export charges.
- Clear separation between application telemetry, infrastructure monitoring, and governance evidence — see [ADR-0027](/docs/0027-governance-evidence-architecture).
- Local developers retain a full metrics-and-traces loop without AWS dependencies.

### Negative

- STAGE and PROD operators depend on AMP, AMG, and X-Ray availability and pricing.
- INT cannot validate end-to-end observability pipelines; that validation belongs in STAGE.
- In-process export ties telemetry behaviour to application release and resource limits; there is no collector buffer for back-pressure or advanced tail sampling without a future architectural change.

If Backbone later requires capabilities such as centralized tail sampling, multi-destination export, or telemetry transformation, this decision should be revisited and an OpenTelemetry Collector re-evaluated.

## **Alternatives considered**

### ADOT Collector as a per-task sidecar

Rejected. A collector sidecar would materially increase the minimum resource allocation per task and undermine the platform's native-image and right-sized-task optimisations. Direct in-process export avoids that overhead.

### Shared OpenTelemetry Collector service

Rejected. No current requirement for collector-only capabilities (tail sampling, multi-destination fan-out, PII scrubbing in a central hop). Adds operational surface without a stated need.

### Application traces to AMP

Rejected. AWS positions AMP for metrics and X-Ray (via OTLP) for traces.

### CloudWatch as the application metrics backend

Rejected for application metrics. AMP with PromQL in AMG is the mature, recommended path for Prometheus-compatible application metrics at scale. CloudWatch remains the right choice for **infrastructure** metrics and alarms.

### Full observability stack on INT

Rejected. INT exists to validate integrations, not to host production-like monitoring infrastructure or incur AMP/AMG cost.

## **References**

- [ADR-0007: Observability strategy](/docs/0007-observability-strategy) — superseded
- [Observability architecture](/docs/observability) — architecture guide
- [Application telemetry](/docs/application-telemetry) — architecture guide
