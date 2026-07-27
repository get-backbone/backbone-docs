---
title: "0017. Notification Service Template Storage"
summary: "The Notification Service requires template storage for email, SMS, and push notifications. Templates need to:"
---

**Status:** Accepted  
**Date:** 2026-01-23  
**Context:** Where and how notification templates (email, SMS, push) are stored, versioned, and scaled.

## **Context**

The Notification Service requires template storage for email, SMS, and push notifications. Templates need to:
- Be versioned and managed (draft → active)
- Support HTML and text formats
- Scale to high throughput (thousands of notifications per minute)
- Integrate with existing platform infrastructure (AWS, LocalStack)

**Template Storage Options:**
- **PostgreSQL** - Relational database, simple queries
- **DynamoDB** - NoSQL, scalable, serverless
- **S3** - Object storage, versioning support
- **Hybrid** - Metadata in DB, assets in S3

**Note:** Template engine selection is covered in [ADR-0018](/docs/0018-notification-service-template-engine).

---

## **Decision**

### **Template Storage: AWS DynamoDB + S3 (Hybrid)**

**Primary Storage: DynamoDB**
- Store template metadata and content in DynamoDB table `notification-templates`
- Partition Key: `template_id` (String)
- Sort Key: `version` (Number) for version history
- Attributes: channel, activation_status, metadata, content (subject, html_body, text_body, placeholders)

**Large Assets: S3**
- Store large HTML bodies, images, attachments in S3
- DynamoDB item contains S3 reference (optional)
- S3 Object Versioning for asset versioning

**Why DynamoDB:**
- **Scalability** - Fully managed, horizontally scalable NoSQL
- **Security** - Fine-grained AWS IAM permissions
- **Concurrency** - Conditional writes prevent overwrite conflicts
- **Integration** - Native AWS integration (Lambda, AppConfig, API Gateway)
- **Reliability** - Point-in-Time Recovery (PITR)
- **Existing Infrastructure** - Platform already uses DynamoDB (via LocalStack)

**Why S3 for Assets:**
- **Large Content** - Better for large HTML bodies, images
- **Versioning** - S3 Object Versioning for asset history
- **Cost** - Lower cost for large files

---

## **Rationale**

### **Why DynamoDB (Not PostgreSQL):**

1. **Scalability** - Better suited for high-throughput read patterns (template lookups)
2. **Serverless** - Fully managed, no connection pooling concerns
3. **Existing Infrastructure** - Platform already uses DynamoDB
4. **Versioning** - Natural fit for versioned templates (PK/SK pattern)
5. **Multi-Region** - Easier multi-region deployment

### **Why Not PostgreSQL:**

1. **Read-Heavy** - Templates are read-heavy (lookup by templateId), DynamoDB optimized for this
2. **Simple Queries** - No complex joins needed, DynamoDB sufficient
3. **Scaling** - DynamoDB scales automatically, PostgreSQL requires more management

---

## **Consequences**

### **Positive:**

- **Scalability** - DynamoDB scales automatically for high throughput
- **Integration** - Native AWS integration (DynamoDB, S3)
- **Versioning** - Natural versioning support (DynamoDB + S3)
- **Cost** - Efficient storage (DynamoDB for metadata, S3 for large assets)
- **Existing Infrastructure** - Platform already uses DynamoDB

### **Negative / Tradeoffs:**

- **DynamoDB Learning Curve** - Team must understand DynamoDB patterns
- **Query Limitations** - DynamoDB has limited query capabilities (GSI required for complex queries)
- **S3 Integration** - Requires S3 client setup for large assets
- **LocalStack** - Must ensure LocalStack DynamoDB support for local development

### **Mitigations:**

- **Documentation** - Clear DynamoDB schema and access patterns
- **GSI** - Use Global Secondary Indexes for query patterns (e.g., by channel)
- **LocalStack** - LocalStack fully supports DynamoDB
- **Examples** - Provide examples of template storage and retrieval

---

## **Implementation**

### **DynamoDB Schema:**

**Table: `notification-templates`**
- **Partition Key:** `template_id` (String)
- **Sort Key:** `version` (Number)
- **Attributes:**
  - `channel` (String) - EMAIL, SMS, PUSH
  - `activation_status` (String) - ACTIVE, DRAFT, ARCHIVED
  - `metadata` (Map) - tags, owner, last_modified
  - `content` (Map) - subject, html_body, text_body, placeholders
  - `s3_reference` (String) - Optional S3 key for large assets

**GSI: `channel-index`**
- Partition Key: `channel`
- Sort Key: `template_id`
- For listing templates by channel

### **Qute Template Rendering:**

```java
// Retrieve template from DynamoDB
Template template = templateRepository.getTemplate("welcome-email-v1");

// Render with Qute
String rendered = Qute.fmt(template.getHtmlBody())
    .data("firstName", "John")
    .data("activationLink", "https://example.com/activate?token=abc123")
    .render();
```

### **Template Storage Pattern:**

1. **Small Templates** - Store directly in DynamoDB `content.html_body`
2. **Large Templates** - Store in S3, reference in DynamoDB `s3_reference`
3. **Versioning** - New version = new DynamoDB item (same template_id, different version)
4. **Activation** - `activation_status` field indicates active version

---

## **Related Decisions**

- **ADR-0012:** Clean Architecture Package Structure (package organization)
- **ADR-0015:** Fire-and-Forget / Asynchronous Messaging Pattern (async processing)
- **ADR-0018:** Notification Service Template Engine Selection (template rendering engine)

---

## **Future Considerations**

**Template Versioning:**
- Current: Immutable templates (new version = new item)
- Future: May add explicit versioning strategy if needed

**Template Management UI:**
- Future: Admin UI for template management
- Current: API-only or seed scripts

**Multi-Language Support:**
- Future: Per-template locales or locale variable
- Current: Single language

---
