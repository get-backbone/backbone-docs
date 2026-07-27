---
title: "0007. Observability Strategy"
summary: "0007. Observability Strategy"
---

**Status:** Outdated  
**Date:** 2025-11-12  
**Superseded by:** [0026. Observability backend strategy](/docs/0026-observability-backend-strategy)  
**Context:** Microservice architecture on AWS, starting local development. Need structured logging, distributed tracing, and metrics readiness.

## Current State

* **Logging:** JSON console logs using `quarkus-logging-json`, local only.
* **Tracing:** OpenTelemetry enabled via `quarkus-opentelemetry`; spans printed locally or ignored.
* **Metrics:** Not implemented yet; future integration planned.
* **Local Deployment:** Single service, logs/traces to console.

## Decision

* Use **JSON console logs** locally for structured output compatible with CloudWatch.
* Enable **OpenTelemetry tracing** in Quarkus; spans recorded locally, propagated automatically via HTTP headers.
* Future AWS deployment:
  * Logs → CloudWatch Logs via Fluent Bit / CloudWatch Agent.
  * Traces → AWS X-Ray via ADOT Collector (OTLP).
  * Metrics → CloudWatch via Micrometer (future).
* Keep Quarkus code unchanged between local and AWS environments; configuration switches endpoints only.

## Consequences

* Minimal local setup; easy to scale to multiple services.
* Observability (logs, traces, metrics) unified via OpenTelemetry.
* Cloud-ready architecture without code changes.

**References:**

* Quarkus OpenTelemetry: [https://quarkus.io/guides/opentelemetry](https://quarkus.io/guides/opentelemetry)
* AWS Distro for OpenTelemetry: [https://aws-otel.github.io/docs/](https://aws-otel.github.io/docs/)

---
