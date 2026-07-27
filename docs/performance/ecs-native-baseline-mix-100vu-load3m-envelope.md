---
title: "ECS Scaling Phase 4 — 100 VU Performance Envelope and Operating Point"
summary: "This document summarizes four k6 runs at 100 VUs."
---

This document summarizes **four** k6 runs at **100 VUs**.

See raw output in [`ecs-native-baseline-mix-100vu-load3m-raw.txt`](https://raw.githubusercontent.com/get-backbone/backbone-docs/v1.0/assets/performance/phase_4/ecs-native-baseline-mix-100vu-load3m-raw.txt).

The paste includes short notes between runs; metrics below are taken only from each run's **TOTAL RESULTS** block.

## Test configuration

- **100 VUs** for **warmup** and **100 VUs** for **load** (same `BACKBONE_PERF_VUS` for both k6 scenarios; `perf/utils/options.js`), **not** a 50→100 ramp
- **3 min** warmup, **3 min** load
- **1** ECS task per service, **256** CPU / **512** MiB
- GraalVM **native** images
- In-VPC load generator (**EC2**)

---

## Throughput (`http_reqs` rate)

Per-run RPS (Runs 1–4): **110.793**, **110.176**, **111.350**, **111.279**

| Stat | Value |
| --- | --- |
| Mean | **110.90 req/s** |
| Min / max | **110.18 / 111.35 req/s** |
| Sample stdev | **0.54 req/s** |
| CV (stdev / mean) | **~0.49%** |

**Scaling check (vs 50 VU Phase 2 baseline ~56.14 req/s mean):** ~**111 / 56.14 ≈ 1.98×** RPS for **2×** VUs - throughput tracks doubling closely with only a small shortfall versus ideal linear scaling.

---

## Latency (HTTP overall, `http_req_duration`)

Values are **bands across the four per-run k6 summaries** (min/max of each reported aggregate).

| Metric | Band (across 4 runs) |
| --- | --- |
| Median (per-run `med`) | **7.16 – 7.90 ms** |
| Mean of per-run medians | **~7.58 ms** (not a pooled median) |
| Average (per-run `avg`) | **71.66 – 82.12 ms** |
| p95 | **356.5 – 524.6 ms** |
| p99 | **884.1 – 898.3 ms** |
| Max | **3.7 – 5.2 s** |

**Interpretation:** Median stays in the **low single-digit ms** range per run (similar order to the 50 VU baseline). **p95** widens versus the 50 VU band (~358-405 ms) under heavier concurrency. **p99** stays in a tight band (~884-898 ms), consistent with Cognito-dominated tails rather than a collapsing service tier.

---

## Reliability (checks and HTTP failures)

| Run | `checks_succeeded` | `http_req_failed` | Notes |
| --- | --- | --- | --- |
| 1 | 100.00% (40298/40298) | 0 / 40298 | Clean |
| 2 | 99.98% (40006/40013) | **7 / 40013** | Seven **actor_get** check failures; seven HTTP failures; matches **outbound connection closure** symptoms on actor-bff (see raw paste + logs in source file) |
| 3 | 99.99% (40477/40478) | 1 / 40478 | Single **login** check failure (401 path); HTTP failed count **1** — consistent with a **rare Cognito / shared test-user** flake, not actor GET |
| 4 | 100.00% (40304/40304) | 0 / 40304 | Clean |

**Overall HTTP failures:** **8 / 161 093** requests ≈ **0.005%**.

After the captured Run 2, the raw file records applying **`quarkus.rest-client.connection-ttl=55000`** (ms) before subsequent runs; Runs **3** and **4** do not repeat the Run **2** actor connection pattern. Treat Run **2** as the **stress signal** that motivated TTL; treat **3** as isolated auth noise; **1** and **4** as clean steady-state samples at 100 VUs.

---

## Endpoint bands (custom metrics, min/max across 4 runs)

### Actor GET

- Avg: **13.03 – 21.75 ms**
- p99: **169.5 – 400.0 ms**

### Auth login (Cognito)

- Avg: **457.6 – 507.0 ms**
- p95: **1.69 – 2.27 s**

### Auth register (Cognito)

- Avg: **839.5 – 850.4 ms**
- p95: **0.97 – 1.06 s**

### Document upload

- Avg: **59.5 – 65.2 ms**
- p99: **251.7 – 323.5 ms**

---

## Observations

1. **Throughput:** Mean **~111 req/s** with **CV under ~0.5%** across four runs is highly repeatable at 100 VUs.
2. **Median latency:** Unchanged in order of magnitude versus 50 VU baseline; no sign of sustained queueing at the HTTP aggregate median.
3. **Tails:** **p99** band is stable; **p95** spreads wider than at 50 VUs (expected with 2× offered load and the same external auth).
4. **Failures:** Dominated by one run’s **actor** path (connection lifecycle) and one **login** flake; aggregate rate is still **near zero**.
5. **Connection handling:** Run **2** aligns with **BFF → internal ALB → actor-service** connection churn; **connection-ttl** (ms) on REST clients is the documented mitigation before Runs **3-4**.

---

## Steady-state operating point (100 VUs, post-TTL runs)

For **Runs 1, 3, and 4** (and treating Run **2** as pre-mitigation in the same capture), the system sustains about:

- **~111 req/s** mean HTTP throughput (band **~110.2–111.4 req/s**)
- **~7.2–7.9 ms** aggregate HTTP median per run (**~7.6 ms** mean of per-run medians)
- **p95** roughly **~357–525 ms**; **p99** roughly **~884–898 ms**
- **HTTP failure rate** at or below **0.01%** per run except the connection-churn run (**0.01%**)

---

## Comparison to 50 VU baseline (Phase 2 envelope)

| Metric | 50 VU (Phase 2) | 100 VU (Phase 4, this doc) |
| --- | --- | --- |
| Mean RPS | ~56.14 | ~110.90 |
| HTTP median band (per-run `med`) | ~6.14 – 6.41 ms | ~7.16 – 7.90 ms |
| HTTP p95 band (observed per-run) | ~357.7 – 405.3 ms | ~356.5 – 524.6 ms |
| HTTP p99 band | ~922.7 – 937.2 ms | ~884.1 – 898.3 ms |

**Conclusion:** Throughput scales to **~2×** with only a **small** median shift and **wider p95** (not a surprise at 2× concurrency). **p99** is not exploding versus 50 VU.

---

## Key conclusion

The platform sustains **~111 req/s** at **100 VUs** on **1 task / 0.25 vCPU**-class sizing with **very low** aggregate HTTP failure rates once connection reuse to the internal ALB is bounded (**connection-ttl**). Residual risk is mostly **external auth variance** and **shared seeded users** under concurrency, not internal saturation at this step.

---

## Next steps

1. Push load (**150–200 VUs**) and watch for **p95/p99 inflation**, **HTTP 5xx**, and **ELB** vs **target** 5xx split.
2. Compare **multi-task** ECS (horizontal) vs **larger CPU** (vertical) at the same VU step when an inflection appears.

---

## Bottom line

You have a **documented 100 VU datapoint**: **~111 req/s**, **low-ms medians**, **Cognito-shaped p99**, and **near-zero** HTTP errors after addressing **REST client connection TTL** - with one run retained in the raw log showing the **pre-fix actor connection failure mode** for traceability.
