---
title: "0020. Single VPC per environment"
summary: "For enterprise microservices (target market), we need a clear stance on network"
---

**Status:** Accepted  
**Date:** 2026-02-22  
**Context:** Whether web frontends and backends use separate VPCs or one VPC per environment with ALB exposure patterns.

## **Context**

For enterprise microservices (target market), we need a clear stance on network segmentation: one VPC for the whole environment vs. a “frontend” VPC (internet-facing) and a “backend” VPC (restricted), connected via peering or PrivateLink.

---

## **Decision**

We use a **single VPC per environment**. Exposure is controlled by **ALB security groups**: only designated frontends (e.g. `ui/*` or services with `allowInternetTraffic: true`) have ALBs that accept traffic from the internet (0.0.0.0/0); all other ALBs are internal (traffic only from ECS services in the same VPC). No separate “frontend” vs “backend” VPC.

---

### Subnet placement (same VPC)

- **Public subnets:** Only the ALB(s) that are internet-facing (e.g. the one fronting `web-actor` with `allowInternetTraffic: true`). No ECS tasks in public subnets.
- **Private subnets:** All other ALBs (internal, VPC-only traffic), plus all ECS tasks (web frontends and backend services). Internal ALBs and tasks are not assigned public IPs; outbound internet uses the NAT gateway. Traffic path: Internet → internet-facing ALB (public) → tasks (private); internal traffic: ECS → internal ALB (private) → tasks (private).

---

## **Consequences**

- **Positive:** Simpler operations, lower cost (no peering, single NAT), easier service discovery and debugging. Tight ALB SGs provide a clear internet boundary; suitable for many enterprise use cases.
- **Tradeoff:** If a frontend is compromised, the attacker is in the same VPC as backends; lateral movement is possible if other controls (SGs, IAM) are misconfigured.
- **Client flexibility:** Any client may fork the codebase and introduce a separate frontend VPC (or other segmentation) if required by compliance or risk. The codebase is structured (network, security, runtime stacks; export/import decoupling) so that splitting or reorganising VPCs is a relatively straightforward change.

---
