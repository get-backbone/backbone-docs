---
title: "0014. Application Caching Strategy"
summary: "The Backbone platform performs several expensive operations on every request or frequently accessed data:"
---

**Status:** Accepted  
**Date:** 2025-01-27  
**Context:** Application-level caching to improve performance and cut load and cost across single-instance dev and multi-instance production.

## **Context**

The Backbone platform performs several expensive operations on every request or frequently accessed data:

1. **Cognito Token Validation** - JWT parsing and signature validation via Cognito JWKS on every authenticated request
2. **User Profile Lookups** - PostgreSQL queries for candidate profiles on profile page loads and header username display
3. **Document Retrieval** - DynamoDB reads for parsed documents

Current state shows no caching layer, resulting in:
- High database query volume (PostgreSQL and DynamoDB)
- Repeated Cognito JWKS lookups and JWT parsing overhead
- Increased latency for frequently accessed data
- Higher operational costs (DynamoDB read capacity units, database connection overhead)

We must choose between:

1. **No caching** - Continue with direct database/external service calls
2. **Local caching only** - In-memory cache (Caffeine) for single-instance deployments
3. **Distributed caching** - Redis/ElastiCache for multi-instance deployments
4. **Hybrid approach** - Start with local caching, migrate to distributed for production

Additionally, distributed rate limiting uses the same Redis infrastructure for multi-replica correctness.

---

## **Decision**

Implement a **three-phase caching strategy** using Quarkus Cache:

1. **Phase 1: Local Caching (Caffeine)** — `%test` profile uses in-memory Caffeine for fast CI (complete)
2. **Phase 2: Metrics and Monitoring** — Cache metrics in Grafana dashboards (complete)
3. **Phase 3: Redis Backend (Production)** — `%dev`, `%int`, and `%prod` use `quarkus-redis-cache` with Redis (complete)

**Primary Use Cases:**
- Cache Cognito token validation results (TTL = token expiration)
- Cache actor profile data (TTL = 10 minutes, invalidate on profile updates)
- Cache parsed document results (TTL = 1 hour, documents are idempotent)

