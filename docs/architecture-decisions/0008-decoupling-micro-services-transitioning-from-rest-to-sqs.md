---
title: "0008. Decoupling micro-services — transitioning from REST to SQS"
summary: "The system is composed of multiple Quarkus microservices (authentication, resume parsing, job spec"
---

**Status:** Accepted  
**Date:** 2025-11-18  
**Context:** Multiple Quarkus microservices in an evolving prototype; synchronous REST now with message-friendly boundaries for a later SQS move.

## **Context**

The system is composed of multiple Quarkus microservices (authentication, resume parsing, job spec parsing, scoring, etc.). The long-term architecture may benefit from asynchronous decoupling (e.g., via AWS SQS) to support scaling, retries, and distributed workflows.

However, the platform is currently in a prototype/demo phase. Although the backend code is production quality, domain boundaries and service interactions are still evolving. Introducing asynchronous messaging now would add complexity without delivering clear value.

## **Decision**

We will implement **synchronous REST calls between microservices** for the initial prototype. We will **intentionally keep service boundaries and internal APIs message-oriented**, so the system can switch to SQS (or another queue) with minimal refactoring later.

## **Rationale**

* **Accelerated delivery:** REST is faster to implement and easier to develop/debug locally.
* **Avoid premature complexity:** SQS introduces DLQs, retries, idempotency concerns, IAM config, and monitoring overhead.
* **Domain is still forming:** Boundaries and workflows are not yet stable enough to justify asynchronous orchestration.
* **Low switching cost:** By isolating business logic from transport concerns, later replacement of REST with SQS producers/consumers is a mechanical change, not a redesign.
* **Current workloads** are small, real-time, and do not require asynchronous processing.

## **Consequences**

### **Positive**

* Faster iteration during prototype phase.
* Simpler local development and testing.
* Cleaner error handling and call flow during early stages.
* Architecture remains “async-ready” by design.

### **Negative**

* Services remain coupled at runtime (availability dependency).
* No built-in retry or buffering until SQS is introduced.
* Long-running operations may eventually outgrow synchronous calls.

## **Future Considerations**

Move to SQS when:

* Workload spikes or batch processing emerge.
* Reliability/retry guarantees are required.
* Workflows become event-driven or fan-out.
* Processing time exceeds typical REST latency.

---
