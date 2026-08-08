---
title: "0024. Internal (private) ALB TLS for east-west traffic (optional)"
summary: "The runtime uses an internal-facing Application Load Balancer in private subnets for"
---

**Status:** Accepted  
**Date:** 2026-04-07  
**Context:** Optional TLS on internal ALBs for in-VPC east-west traffic, versus HTTP behind security groups and private subnets.

## **Context**

The runtime uses an **internal-facing Application Load Balancer** in **private subnets** for **service-to-service** and **browser-to-API** paths that are routed by hostname and path rules. Traffic stays **inside the VPC** between **Fargate** tasks and the internal ALB.

**TLS on the internet-facing ALB** is **required** for browser-facing HTTPS and is **implemented** (ACM certificate, HTTPS listener on port 443; no public HTTP listener). Viewer HTTP→HTTPS redirect is enforced at **CloudFront** (`REDIRECT_TO_HTTPS`), not via an ALB :80 redirect. That work establishes a **repeatable pattern**: ACM in the same region as the ALB, listeners, and task **client** configuration using the **correct scheme** (for example **CORS** `Origin` matching **https** for the public hostname).

**TLS on the internal ALB** adds **encryption in transit** for **east-west** hops and raises the bar against **passive** observation or trivial **in-VPC** manipulation of cleartext. It does **not** replace **service authentication**, **stateless** tokens, or **least-privilege IAM**; those controls prove **who** is calling. **Defense in depth** combines **identity** with **confidentiality** on the wire.

**Operational reality:** not every deployment has **strict PII**, **regulated** workloads, or organizational mandates for **encrypt everything** inside the VPC. Internal HTTP behind **security groups** and **private** subnets is a **common** AWS pattern. Rolling out internal **HTTPS** on the **listener** is **incremental** once public TLS is understood: ACM-issued certificate for a resolvable internal hostname, or a private CA-issued certificate where required, **443** listener on the internal ALB, **security group** ingress on **443**, and **https://** base URLs in **REST clients** and health checks where applicable. That still leaves the **ALB → target** hop as a **separate** decision (see below).

---

## **ALB TLS termination and the hop to targets**

**Application Load Balancers** terminate **TLS** on the **listener** when you use an **HTTPS** listener. Traffic from the load balancer to **registered targets** uses whatever **protocol** the **target group** is configured for.

**Baseline Backbone CDK**: **all** application target groups use **`protocol: HTTP`** to **ECS tasks** on the **container port**. The **public** ALB uses an **HTTPS** listener toward clients and **forwards** to those **HTTP** target groups. So today:

| Segment                        | Encrypted in baseline?                     |
|--------------------------------|--------------------------------------------|
| **Client → internet-facing ALB**        | **Yes** (HTTPS listener, ACM on the ALB).  |
| **Internet-facing ALB → ECS task**      | **No** (HTTP target group within the VPC). |
| **Task → internal ALB → task** | **No** (HTTP throughout).                  |

**Implication:** Turning on an **HTTPS** listener on the **internal** ALB encrypts **caller → internal ALB** only. It does **not**, by itself, encrypt **internal ALB → Fargate**. **Full** encryption on that last hop requires **HTTPS target groups** (and tasks serving **TLS**, with certificate trust and health checks aligned), or a **service mesh** / **mTLS** pattern (for example **AWS App Mesh**, **Istio**), or other designs **outside** the baseline ADR.

This ADR avoids implying that **listener-only HTTPS** equals **end-to-end** encryption to the task. Reviewers and customer teams should treat **infrastructure observation** between ALB and task as **out of scope** for listener TLS alone.

---

## **Decision**

1. **Default baseline keeps the internal ALB on HTTP** (`internalHttps: false` in `platform-config.yml`) for listeners between tasks and the internal ALB, using the raw ELB DNS name over `http://`.

2. **HTTPS on the internal ALB is a first-class platform opt-in** via `baseline.internalHttps` (or a per-env override). When `true`, CDK provisions a public ACM certificate for `api.{env}.{root}` (DNS validation in the public hosted zone), a **private** hosted zone apex alias to the Internal ALB (VPC-only resolution), an HTTPS:443 listener, security-group ingress on 443, and `BACKBONE_INTERNAL_ALB_URL=https://api.{env}.{root}` for REST clients. Target groups remain HTTP to Fargate (listener-only TLS).

