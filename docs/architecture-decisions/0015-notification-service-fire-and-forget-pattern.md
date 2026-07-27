---
title: "0015. Notification Service Fire-and-Forget / Asynchronous Messaging Pattern"
summary: "The Notification Service is being designed as a centralized, multi-channel notification delivery service"
---

**Status:** Accepted  
**Date:** 2026-01-23  
**Context:** Centralized Notification Service design: how requests are acknowledged vs delivered asynchronously at scale.

## **Context**

The Notification Service is being designed as a centralized, multi-channel notification delivery service (email, SMS, push notifications). A key architectural decision is how the service handles notification requests and delivery.

**Options Considered:**

1. **Synchronous Delivery** - Client waits for notification to be delivered before receiving response
2. **Fire-and-Forget / Asynchronous Messaging** - Client receives immediate acknowledgment, delivery happens asynchronously
3. **Hybrid** - Synchronous for critical, asynchronous for bulk

**Requirements:**
- High throughput (thousands of notifications per minute)
- Priority-based processing (critical notifications must not be blocked)
- Graceful degradation under load
- Stateless service design (following ADR-0011)
- Eventually-consistent delivery guarantees

---

## **Decision**

The Notification Service will follow the **Fire-and-Forget** (also known as **Asynchronous Messaging**) pattern with **eventual consistency**.

### **Core Principles:**

1. **Immediate Acknowledgment**
   - Client sends notification request
   - Service immediately returns `201 Created` with notification ID
   - Client does **not** wait for delivery confirmation
   - Client only receives confirmation that the notification was **accepted** by the system

2. **Asynchronous Processing**
   - Notifications are queued and processed asynchronously
   - Processing happens at a later time (decoupled from client request)
   - Delivery time depends on system load, priority, and provider availability

3. **Eventually-Consistent**
   - Notifications are guaranteed to be delivered eventually, but not immediately
   - No immediate delivery guarantee
   - Delivery status must be checked via status endpoint
   - No synchronous waiting for delivery confirmation

4. **Priority-Based Processing**
   - High-priority notifications (e.g., password resets) processed before low-priority (e.g., marketing)
   - When system approaches capacity, low-priority notifications may be delayed
   - Priority ensures critical notifications are not blocked by bulk operations

### **Priority Levels:**

- **CRITICAL** - Security-critical (password resets, account lockouts) - processed immediately
- **HIGH** - Transactional (order confirmations, activation emails) - processed within seconds
- **NORMAL** - Standard notifications (welcome emails, updates) - processed within minutes
- **LOW** - Marketing, newsletters - processed when capacity available

---

## **Rationale**

### **Why Fire-and-Forget:**

1. **Scalability** - Decouples client from delivery, enabling high throughput
2. **Fault Tolerance** - Client doesn't block on provider failures
3. **Resource Efficiency** - Better utilization of system resources
4. **Priority Support** - Enables priority-based processing without blocking clients

### **Why Eventually-Consistent:**

1. **High Load Handling** - System can handle spikes without blocking
2. **Graceful Degradation** - Low-priority notifications can be delayed during high load
3. **Provider Constraints** - External providers (SES, Twilio) have rate limits that require queuing
4. **Retry Logic** - Failed notifications can be retried without client involvement

### **Why Not Synchronous:**

1. **Blocking** - Clients would wait for provider responses (SES, Twilio), increasing latency
2. **Coupling** - Client availability depends on provider availability
3. **No Priority** - Cannot prioritize critical notifications over bulk operations
4. **Poor Scalability** - Limited by slowest provider response time

---

## **Consequences**

### **Positive:**

- **High Throughput** - System can handle thousands of notifications per minute
- **Better Scalability** - Horizontal scaling without client coordination
- **Fault Tolerance** - Provider failures don't block clients
- **Priority Support** - Critical notifications always processed first
- **Graceful Degradation** - System degrades gracefully under load
- **Stateless Design** - Aligns with ADR-0011 (stateless services)

### **Negative / Tradeoffs:**

- **No Immediate Delivery Guarantee** - Clients cannot assume immediate delivery
- **Status Checking Required** - Clients must check delivery status if confirmation needed
- **Potential Delays** - Low-priority notifications may experience delays during high load
- **Complexity** - Requires delivery tracking, retry logic, and status endpoints

### **Mitigations:**

- **Status Endpoint** - `GET /notifications/{notificationId}` for delivery status
- **Priority System** - Ensures critical notifications are not delayed
- **Retry Logic** - Failed notifications are automatically retried
- **Monitoring** - Metrics and dashboards for delivery tracking

---

## **Implementation**

### **API Design:**

**Request Flow:**
1. Client sends `POST /notifications` with notification details
2. Service validates request, creates notification record (status: QUEUED)
3. Service immediately returns `201 Created` with notification ID
4. Service processes notification asynchronously (renders template, calls provider)
5. Service updates notification status (SENT, DELIVERED, FAILED, etc.)

**Status Checking:**
- Client can check delivery status via `GET /notifications/{notificationId}`
- Status endpoint returns current delivery status and events

### **Processing Flow:**

1. **Queue** - Notification created with status QUEUED
2. **Process** - Asynchronous processor picks up notification (priority-ordered)
3. **Render** - Template retrieved and rendered with variables
4. **Send** - Provider called (SES, Twilio, etc.)
5. **Track** - Delivery status updated based on provider response/webhooks

### **Priority-Based Processing:**

Notifications are processed in priority order:
- CRITICAL → HIGH → NORMAL → LOW
- Within same priority, FIFO (first-in-first-out)
- Low-priority notifications may be throttled/paused during high load

---

## **Related Decisions**

- **ADR-0008:** REST vs SQS (synchronous REST API, async-ready design)
- **ADR-0011:** Stateless JWT Authentication (stateless service design)
- **ADR-0010:** REST API Design Standards (RESTful endpoint design)

---

## **Future Considerations**

**SQS Integration (Phase 2):**
- When SQS is implemented (per ADR-0008), notification queuing can move to SQS
- Service remains fire-and-forget, but queue moves from database to SQS
- Enables better scalability and retry handling

**Real-Time Delivery Status:**
- Future: WebSocket or Server-Sent Events for real-time delivery status
- Current: Polling via status endpoint

---
