---
title: "0013. Migration from MapStruct to Manual Mappers"
summary: "We implemented MapStruct to handle DTO-to-entity mapping, but it has caused significant build issues:"
---

**Status:** Accepted  
**Date:** 2026-01-30  
**Context:** MapStruct introduced build cache, CDI, annotation processor, and CI issues; manual mappers are proposed instead.

## **Context**

We implemented MapStruct to handle DTO-to-entity mapping, but it has caused significant build issues:

1. **Build cache problems**: Generated code interferes with incremental builds and Maven build cache
2. **CDI injection failures**: MapStruct-generated implementations sometimes fail to be recognized as CDI beans, causing `UnsatisfiedResolutionException`
3. **Annotation processing complexity**: Requires careful configuration of annotation processors, which can break with Maven/Gradle version changes
4. **CI/CD unpredictability**: Build failures due to stale generated code or annotation processing issues
5. **Debugging difficulty**: Generated code is harder to debug and reason about

The build issues have become more costly than the benefits MapStruct provides.

## Alternatives Considered

### Why Not Other Mapping Libraries?

We evaluated whether other 3rd party mapping libraries would solve our issues, but determined they would likely face similar problems:

#### Compile-Time Code Generation Libraries (Selma, JMapper)
- **Selma**: Similar annotation processing approach to MapStruct
  - Would likely have same build cache issues
  - Would likely have same CDI injection problems
  - Smaller community/less mature than MapStruct
- **JMapper**: Hybrid approach (runtime + optional compile-time)
  - Still uses annotation processing for compile-time mode
  - More complex configuration
  - Less active community

**Conclusion**: Other compile-time generators would likely face the same annotation processing and CDI integration issues that plagued MapStruct.

#### Runtime Reflection-Based Libraries (Dozer, ModelMapper, Orika)
- **Dozer**: Runtime reflection-based mapping
  - **Pros**: No annotation processing, no build issues
  - **Cons**:
    - Significant performance overhead (5-10x slower than MapStruct)
    - Runtime errors instead of compile-time safety
    - Less type safety
    - Declining community support
- **ModelMapper**: Convention-based runtime mapping
  - **Pros**: No annotation processing, easy configuration
  - **Cons**:
    - Runtime reflection overhead (6x slower than MapStruct)
    - Convention-based mapping can be unpredictable
    - Runtime errors for mismatched fields
- **Orika**: Runtime bytecode generation
  - **Pros**: Good performance after initial setup, no annotation processing
  - **Cons**:
    - Still uses reflection/bytecode generation at runtime
    - Less type safety than compile-time solutions
    - Moderate community support

**Conclusion**: Runtime reflection-based mappers avoid build issues but introduce:
- Performance overhead (unacceptable for high-throughput services)
- Loss of compile-time type safety
- Runtime errors that could be caught at compile-time
- Less predictable behavior

### Why Manual Mappers?

Given that:
1. **Compile-time generators** (MapStruct, Selma, JMapper) all use annotation processing → same build issues
2. **Runtime reflection mappers** (Dozer, ModelMapper, Orika) avoid build issues but introduce performance and type safety problems
3. **Manual mappers** provide:
   - Zero build complexity
   - Full type safety
   - Best performance (no reflection, no code generation)
   - Explicit, debuggable code
   - No external dependencies

The decision to use manual mappers was made because:
- The mapping logic is relatively simple (mostly field-to-field)
- The estimated boilerplate (~200-300 lines) is manageable
- Build stability and performance are critical
- Explicit code aligns with clean architecture principles

## Decision

We will migrate from MapStruct to **manual mapper classes** that explicitly handle conversions between DTOs and persistence entities.

### Rationale

1. **Build stability**: No annotation processing means no generated code, eliminating build cache issues
2. **Explicit and debuggable**: Manual code is easier to understand, debug, and maintain
3. **Type safety**: Java's type system provides compile-time safety without code generation
4. **Clean architecture alignment**: Explicit mappers fit better with our clean architecture approach
5. **Simple maintenance**: No special build configuration or annotation processor setup required
6. **Performance**: Manual mappers are as fast as MapStruct (both compile-time, no reflection)

### Trade-offs

**Pros:**
- Zero build-time code generation
- Predictable builds and CI/CD
- Easy to debug and test
- Full control over mapping logic
- No external dependencies for mapping

**Cons:**
- More boilerplate code (mitigated by keeping mappers small and focused)
- Manual updates when DTOs/entities change (caught by compiler)
- Slightly more code to maintain

## Implementation Strategy

### Phase 1: Create Reference Implementation
- Migrate `CandidateMapper` first as it's the simplest
- Establish patterns and conventions
- Validate approach works end-to-end

### Phase 2: Migrate Document Service Mappers
- `ResumeMapper` (most complex, has JSON processing)
- `JobSpecMapper`

### Phase 3: Cleanup
- Remove MapStruct dependencies
- Remove annotation processor configuration
- Clean up generated sources
- Update documentation

## Mapper Implementation Pattern

All manual mappers will follow this pattern:

```java
@ApplicationScoped
public class CandidateMapper {
    
    public ActorRecord toRecord(RegisterRequestWithAuthIdentity source) {
        // Explicit field mapping
    }
    
    public ActorResponse toActorResponse(ActorRecord source) {
        // Explicit field mapping
    }
}
```

### Principles

1. **One mapper class per aggregate**: Keep mappers focused on one domain concept
2. **Explicit null handling**: Handle nulls explicitly rather than relying on defaults
3. **Immutable where possible**: Prefer creating new objects over mutating
4. **Testable**: Each mapper method should be easily unit testable
5. **Documented**: Complex mappings should have comments explaining business logic

## Migration Checklist

- [ ] Create manual `CandidateMapper` implementation
- [ ] Update `CandidateService` to use new mapper
- [ ] Test actor-service end-to-end
- [ ] Create manual `ResumeMapper` implementation
- [ ] Create manual `JobSpecMapper` implementation
- [ ] Update `DocumentService` to use new mappers
- [ ] Test document-service end-to-end
- [ ] Remove MapStruct from root `pom.xml`
- [ ] Remove annotation processor paths from `maven-compiler-plugin`
- [ ] Clean up any generated sources in `target/` directories
- [ ] Update any documentation referencing MapStruct
- [ ] Verify all builds pass without MapStruct

## Consequences

### Positive
- Build stability and predictability
- Easier debugging and maintenance
- No annotation processing complexity
- Better alignment with clean architecture

### Negative
- More code to maintain (estimated ~200-300 lines total)
- Manual updates when DTOs change (but compiler catches these)

### Neutral
- Performance remains the same (both are compile-time, no reflection)
- Runtime behavior unchanged

## References

- [MapStruct Documentation](https://mapstruct.org/)
- [Quarkus CDI Guide](https://quarkus.io/guides/cdi)
- ADR-0012: Clean Architecture Package Structure
- [Java Mapping Libraries Comparison](https://www.frank-rahn.de/java-bean-mapper/) - Performance benchmarks and feature comparison
