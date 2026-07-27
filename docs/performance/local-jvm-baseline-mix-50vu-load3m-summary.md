---
title: "Performance Testing Baseline (Local Environment)"
summary: "This document captures the baseline performance characteristics of the Backbone"
---

## Overview

This document captures the baseline performance characteristics of Backbone under controlled local testing conditions. These results are not intended for publication. They serve as an internal reference point to:

- Validate system behavior under load
- Identify bottlenecks and instability
- Establish a benchmark before moving to AWS/ECS
- Inform future performance claims and comparisons

Raw k6 end-of-test output for this environment (baseline-mix, 50 VUs, 3 min warmup + 3 min load) is archived at [`local-jvm-baseline-mix-50vu-load3m-raw.txt`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_1/local-jvm-baseline-mix-50vu-load3m-raw.txt).

## Executive Summary

With this interim test result, it is defensible to say the system supports:

“~50 concurrent active users generating ~50–55 RPS sustained in mixed workflow testing, with stable latency and no systemic failure.”

---

## Test Environment

### Runtime Topology

- Application runtime: Quarkus (JVM mode, dev servers)
- Test client: k6 (running locally)
- Infrastructure (local emulation):
  - DynamoDB -> LocalStack
  - PostgreSQL -> LocalStack
  - S3 -> LocalStack
- External dependency:
  - Cognito -> real AWS (`us-west-2`)

### Key Characteristics

- Mixed local + real cloud dependency model
- No network isolation (localhost execution)
- No containerization (yet)
- JVM warm-up behavior present
- Circuit breaker + timeout tuning applied during test iteration

---

## Test Configuration

```text
scenarios: 2 total
- warmup: 50 VUs for 1 minute
- load:   50 VUs for 3 minutes (steady state)
max VUs: 100
```

- Each run includes full test data cleanup beforehand
- Same scenario executed repeatedly for consistency
- Endpoint-level custom metrics enabled

---

## Summary of Results (Steady-State Runs)

Runs 3-5 represent stable system behavior after warm-up and tuning.

### Throughput

- Sustained: ~52-55 requests/sec
- Stable across runs

### Latency (aggregate)

| Metric | Value |
| --- | --- |
| p50 | ~9-12 ms |
| p95 | ~500-700 ms |
| p99 | ~1.0-1.1 s |

---

## Endpoint-Level Performance

### Actor Fetch (`endpoint_actor_get`)

- p50: ~7-9 ms
- p95: ~70-125 ms
- p99: ~130-425 ms

Interpretation:

- Fast and stable
- Minimal variance
- Represents core service performance

### Login (`endpoint_auth_login`)

- avg: ~400-450 ms
- p95: ~600-900 ms
- p99: ~700 ms-1 s

Interpretation:

- Dominated by external identity provider (Cognito)
- Stable latency profile
- Not an internal bottleneck

### Registration (`endpoint_auth_register`)

- avg: ~1.0 s
- p95: ~1.2-1.3 s
- p99: ~1.3-2.2 s

Interpretation:

- Heavier external operation (user creation)
- Stable after warm-up
- Occasional 400 errors due to test data collisions (acceptable)

### Document Upload (`endpoint_document_upload`)

- avg: ~130-175 ms
- p95: ~250-340 ms
- p99: ~400 ms-2 s

Interpretation:

- Initially unstable due to circuit breaker configuration
- Stabilized after tuning:
  - Timeout: `1000ms -> 5000ms`
  - Circuit breaker delay + thresholds adjusted
- Now predictable with bounded tail latency

---

## Observed System Behavior

### 1. Cold vs Warm State

Earlier runs (1-2) exhibited:

- Long tail latency (up to ~10s)
- Higher variability

After sustained execution (runs 3-5):

- Tail latency reduced significantly
- System stabilized

Conclusion: Warm-up phase alone is insufficient; system requires extended load to reach steady state.

### 2. Tail Latency Drivers

Primary contributors:

- External dependency (Cognito)
- Document upload path (now mitigated)
- Initial circuit breaker sensitivity

### 3. Failure Analysis

#### Registration Failures

- Cause: duplicate user creation (Cognito)
- Frequency: extremely low (~0.01%)
- Status: acceptable, realistic

#### Document Upload Failures (Run 4)

- Cause: circuit breaker half-open state
- Resolution: configuration tuning
- Status: resolved and reproducible

---

## Key Insights

### Fast Core, Slower Edges

- Internal services respond in single-digit milliseconds
- End-to-end latency is driven by external systems

### Stability Achieved Through Tuning

- Circuit breaker configuration had a material impact
- Proper thresholds eliminated instability

### Repeatability Established

- Runs 3-5 show consistent, predictable behavior
- Variance is understood and explainable

---

## Internal Positioning Statement

The platform demonstrates stable, repeatable performance under sustained load with predictable latency characteristics. Internal services respond in single-digit milliseconds, while end-to-end latency is dominated by external dependencies such as identity providers.

---

## Limitations of This Test

- Localhost execution (no real network conditions)
- LocalStack does not fully replicate AWS performance characteristics
- No container orchestration (ECS not yet introduced)
- No load balancer layer
- JVM mode only (no native image comparison yet)

---

## Next Steps

### Move to AWS ECS

- Deploy services as ECS tasks
- Introduce ALB
- Run load tests from EC2 (same region)

### Validate Horizontal Scaling

- 1 -> 2 -> 5+ tasks
- Measure scaling efficiency
- Track latency distribution changes

### Optional Enhancements

- JVM vs native image comparison
- Per-endpoint isolated scenarios
- Extended soak testing

---

## Appendix: Selected Run Data

### Run 3 (Representative Stable Run)

```text
Throughput: 54.5 req/s
http_req_duration:
  p50: 11.11ms
  p95: 513.29ms
  p99: 1.04s
  max: 2.22s
```

### Run 4 (Circuit Breaker Event)

```text
Failures: 25 (document upload)
Cause: circuit breaker half-open state
Resolution: increased timeout + adjusted thresholds
```

### Run 5 (Post-Tuning Stability)

```text
Throughput: 53.5 req/s
http_req_duration:
  p50: 12.14ms
  p95: 672.74ms
  p99: 1.1s
  max: 4s
Failures: 0
```

---

## Conclusion

The system is now in a stable, well-understood state suitable for:

- Migration to AWS infrastructure
- Horizontal scaling validation
- Generation of publishable performance data

No critical performance or reliability concerns remain at this stage.