3. **No requirement** that every Backbone deployment enable internal TLS. Operators flip the flag when compliance or threat model requires east-west encryption on the ALB hop.

---

## **Guidance: when internal HTTP is acceptable vs when to prefer internal HTTPS**

The baseline choice is **deliberately** HTTP on the internal ALB. The following thresholds make **when to stay** on that posture vs **when to add** internal TLS explicit for **security reviewers** and **customer teams**. They do **not** change the **Decision**: CDK still ships HTTP internally until a team opts in.

**Internal HTTP on the private ALB is a reasonable posture when** (all are generally true for the deployment):

- Payloads are **non-sensitive** or sensitivity is **low** (no material **PII**, **secrets**, or **long-lived tokens** on these hops).
- Workloads are **single-tenant** or **tightly scoped** (limited blast radius if a task is compromised).
- The **threat model** does **not** assume an **in-VPC** adversary who can passively capture or alter east-west traffic (for example no requirement to defend against sniffing inside the VPC boundary).
- **Compliance** obligations do **not** mandate **encryption in transit** for east-west segments.

**Internal HTTPS (or mTLS) on the private ALB is strongly recommended when** **any** of the following apply:

- Requests or responses carry **PII**, **credentials**, **bearer tokens** (for example **JWTs**, access tokens), or other **confidential** payload data across the internal ALB.
- **Multi-tenant** workloads or **shared AWS accounts** widen **blast radius** or co-mingle trust boundaries.
- **Compliance** or customer policy expects **encryption in transit** end-to-end (for example **SOC 2**, **HIPAA**, or equivalent regimes; exact scope is **customer** legal interpretation).
- The organization assumes **workload compromise** is plausible and wants **defense in depth** against **passive** collection or trivial tampering **on the wire** inside the VPC (internal TLS raises the bar; it does **not** remove the need for **authn** and **authz**).

**Customer-owned action:** set `internalHttps: true` in `platform-config.yml` (baseline or per-env) and redeploy Domain + Security + Runtime stacks together. Flipping the flag changes listener port, DNS, certificate, and all task `BACKBONE_INTERNAL_ALB_URL` values in one cut — treat it as a planned change, not a hot toggle. If **policy** requires **encryption all the way to the task**, teams must also plan **HTTPS target groups** (and **TLS** on the application) or **mesh** / **mTLS**, as described in **ALB TLS termination and the hop to targets** above.

---

## **Consequences**

- **Positive:** Simpler **baseline** operations (single scheme for internal URLs, no extra cert lifecycle for the internal hostname unless needed); **flexibility** for customers with different **compliance** bars; **clear** separation between **internet** risk (must encrypt) and **VPC** risk (policy-driven).

- **Explicit gap:** **Baseline** east-west traffic to the internal ALB is **not** TLS-protected. See **Guidance** above for **when** that is acceptable vs **when** customer teams should treat internal **HTTPS** (or **mTLS**) as **expected**, not optional.

- **Listener TLS alone:** Even the **public** ALB does **not** use **HTTPS** to targets in baseline; **ALB → task** is **HTTP**. Internal listener HTTPS would mirror that split. Do **not** equate **ALB HTTPS** with **end-to-end** encryption to **ECS** without **target group** and **application** changes (or mesh). See **ALB TLS termination and the hop to targets**.

- **Security posture:** **Zero-trust** service authentication and **stateless** conversations remain **authoritative** for **caller identity**; internal TLS is an **additional** layer for **payload** protection on the **network path**, not a substitute for **authn** or **authz**.

---

## **Related**

- **ADR-0020:** Single VPC per environment (internal ALB placement, security groups).
- **ADR-0022:** Internet-facing ALB edge and origin protection (internet-facing ALB TLS, WAF, CloudFront direction).
- **ADR-0023:** DNSSEC and Route 53 hosted zones (baseline posture; registrar and certificate context).
- **AWS (ALB HTTPS):** [ALB HTTPS listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)
- **AWS (target groups):** [ALB target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) (protocol and TLS to targets).

---
