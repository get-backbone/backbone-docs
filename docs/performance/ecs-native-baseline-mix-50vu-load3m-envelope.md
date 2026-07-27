---
title: "ECS Baseline Phase 2 - Performance Envelope and Operating Point"
summary: "This document is the Phase 2 output from PERFTESTPLAN.md: a single reference"
---

This document is the **Phase 2 output** from [`PERF_TEST_PLAN.md`](/docs/perf-test-plan): a single reference profile derived from five repeated k6 runs (no new load testing). It defines the **official baseline performance envelope** and the **steady-state operating point** for ECS native at 50 VUs, baseline-mix, 3 min warmup and 3 min load.

## Sources

| Artifact | Path |
| --- | --- |
| Raw k6 end-of-test summaries (5 runs) | [`../phase_1/ecs-native-baseline-mix-50vu-load3m-raw.txt`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_1/ecs-native-baseline-mix-50vu-load3m-raw.txt) |
| Per-run extract (spreadsheet-friendly) | [`ecs-native-baseline-mix-50vu-load3m-data.csv`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_2/ecs-native-baseline-mix-50vu-load3m-data.csv) |
| Narrative interpretation | [`../phase_1/ECS_NATIVE_BASELINE_MIX_50VU_LOAD3M_SUMMARY.md`](/docs/ecs-native-baseline-mix-50vu-load3m-summary) |

## Method

* **Throughput:** each run contributes one RPS sample; report mean, min, max, sample standard deviation, and coefficient of variation (CV).
* **HTTP percentiles (p95, p99):** report the **band** across runs (min and max of the five k6-reported values). Do not treat the mean of five run-level p95s as a pooled p95.
* **Medians and averages:** report min/max across runs and the mean of the five per-run values where useful.

## Throughput (`http_reqs` rate)

**req/s (RPS)** is k6’s `http_reqs` per second: the **aggregate HTTP request rate across all endpoints** invoked in the baseline-mix workflow (login, actor fetch, register, upload, and any supporting calls). It is **not** VU arrivals per second, **not** a single business transaction type, and **not** scenario iterations per second (iteration rate is reported separately below).

| Stat | Value |
| --- | --- |
| Per-run RPS | 55.969, 56.046, 56.154, 56.269, 56.278 |
| Mean | **56.14 req/s** |
| Min / max | **55.97 / 56.28 req/s** |
| Sample stdev | **0.136 req/s** |
| CV (stdev / mean) | **0.24%** |

At CV **0.24%**, throughput is effectively **deterministic steady-state** at this load: variation across runs is negligible for capacity planning.

### Iteration rate (scenario completions)

k6 reports completed scenario iterations separately from HTTP RPS. Each iteration runs the mixed workflow (including the weighted register path).

| Stat | Value |
| --- | --- |
| Per-run iterations/s | 44.504, 44.714, 44.800, 44.812, 45.006 |
| Min / max | **44.50 / 45.01 iter/s** |
| Mean | **44.77 iter/s** |

**HTTP requests per iteration** (total `http_reqs` ÷ iterations for that run): about **1.25–1.26** (mean **1.25**). That ratio is useful when comparing **business throughput** (iterations) to **raw HTTP throughput** (RPS) under different VU counts or scenario mixes.

## Latency (HTTP overall, `http_req_duration`)

| Stat | Value |
| --- | --- |
| Median (per-run med): min / max | **6.14 / 6.41 ms** |
| Mean of per-run medians | **6.28 ms** |
| Average (per-run avg): min / max | **64.13 / 71.02 ms** |
| Mean of per-run averages | **67.44 ms** |
| Observed aggregate HTTP p95 (min/max of the five per-run k6 values) | **357.7 / 405.3 ms** |
| Observed aggregate HTTP p99 (min/max of the five per-run k6 values) | **922.7 / 937.2 ms** |
| Max (per-run max): min / max | **2.10 / 2.84 s** |

**Note on “mean of per-run medians”:** it summarizes central tendency across runs when variance is small; it is **not** a pooled median over all requests. Use it as a compact headline figure, not as a substitute for a single-run or merged histogram.

**Note on per-run maxima:** rare multi-second maxima (**~2.1–2.8 s**) sit **above** the aggregate p99 band and are therefore **outliers** in the sense of the reported percentile ladder. Plausible contributors include **external dependency latency (AWS Cognito)** and **network variance**; causality is not proven from summary text alone.

## Error rate

* **0.00%** `http_req_failed` in all 5 runs (**101,882** total HTTP requests).

## Endpoint bands (custom metrics, across 5 runs)

### Actor GET

* Avg: **7.01 / 9.00 ms** (mean **8.02 ms**)
* p99: **32.5 / 70.5 ms**
* Max: **272.7 / 798.8 ms**

### Auth login (Cognito)

* Avg: **407.4 / 437.5 ms** (mean **417.5 ms**)
* Median: **327.6 / 330.6 ms**
* p95: **1.10 / 1.39 s**

### Auth register (Cognito)

* Avg: **869.1 / 883.9 ms** (mean **875.6 ms**)
* Median: **857.0 / 860.1 ms**
* p95: **992.0 / 1020 ms**

### Document upload

* Avg: **49.30 / 52.63 ms** (mean **50.81 ms**)
* Median: **44.94 / 47.14 ms**
* p95: **74.02 / 83.13 ms**
* p99: **108.08 / 137.01 ms**

### Aggregate tail behaviour

**Auth endpoints (Cognito-backed login and register) dominate aggregate `http_req_duration` p95/p99** at this mix and therefore define **system-wide tail characteristics** for the workload as executed. Internal and upload paths are comparatively flat; tail shape is driven by auth latency and its variance.

## Steady-state operating point (50 VUs, baseline-mix, ECS native)

At **50 concurrent users**, **1 ECS task per service**, **256 CPU / 512 MiB**, GraalVM **native** images, load generator **in-VPC EC2**, the system sustains about **56.1 req/s** mean HTTP throughput with **55.97–56.28 req/s** observed across five runs, and **44.77 iter/s** mean iteration rate (**44.50–45.01 iter/s** band). Aggregate HTTP latency shows **~6.3 ms** as the mean of per-run medians, **observed aggregate p95 values** in the **357.7–405.3 ms** range across runs, **observed aggregate p99** in the **922.7–937.2 ms** range, and rare **~2.1–2.8 s** maxima that sit above p99 and are treated as outliers (see latency notes above). **Zero** HTTP failures were recorded across all runs.

**Saturation:** there was **no throughput degradation**, **no broad latency inflation** (percentile bands stayed tight run-to-run), and **no error emergence** across the five runs. That indicates the system was **operating below saturation** at this load level for the tested topology.

Use this profile as the quantitative reference for Phase 3 scaling comparisons (VU steps, task count, or runtime changes). With it, statements such as “at 100 VUs, aggregate p95 rose X% versus baseline” or “throughput stopped scaling linearly beyond Y VUs” become **comparable to a defensible baseline**.
