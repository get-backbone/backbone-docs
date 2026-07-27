---
title: "0019. Notification Service Unsubscribe Token Security"
summary: "The Notification Service must provide a secure unsubscribe mechanism that allows recipients to opt out of"
---

**Status:** Accepted  
**Date:** 2026-01-23  
**Context:** Secure, login-free unsubscribe links with tamper resistance, auditability, and revocation.

## **Context**

The Notification Service must provide a secure unsubscribe mechanism that allows recipients to opt out of notifications without requiring authentication. Unsubscribe links are included in every notification (email, SMS, push) and must be:

- **Secure** - Cannot be forged or tampered with
- **User-Friendly** - No login required, single-click unsubscribe
- **Compliant** - Meets enterprise compliance and audit requirements
- **Revocable** - Can be revoked or rotated if compromised
- **Auditable** - Track usage, IP, timestamp for compliance evidence

**Token Approach Options Considered:**

1. **JWT (JSON Web Token)**
   - Pros: Stateless, no database lookup required
   - Cons: Long URLs, hard to revoke, embedded data reveals information if leaked, complex key management

2. **HMAC-Signed URL**
   - Pros: Stateless, shorter URLs than JWT, tamper-proof
   - Cons: Hard to revoke, limited expressiveness, medium auditability

3. **Opaque Token + Server Lookup (Selected)**
   - Pros: Strongest security, easy to revoke, excellent auditability, flexible
   - Cons: Stateful (requires database), slightly more operational overhead

---

## **Decision**

The Notification Service will use **Opaque Token + Server Lookup** for unsubscribe links.

### **Pattern:**

1. **Token Generation**
   - Generate random opaque token (UUID or ULID) for each unsubscribe link
   - Token is cryptographically random, no embedded data
   - Token is unique per recipient/channel combination

2. **Token Storage**
   - Store token server-side in DynamoDB table `notification-unsubscribe-tokens`
   - Store with:
     - Recipient identifier (email/phone)
     - Scope (channel, or all channels)
     - Expiry (e.g., 30-90 days)
     - Usage count (single-use or idempotent)
     - Metadata (created timestamp, IP address if available)

3. **Unsubscribe URL**
   - Include token in unsubscribe URL: `/notifications/unsubscribe?token={opaque-token}`
   - No PII in URL (no email/phone in URL)
   - HTTPS only

4. **Validation**
   - Validate by server-side lookup before processing
   - Check token exists, not expired, not already used (if single-use)
   - Track usage (IP, timestamp, user agent) for audit

---

## **Rationale**

### **Why Opaque Token (Not JWT):**

1. **Security** - Token reveals nothing if leaked (no embedded data)
   - JWT contains embedded data (email, scope) that can be decoded
   - Opaque token is meaningless without server lookup

2. **Revocability** - Easy to revoke or rotate server-side
   - JWT cannot be revoked without blacklist (adds complexity)
   - Opaque token can be deleted from database immediately

3. **Auditability** - Track usage, IP, timestamp for compliance
   - JWT provides limited audit trail (must log at validation time)
   - Opaque token can store rich metadata (IP, user agent, usage count)

4. **Compliance** - Excellent audit trail for enterprise requirements
   - Regulatory requirements (GDPR, CAN-SPAM) require audit trails
   - Opaque token enables comprehensive tracking

5. **Flexibility** - Can evolve behavior without breaking old links
   - JWT structure is fixed once issued
   - Opaque token can change validation logic without breaking existing tokens

6. **Single-Use** - Can enforce single-use semantics to prevent replay attacks
   - JWT cannot enforce single-use without state
   - Opaque token can track usage count and enforce single-use

### **Why Opaque Token (Not HMAC-Signed URL):**

1. **Revocability** - HMAC-signed URLs cannot be revoked without blacklist
2. **Auditability** - Limited audit trail (must log at validation time)
3. **Expressiveness** - Limited ability to store metadata
4. **Compliance** - Less comprehensive audit trail than opaque tokens

### **Why Not Stateless (JWT/HMAC):**

1. **Revocation** - Stateless tokens cannot be revoked without blacklist (adds complexity)
2. **Auditability** - Limited audit trail for compliance
3. **Security** - Embedded data in JWT reveals information if leaked
4. **Flexibility** - Fixed structure once issued, cannot evolve behavior

### **Tradeoff: Stateful vs Stateless**

