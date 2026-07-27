---
title: "0018. Notification Service Template Engine Selection"
summary: "The Notification Service requires a template engine to render notifications (email, SMS, push) with variable substitution. Templates need to:"
---

**Status:** Accepted  
**Date:** 2026-01-23  
**Context:** Choose a template engine for rendering notifications with variables, HTML/text, and Quarkus integration.

## **Context**

The Notification Service requires a template engine to render notifications (email, SMS, push) with variable substitution. Templates need to:
- Support variable substitution (e.g., `{{firstName}}`, `{{activationLink}}`)
- Support HTML and text formats
- Handle loops and conditionals for complex templates
- Scale to high throughput (thousands of notifications per minute)
- Integrate with Quarkus (the platform's microservice framework per ADR-0002)

**Template Engine Options Considered:**

1. **Simple String Replacement** - Custom `{{variable}}` replacement
   - Pros: Simple, no dependencies
   - Cons: Limited functionality, hard to maintain for complex logic (loops, conditionals), no validation

2. **Mustache** - Logic-less templates
   - Pros: Simple syntax, widely used
   - Cons: Logic-less (no conditionals/loops), requires external dependency

3. **Handlebars** - Mustache with helpers
   - Pros: More powerful than Mustache, supports helpers
   - Cons: Requires external dependency, not Quarkus-native

4. **Qute** - Quarkus native templating engine
   - Pros: Built-in to Quarkus, high performance, async support, type-safe
   - Cons: Quarkus-specific (not portable to other frameworks)

5. **Thymeleaf** - Full-featured, Spring-like
   - Pros: Very powerful, Spring ecosystem
   - Cons: Spring-oriented, heavier, not Quarkus-native

---

## **Decision**

The Notification Service will use **Qute** as the template engine.

### **Qute Features:**

- **Native Integration** - Built-in to Quarkus, no extra dependencies
- **Performance** - Designed for high throughput, low memory footprint
- **Async Support** - Fully non-blocking, perfect for reactive applications
- **Type Safety** - Validates templates at build time (if using typed templates) or efficient runtime rendering
- **Syntax** - Mustache-like syntax but more powerful (loops, conditionals)
- **Developer Experience** - Idiomatic Quarkus choice

---

## **Rationale**

### **Why Qute:**

1. **Native Integration** - Built-in to Quarkus (ADR-0002: Quarkus adoption)
   - No external dependencies required
   - Seamless integration with Quarkus ecosystem
   - Consistent with platform's Quarkus-first approach

2. **Performance** - Designed for Quarkus performance goals
   - High throughput, low memory footprint
   - Optimized for serverless and containerized deployments
   - Non-blocking I/O support

3. **Async Support** - Fully non-blocking
   - Perfect for reactive applications
   - Aligns with fire-and-forget pattern (ADR-0015)
   - Supports high concurrency

4. **Type Safety** - Compile-time validation (if using typed templates)
   - Catches template errors at build time
   - Better developer experience
   - Reduces runtime errors

5. **Platform Consistency** - Uses Quarkus-native tools
   - Consistent with platform architecture
   - Reduces technology diversity
   - Easier for team to maintain

### **Why Not Simple String Replacement:**

1. **Limited Functionality** - Hard to maintain for complex logic (loops, conditionals)
2. **No Validation** - No template validation, more runtime errors
3. **Maintenance** - More custom code to maintain
4. **Scalability** - Less efficient for high-throughput scenarios

### **Why Not Mustache/Handlebars:**

1. **External Dependency** - Requires additional dependencies
2. **Not Quarkus-Native** - Not optimized for Quarkus performance goals
3. **Less Integration** - Less seamless with Quarkus ecosystem
4. **Technology Diversity** - Adds another technology to the stack

### **Why Not Thymeleaf:**

1. **Spring-Oriented** - Designed for Spring ecosystem, not Quarkus
2. **Heavier** - More features than needed, higher memory footprint
3. **Not Quarkus-Native** - Not optimized for Quarkus

---

## **Consequences**

### **Positive:**

- **No External Dependencies** - Built-in to Quarkus, reduces dependency management
- **High Performance** - Optimized for Quarkus performance goals
- **Async Support** - Non-blocking, supports high concurrency
- **Type Safety** - Compile-time validation reduces runtime errors
- **Developer Experience** - Idiomatic Quarkus choice, easier for team
- **Platform Consistency** - Aligns with Quarkus-first architecture

### **Negative / Tradeoffs:**

- **Quarkus-Specific** - Not portable to other frameworks (acceptable, platform is Quarkus-based)
- **Learning Curve** - Team must learn Qute syntax (minimal, similar to Mustache)
- **Less Mature** - Less mature than Mustache/Handlebars (acceptable, Quarkus-native)

### **Mitigations:**

- **Documentation** - Clear Qute template examples and patterns
- **Examples** - Provide template examples for common use cases
- **Training** - Qute syntax is similar to Mustache, minimal learning curve

---

## **Implementation**

### **Template Rendering:**

```java
// Retrieve template from storage (DynamoDB/S3)
Template template = templateRepository.getTemplate("welcome-email-v1");

// Render with Qute
String rendered = Qute.fmt(template.getHtmlBody())
    .data("firstName", "John")
    .data("activationLink", "https://example.com/activate?token=abc123")
    .render();
```

### **Template Syntax Example:**

```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome {{firstName}}!</title>
</head>
<body>
    <h1>Welcome {{firstName}}!</h1>
    <p>Click the link below to activate your account:</p>
    <a href="{{activationLink}}">Activate Account</a>
    
    {#if showPromo}
    <p>Special offer: {{promoCode}}</p>
    {/if}
    
    {#for item in items}
    <p>{{item.name}}: {{item.price}}</p>
    {/for}
</body>
</html>
```

### **Integration Points:**

- **Template Storage** - Templates retrieved from DynamoDB/S3 (ADR-0017)
- **Rendering** - Qute renders templates with variable substitution
- **Async Processing** - Qute supports async rendering for fire-and-forget pattern (ADR-0015)

---

## **Related Decisions**

- **ADR-0002:** Adoption of Quarkus (Quarkus-native tools)
- **ADR-0015:** Fire-and-Forget / Asynchronous Messaging Pattern (async processing)
- **ADR-0017:** Notification Service Template Storage (DynamoDB + S3)

---

## **Future Considerations**

**Typed Templates:**
- Current: Runtime template rendering
- Future: May use typed templates for compile-time validation

**Template Caching:**
- Current: Templates retrieved from storage per request
- Future: May cache compiled templates for performance

**Template Validation:**
- Current: Runtime validation
- Future: May add template validation at storage time

---
