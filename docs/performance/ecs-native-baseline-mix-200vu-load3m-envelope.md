---
title: "ECS Scaling Phase 4 — 200 VU Performance Envelope and Operating Point"
summary: "This document is the Phase 4 output from PERFTESTPLAN.md: a consolidated performance profile at 200 VUs"
---

This document is the **Phase 4 output** from `PERF_TEST_PLAN.md`: a consolidated performance profile at **200 VUs** derived from five k6 runs, incorporating both configuration tuning and horizontal scaling adjustments.

## Important Context

- 50 / 100 VU runs used an incorrect auth distribution (single-identity bias)
- 200 VU runs use **true multi-user concurrency**
- This is the **first production-representative dataset**

### Run provenance (how to read the numbers)

| Runs | Role |
|------|------|
| **1–2** | Exploratory: 1 ECS task per service; Run 2 includes connection-pool tuning. Use for **before/after horizontal scaling**, not as the steady-state envelope. |
| **3–5** | **Steady-state envelope:** 200 VUs, **2 ECS tasks** per service, same reduced pool sizing (`max-size=14`, `min-size=5`), same k6 scenario. Raw output: [`ecs-native-baseline-mix-200vu-load3m-raw.txt`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_4/ecs-native-baseline-mix-200vu-load3m-raw.txt). |

**Stabilization nuance:** Run 3 is the first run after the topology + pool change; Runs 4–5 are fully settled. Metrics align across all three; Run 3 confirms immediate consistency after the change, Runs 4–5 are the cleanest steady-state samples for a strict reading.

---

## Throughput (`http_reqs` rate)

**All five runs** (includes single-task Runs 1–2):

- Per-run RPS: 190.49, 189.32, 222.35, 221.79, 221.97
- Mean: **209.18 req/s**
- Min / max: **189.32 / 222.35 req/s**
- Std dev: ~15.7 (population) / ~17.6 (sample)
- CV: ~7.5%

**Steady-state only (Runs 3–5, 2 tasks):**

- Per-run RPS: 222.35, 221.79, 221.97
- Mean: **222.04 req/s**
- Min / max: **221.79 / 222.35 req/s**
- Spread: **under 0.3%** between min and max

**Interpretation:**
- ~190 RPS → single-task ceiling (Runs 1–2)
- ~222 RPS → post horizontal scaling (Runs 3–5)
- Scaling is step-based, not linear

---

## Latency (HTTP overall)

- Median: **6.5 / 180.7 ms**
- Mean median: ~67 ms
- Avg: **75.28 / 232.05 ms**
- p95: **364 / 793 ms**
- p99: **852 ms / 1.17 s**

**Interpretation:**
- Clear split between saturated vs scaled system
- Tail latency stable but auth-influenced

---

## Error Rate

- ~0.0024% overall (9 failures / 379,269 requests)

Failure types:
- 502 → transient infra/network edge
- 401 → test data concurrency
- 400 → registration race

**Conclusion:** No systemic instability at the envelope level; only isolated transient failures.

---

## Endpoint Summary

### Actor GET
- Avg: 14–159 ms
- Median: ~6 ms (stable when scaled)
- p99: 200–697 ms

→ Strong improvement with scaling

---

### Auth Login
- Avg: 551 ms – 1.01 s
- p95: up to ~6 s
- p99: up to ~7.3 s

→ Dominant latency concentration in the current profile (source not fully isolated).

---

### Auth Register
- Avg: 803–1010 ms
- Stable distribution

---

### Document Upload
- Avg: 60–246 ms
- Recovers strongly after scaling

---

## Steady-State Operating Point (200 VUs)

Configuration (Runs **3–5**):

- 200 VUs
- 2 ECS tasks per service
- Tuned DB + HTTP pools (including post-scale pool reduction)
- In-VPC load generator

Performance:
- Throughput: **~222 RPS** (Runs 3–5 cluster ~221.8–222.4 req/s)
- Median: **~6–7 ms**
- Avg: **~75–80 ms**
- p95: **~360–420 ms**
- p99: **~850–900 ms**
- Errors: ~0%

---

## Scaling Interpretation

### Horizontal Scaling
- 190 → 222 RPS (+17%)
- Latency reduced ~3×

### Bottleneck
- Internal services scale well
- Auth layer dominates latency in this mixed-flow test, but attribution is not fully isolated to Cognito

---

## Conclusion

- System scales cleanly and predictably
- Internal services are not the limiting factor
- Auth/login path currently defines the observed latency ceiling; confirm source with auth-isolated tests

---

## Recommendation

Next steps:
- Test 300 / 400 VUs
- Run auth-isolated tests (token reuse)

This dataset is the first valid production-representative performance baseline.