- **Stateful (Opaque Token):** Requires database lookup, but provides:
  - Revocability
  - Comprehensive audit trail
  - Flexibility
  - Single-use enforcement

- **Stateless (JWT/HMAC):** No database lookup, but:
  - Cannot revoke without blacklist
  - Limited audit trail
  - Less flexible
  - Harder to enforce single-use

**Decision:** Stateful approach is preferred for enterprise-grade unsubscribe mechanism due to security, compliance, and flexibility benefits.

---

## **Consequences**

### **Positive:**

- **Strongest Security** - Token reveals nothing if leaked, easy to revoke
- **Excellent Auditability** - Comprehensive audit trail for compliance
- **Revocability** - Can revoke or rotate tokens immediately
- **Compliance** - Meets enterprise compliance requirements (GDPR, CAN-SPAM)
- **Flexibility** - Can evolve behavior without breaking old links
- **Single-Use Enforcement** - Can prevent replay attacks

### **Negative / Tradeoffs:**

- **Stateful** - Requires database lookup (acceptable tradeoff for security/compliance)
- **Operational Overhead** - Requires DynamoDB table and cleanup (TTL handles this)
- **Latency** - Database lookup adds minimal latency (acceptable for unsubscribe flow)

### **Mitigations:**

- **TTL** - DynamoDB TTL automatically cleans up expired tokens
- **Caching** - Can cache active tokens for performance (optional)
- **Indexing** - DynamoDB provides fast lookups by token (PK lookup)

---

## **Implementation**

### **DynamoDB Schema:**

**Table: `notification-unsubscribe-tokens`**
- **Partition Key:** `token` (String) - Opaque UUID/ULID
- **Attributes:**
  - `recipient_email` (String)
  - `recipient_phone` (String)
  - `scope` (String) - Channel (EMAIL, SMS, PUSH) or "ALL"
  - `expiry` (Number) - Unix timestamp
  - `usage_count` (Number) - For single-use enforcement
  - `created_at` (Number) - Unix timestamp
  - `metadata` (Map) - IP address, user agent, etc.
- **TTL:** `expiry` attribute for automatic cleanup

### **Token Generation:**

```java
// Generate opaque token
String token = UUID.randomUUID().toString();
// or
String token = UlidCreator.getUlid().toString();

// Store in DynamoDB
UnsubscribeToken unsubscribeToken = UnsubscribeToken.builder()
    .token(token)
    .recipientEmail("user@example.com")
    .scope("EMAIL")
    .expiry(Instant.now().plus(90, ChronoUnit.DAYS).getEpochSecond())
    .usageCount(0)
    .createdAt(Instant.now().getEpochSecond())
    .build();

tokenRepository.save(unsubscribeToken);
```

### **Unsubscribe URL:**

```
https://api.example.com/notifications/unsubscribe?token={opaque-token}
```

### **Validation Flow:**

1. Receive unsubscribe request with token
2. Lookup token in DynamoDB
3. Validate:
   - Token exists
   - Not expired (`expiry > now`)
   - Not already used (if single-use: `usage_count == 0`)
4. Process unsubscribe:
   - Update recipient preferences (PostgreSQL)
   - Increment `usage_count` (if single-use)
   - Log audit trail (IP, timestamp, user agent)
5. Return success page

### **Security Considerations:**

- **Single-Purpose Scope** - Token is for unsubscribe only, not authentication
- **Idempotent Operations** - Safe to click multiple times (idempotent)
- **Rate Limiting** - Rate limit on unsubscribe endpoint to prevent abuse
- **HTTPS Only** - All unsubscribe URLs must use HTTPS
- **No PII in URLs** - No email/phone in URL, only opaque token
- **Graceful Success Page** - No error leakage (always show success page)
- **Do Not Require Login** - Unsubscribe must work without authentication

---

## **Related Decisions**

- **ADR-0017:** Notification Service Template Storage (DynamoDB for token storage)
- **ADR-0011:** Stateless JWT Authentication (stateless service design, but unsubscribe is intentionally stateful for security/compliance)

---

## **Future Considerations**

**Token Rotation:**
- Current: Tokens expire after 30-90 days
- Future: May add token rotation for long-lived tokens

**Distributed Token Storage:**
- Current: Single DynamoDB table
- Future: May need multi-region token storage for global deployments

**Token Analytics:**
- Current: Basic metadata (IP, timestamp, user agent)
- Future: May add more analytics (geolocation, device type, etc.)

---
