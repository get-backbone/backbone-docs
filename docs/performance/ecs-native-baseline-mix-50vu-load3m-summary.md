---
title: "ECS Performance Testing Results (5× Run Analysis)"
summary: "This document captures and analyses 5 repeated load test runs executed against the AWS ECS deployment running GraalVM native images."
---

## Overview

This document captures and analyses 5 repeated load test runs executed against the AWS ECS deployment running GraalVM native images.

### Test Conditions (constant across all runs)

* 1 ECS task per service
* CPU: 256 (0.25 vCPU)
* Memory: 512 MiB
* Runtime: GraalVM native images (Quarkus)
* Load generator: EC2 instance in same VPC
* External dependency: AWS Cognito (us-west-2)
* Data stores: LocalStack (DynamoDB, Postgres, S3)
* Test profile:

  * 50 VUs warmup (3 min)
  * 50 VUs load (3 min)

### Key change vs earlier local tests

* Transition from localhost + LocalStack only → ECS + EC2 + mixed AWS dependency model
* Native image execution enabled
* More realistic network topology and latency profile

---

## Executive Summary

Across 5 identical runs, the system demonstrates:

* **Highly stable throughput (~55–56 RPS)**
* **Very consistent latency distribution across runs**
* **Zero systemic failure rate (100% success in 5/5 runs)**
* **Predictable external dependency latency (Cognito dominates auth variability)**
* **Document upload performance is now stable (~45–55ms median)**

### Core takeaway

> The system exhibits a stable, repeatable performance envelope under steady-state load at 50 concurrent users, with no signs of internal saturation at this level.

---

## Aggregate Performance Summary (5 runs)

### Throughput

* **RPS range:** 55.96 – 56.27 req/s
* **Variance:** extremely low (<1%)

### Latency (HTTP overall)

* **Median:** ~6–12 ms
* **Average:** ~64–71 ms
* **p95:** ~350–405 ms
* **p99:** ~920 ms – 940 ms
* **Max spikes:** up to ~2–2.8s (external dependency or rare pathing)

### Error Rate

* 0% failures in 5/5 runs

Quantitative consolidation (mean RPS, percentile bands, operating point) lives in [`../phase_2/ECS_NATIVE_BASELINE_MIX_50VU_LOAD3M_ENVELOPE.md`](/docs/ecs-native-baseline-mix-50vu-load3m-envelope).

---

## Endpoint-Level Behaviour

### Actor GET

* Avg: ~7–9 ms
* p99: ~30–70 ms
* Max: occasional spikes up to ~700–800 ms

**Interpretation:**

* Extremely healthy
* Likely fully in-memory / cached path

---

### Auth Login (Cognito-backed)

* Avg: ~407–437 ms
* Median: ~327–330 ms
* p95: ~1.1–1.4 s

**Interpretation:**

* Dominant latency source
* Highly consistent across runs
* External dependency bound (not ECS-bound)

---

### Auth Register (Cognito-backed)

* Avg: ~870–883 ms
* Median: ~850–860 ms
* p95: ~990 ms – 1.02 s

**Interpretation:**

* Stable but expensive operation
* Likely constrained by Cognito user pool operations

---

### Document Upload

* Avg: ~49–53 ms
* Median: ~45–47 ms
* p95: ~74–83 ms
* p99: ~108–137 ms

**Interpretation:**

* Very stable after tuning
* No longer a bottleneck
* Well-behaved under concurrency

---

## Run-to-Run Stability Analysis

### Throughput stability

All runs:

* ~55–56 RPS
* Negligible variance

✔ Indicates stable CPU scheduling and no ECS contention at this load

---

### Latency stability

* Aggregate HTTP p95 across runs falls roughly in **~350–405 ms** (mix-sensitive; not endpoint-level p95)
* p99 consistently ~900 ms–1.1 s

✔ Stable tail behaviour across runs

---

### Error behaviour

* No systemic errors observed

✔ No infrastructure instability indicated

---

## Comparison to Local Environment (previous tests)

### Throughput

* Local: ~51–54 RPS
* ECS native: ~55–56 RPS

→ Slight improvement, primarily due to:

* more consistent scheduling
* reduced local contention noise

---

### Latency

* Local avg HTTP: ~95–120 ms
* ECS avg HTTP: ~64–71 ms

→ ECS native performs better overall, despite network overhead

**Key insight:** Native images + ECS steady-state execution offset network overhead from localhost testing.

---

### Tail latency

* Local p99: ~900 ms – 1.1 s
* ECS p99: ~920 ms – 940 ms

→ Essentially unchanged

**Interpretation:** Tail latency is dominated by external dependency (Cognito), not compute layer.

---

## Key Technical Insights

### 1. System is not CPU bound at 50 VUs

* Very low internal latencies
* No throughput degradation across runs

---

### 2. External dependency dominates tail latency

* Auth flows define p95/p99 behaviour
* ECS does not contribute materially to tail spikes

---

### 3. Document pipeline is stable post-tuning

* Circuit breaker tuning successful
* No sustained error propagation

---

### 4. System exhibits strong repeatability

* Across 5 runs, variance is minimal
* Indicates a stable production-like baseline

---

## Final Conclusion

At 50 concurrent users:

> The system demonstrates a stable, repeatable performance envelope with ~55 RPS throughput, sub-10ms median internal latency, and predictable tail latency primarily driven by AWS Cognito.

### What can be confidently stated

* ECS results are actually cleaner than local environment results
* ECS + GraalVM native deployment is stable under steady load
* No evidence of internal bottlenecks at current configuration
* Horizontal scaling tests are now valid to proceed
* Performance characteristics are reproducible across runs

---

## Recommended Next Step

Proceed to scaling analysis:

* 100 VUs
* 200 VUs
* 300+ VUs

Goal:

> Identify saturation point and confirm horizontal scaling behaviour
