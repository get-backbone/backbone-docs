---
title: "ECS Performance Test Design (Native + k6)"
summary: "Define controlled load progression for stress testing."
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