**Cache Implementation:**
- Use Quarkus Cache annotations (`@CacheResult`, `@CacheInvalidate`)
- Cache-aside pattern: reads populate the cache; updates invalidate existing entries; creates rely on lazy population on first read
- Fail-open strategy (cache failures don't break requests)

**Infrastructure:**
- `%dev`: Compose Redis (`localhost:6380`, plain) for prod-like local development
- `%test`: Caffeine (in-memory, no Redis dependency in CI)
- `%int` / `%prod`: Shared ElastiCache Redis cluster for distributed caching

---

## **Rationale**

### **Why Quarkus Cache**

- **Framework Integration** - Native Quarkus support with CDI annotations
- **Multiple Backends** - Supports Caffeine (local) and Redis (distributed) via configuration
- **Simple API** - `@CacheResult` and `@CacheInvalidate` annotations reduce boilerplate
- **Metrics** - Automatic Micrometer integration for cache metrics
- **Production Ready** - Battle-tested in Quarkus ecosystem

### **Why Three-Phase Approach**

- **Incremental Risk** - Start simple (local cache), add metrics, then migrate to distributed
- **Development First** - Local caching works immediately without infrastructure dependencies
- **Metrics Before Production** - Understand cache performance before distributed migration
- **Production Hardening** - Redis backend provides consistency across instances

### **Why Cache-Aside Pattern**

- **Simplicity** - Clear separation: reads populate cache, updates invalidate cache, creates populate lazily on first read
- **Consistency** - Database remains source of truth
- **Flexibility** - Easy to add/remove caching without changing business logic

### **Why Shared Redis Infrastructure**

- **Cost Optimization** - Single ElastiCache cluster serves caching + rate limiting
- **Operational Simplicity** - One infrastructure component to manage
- **Synergy** - Shared ElastiCache cluster also backs distributed HTTP rate limiting (see [Application caching and distributed scale](/docs/caching))

### **Why These Use Cases**

- **Token Validation** - Highest frequency (every authenticated request), high cryptographic overhead
- **Candidate Profiles** - Frequently accessed, low change rate, database query overhead
- **Parsed Documents** - Idempotent (safe to cache), DynamoDB read cost reduction

---

## **Architecture Overview**

### Phase 1: Local Caching (Caffeine)

```
┌─────────────────┐
│  Service        │
│  Instance       │
│                 │
│  ┌───────────┐  │
│  │ Caffeine  │  │
│  │ Cache     │  │
│  └───────────┘  │
│       │         │
└───────┼─────────┘
        │
        ├──> PostgreSQL (candidate profiles)
        ├──> DynamoDB (parsed documents)
        └──> Cognito (token validation)
```

### Phase 3: Distributed Caching (Redis)

```
┌─────────────────┐     ┌─────────────────┐
│  Service        │     │  Service        │
│  Instance 1     │     │  Instance 2     │
│                 │     │                 │
│  ┌───────────┐  │     │  ┌───────────┐  │
│  │ Quarkus   │  │     │  │ Quarkus   │  │
│  │ Cache     │  │     │  │ Cache     │  │
│  └─────┬─────┘  │     │  └─────┬─────┘  │
└────────┼────────┘     └────────┼────────┘
         │                       │
         └────────────┬──────────┘
                      │
              ┌───────▼───────┐
              │ ElastiCache   │
              │ Redis Cluster │
              │               │
              │ cache:*       │
              │ ratelimit:*   │
              └───────┬───────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   PostgreSQL    DynamoDB       Cognito
```

---

## **Implementation Details**

### Cache Configuration

**Token Validation Cache:**
- Cache name: `token-validation`
- Key: Composite key: hashed token reference plus expiry (so entries line up with JWT validity windows; never use the raw bearer token as all or part of the key).
- TTL: Token expiration time (automatic)
- Max size: 10,000 entries

**Actor Profile Cache:**
- Cache name: `actor-profiles`
- Key: `actor:{actorId}` via `ActorCacheKeyGenerator`
- TTL: 10 minutes
- Max size: 5,000 entries
- Population: `@CacheResult` on profile reads (`getActorProfileViaCache`)
- Invalidation: `@CacheInvalidate` on profile updates (`updateLinkedInSub`, `storeRefreshToken`)
- Registration (`register`): no cache annotation; new actors populate the cache on first read (cache miss). No invalidation is required because no prior entry exists for a new `actorId`.

**Document Cache:**
- Cache name: `parsed-resumes`, `parsed-jobspecs`
- Key: Composite key: document-type namespace plus candidate identifier or job-spec transaction identifier, depending on which artifact is cached
- TTL: 1 hour
- Max size: 10,000 entries
- Invalidation: On document re-upload

### Error Handling

- **Fail-Open Strategy** - Cache failures don't break requests
- **Negative Caching** - Don't cache null/empty results (except token validation)
- **Exception Handling** - Quarkus Cache handles exceptions gracefully, falls back to underlying operation

### Security Considerations

- **Token Caching** - Cache keys use token hashes (not full tokens)
- **PII in Cache** - Ensure Redis encryption at rest and in transit
- **Key Namespacing** - Use prefixes (`cache:token:`, `cache:candidate:`) to prevent collisions

---

## **Consequences**

### **Positive**

- **Performance** - Reduced latency for cached operations (20-50ms improvement)
- **Cost Reduction** - 20-30% reduction in DynamoDB read capacity, 30-40% reduction in PostgreSQL queries
- **Scalability** - Redis backend enables horizontal scaling with shared cache
- **Operational Efficiency** - Shared Redis infrastructure for caching + rate limiting
- **Developer Experience** - Simple annotations, no boilerplate cache management code

### **Negative**

- **Complexity** - Additional infrastructure component (Redis) in production
- **Cache Invalidation** - Must ensure cache invalidation on profile updates; creates rely on lazy population on first read
- **Memory Usage** - Local cache (Caffeine) consumes JVM heap memory
- **Cache Stampede** - Risk of thundering herd if cache expires simultaneously (mitigated by TTL variance)

### **Risks and Mitigations**

- **Cache Inconsistency** - Mitigated by cache-aside pattern (database is source of truth)
- **Cache Failures** - Mitigated by fail-open strategy (requests continue without cache)
- **Memory Pressure** - Mitigated by size limits and TTL-based eviction
- **Redis Availability** - Mitigated by ElastiCache high availability configuration

---

## **Success Metrics**

- Cache hit rate > 80% for token validation
- Cache hit rate > 60% for candidate profiles
- Cache hit rate > 70% for documents
- P95 latency reduction of 20-50ms for cached operations
- 20-30% reduction in DynamoDB read capacity units
- 30-40% reduction in PostgreSQL query volume

---

## **References**

- [Quarkus Cache Guide](https://quarkus.io/guides/cache)
- [Quarkus Redis Cache](https://quarkus.io/guides/cache-redis-reference)

---
