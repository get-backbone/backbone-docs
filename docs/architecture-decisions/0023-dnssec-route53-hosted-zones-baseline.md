---
title: "0023. DNSSEC and Route 53 hosted zones (baseline posture)"
summary: "signed by keys the parent chain trusts. It addresses DNS data integrity at the resolution layer, not transport"
---

**Status:** Accepted  
**Date:** 2026-04-06  
**Context:** DNSSEC signing trade-offs for Route 53 child zones under signed parents; baseline posture vs portability and ACM validation.

## **Context**

**DNSSEC** (DNS Security Extensions) lets validating resolvers cryptographically verify that DNS answers for a zone were signed by keys the parent chain trusts. It addresses **DNS data integrity** at the resolution layer, not transport encryption to HTTPS endpoints.

The Backbone **Domain stack** provisions a **Route 53 public hosted zone** per environment (e.g. `int.example.com`). The **parent** zone (e.g. `example.com`) often lives at a **registrar** and may have **DNSSEC enabled** with **DS** records at the TLD. The **child** zone can be **unsigned** (no Route 53 DNSSEC signing) while the parent **is** signed. That combination can produce **bogus** DNSSEC validation for some validators when the **signed parent** does not answer **DS** queries or **denial-of-existence** proofs for the delegation in a **standards-compliant** way (see RFC 4034). **AWS Certificate Manager** domain validation can fail in that situation even when a **plain** `dig` CNAME for the validation record appears correct.

**Route 53** can **enable DNSSEC signing** for a hosted zone using a **Key Signing Key (KSK)** backed by **KMS**, after which **DS** records must be published at the **parent** (registrar or DNS operator). That is **operationally** heavier than an unsigned zone: **KMS** policies, **KSK** lifecycle, **DS** updates at the parent, and alignment with registrar capabilities.

**Product positioning:** the platform is intended to stay **vanilla** and **portable**. Customers may **own** their registrar (GoDaddy, Cloudflare, Route 53 Registrar, and others) and **DNS** layout. **DNSSEC** policy is often **organizational** and **registrar-specific**; baking it into the baseline CDK would constrain deployments that do not need it.

**Security posture elsewhere:** the platform already relies on **TLS** (ACM on load balancers), **AWS WAF** on the internet-facing ALB, **security groups**, **least-privilege IAM**, **identity** (for example Cognito and OIDC patterns per existing ADRs), and a **documented target** of **CloudFront** in front of the public edge (ADR-0022). Those controls address **transport**, **application**, **network**, and **identity** abuse. They **do not** replace DNSSEC, which targets a **different** threat class (tampering or spoofing of **DNS answers** as seen by **validating** resolvers).

---

## **Decision**

1. **Baseline CDK does not enable Route 53 DNSSEC signing** for environment hosted zones. Zones remain **DNSSEC-insecure** (unsigned) unless a **customer** or **operator** explicitly enables signing later.

2. **Registrar** and **parent** DNSSEC (for example DNSSEC on the apex at GoDaddy) are **out of scope** for the reference codebase. Operators choose **DNSSEC on** or **off** at the parent based on **policy** and **ACM** validation behavior; **disabling** DNSSEC at the parent is a **valid** operational path when **signed parent + unsigned delegated child** causes **bogus** validation and **ACM** cannot issue certificates.

3. **Extension path:** nothing in the architecture **blocks** a future choice to **enable Route 53 DNSSEC** for a hosted zone (KSK, KMS policy, **`AWS::Route53::DNSSEC`**) and to publish **DS** at the parent, or to move **apex** DNS fully into **Route 53** for a single **AWS**-managed story. **Customers** with strict **DNSSEC** requirements can adopt that pattern **outside** or **on top of** the baseline.

4. **Documentation:** treat **DNSSEC** as an **optional** hardening layer for **DNS integrity**, not a prerequisite for **TLS**, **WAF**, **IAM**, or **application** security.

---

## **Consequences**

- **Positive:** Simpler **CDK** and **operations**; fewer **KMS** and **registrar** coupling points; **ACM** DNS validation **often** succeeds when **parent** DNSSEC mis-delegation is **not** present; aligns with **customer-owned** DNS and registrar.

- **Negative (explicit):** Baseline **does not** provide **DNSSEC-signed** answers for **public** DNS. Clients that **mandate** DNSSEC for **all** zones must **plan** **Route 53** signing **plus** **DS** at the parent, or **DNS** architecture that satisfies their **policy**.

- **Security posture:** **No change** to the claim that the platform uses **HTTPS**, **WAF**, **IAM**, **network isolation**, and **identity** controls as described in existing ADRs. **DNSSEC** is **not** a substitute for those controls; omitting it **does not** invalidate them. It **does** mean **DNS integrity** for **validating** resolvers is **not** a **baseline** guarantee.

---

## **Related**

- **ADR-0022:** Internet-facing ALB edge and origin protection (TLS, WAF, CloudFront target direction).
- **ADR-0020:** Single VPC per environment (networking and security groups).
- **ADR-0011:** Stateless JWT authentication (application identity).
- **AWS (DNSSEC):** [DNSSEC signing in Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec.html)
- **AWS (ACM):** [DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html)

---
