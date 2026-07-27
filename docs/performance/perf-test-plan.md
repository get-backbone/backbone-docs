---
title: "ECS Performance Testing Plan (Native + k6)"
summary: "This document defines the structured performance testing workflow for validating Backbone"
---

## Overview

This document defines the structured performance testing workflow for validating Backbone running on AWS ECS with GraalVM native images, using k6 load generation from an EC2 instance within the same VPC.

The goal is to establish:

* A stable baseline performance profile
* A repeatable scaling curve
* A clear saturation point per ECS task
* Evidence of horizontal scalability

---

## Phase 1 — Baseline Confirmation (Stability Validation)

### Objective

Confirm that the system behaves consistently at a fixed load level.

### Test Configuration

* 50 virtual users (VUs)
* 3 min warm-up
* 3 min load phase
* ECS: 1 task per service
* Runtime: GraalVM native images
* Load generator: EC2 in same VPC

### Execution

* Run the same test **5 times**
* Do not change infrastructure between runs
* Do not restart services between runs

### Success Criteria

* RPS variation within tight band (±5–10%)
* p95 latency stable across runs
* p99 latency stable across runs
* 0% unexplained failures

### Output

* Baseline throughput (RPS range)
* Baseline latency profile (p50/p95/p99)
* Error consistency profile

---

## Phase 2 — Baseline Consolidation

### Objective

Convert raw runs into a single reference performance profile.

### Process

* Aggregate 5 baseline runs
* Compute:

  * Mean RPS
  * RPS variance
  * p95/p99 range bands
  * Error rate consistency

### Output

* **Official baseline performance envelope** and **steady-state operating point** (ECS, 50 VUs, baseline-mix, 3 min warmup + 3 min load): [`phase_2/ECS_NATIVE_BASELINE_MIX_50VU_LOAD3M_ENVELOPE.md`](/docs/ecs-native-baseline-mix-50vu-load3m-envelope) (raw k6 text: [`phase_1/ecs-native-baseline-mix-50vu-load3m-raw.txt`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_1/ecs-native-baseline-mix-50vu-load3m-raw.txt); spreadsheet extract: [`phase_2/ecs-native-baseline-mix-50vu-load3m-data.csv`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_2/ecs-native-baseline-mix-50vu-load3m-data.csv))
* **Narrative summary** (interpretation, not Phase 2 math): [`phase_1/ECS_NATIVE_BASELINE_MIX_50VU_LOAD3M_SUMMARY.md`](/docs/ecs-native-baseline-mix-50vu-load3m-summary)

---

## Phase 3 — Scaling Test Design

### Objective

Define controlled load progression for stress testing.

### VU Steps

* 50 VUs (baseline)
* 100 VUs
* 200 VUs
* 300+ VUs (as needed)

### Rules

* Keep scenario identical across all runs
* Maintain same warm-up structure
* Run each level multiple times (2–3 minimum)

### Output

* Scaling test plan ready for execution

---

## Phase 4 — Horizontal Scaling Curve Test

### Objective

Measure system behaviour under increasing concurrency.

### Metrics

For each VU level:

* RPS scaling factor
* p50 / p95 / p99 latency
* error rate
* CPU/memory utilisation (ECS)

### Success Criteria

* Near-linear scaling initially
* Predictable degradation curve at saturation

### Output

* Scaling curve (VUs vs RPS)
* Latency degradation profile

---

## Phase 5 — Saturation Identification

### Objective

Determine the system’s per-instance limits.

### Indicators

* RPS plateaus despite increased VUs
* p95 increases non-linearly
* p99 spikes significantly
* error rates begin to appear

### Bottleneck Classification

* ECS CPU saturation
* Database connection limits
* External dependency limits (e.g. Cognito)
* Network / ALB constraints

### Output

* Maximum stable RPS per ECS task
* Identified primary bottleneck

---

## Phase 6 — Horizontal Scaling Validation

### Objective

Verify linear scaling across multiple ECS tasks.

### Setup

* Increase task count (1 → 2 → 5 → N)
* Run mid-range load (e.g. 100–200 VUs)

### Metrics

* Throughput scaling linearity
* Latency stability under distribution
* Error rate consistency

### Output

* Scaling efficiency model
* Confirmation of stateless architecture

---

## Phase 7 — Production-Style Load Testing

### Objective

Simulate real-world traffic variability.

### Patterns

* Ramp-up / ramp-down loads
* Burst traffic spikes
* Extended soak tests (10–30+ minutes)

### Output

* Stability under realistic conditions
* Long-run degradation behaviour

---

## Phase 8 — Final Performance Model

### Objective

Produce a formal performance characterization.

### Deliverables

* Per-task capacity (RPS ceiling)
* Scaling efficiency curve
* Safe operating envelope
* Known bottlenecks

### Outcome

A complete, defensible performance profile suitable for:

* architecture decisions
* customer communication
* future optimisation work

---

## Summary

This plan progresses from:

1. Stability validation
2. Baseline consolidation
3. Controlled scaling
4. Saturation discovery
5. Horizontal scaling validation
6. Production behaviour modelling

Result: a full understanding of system performance characteristics under load.
