---
title: "0011. Stateless JWT Authentication"
summary: "The application initially used a hybrid authentication system:"
---

**Status:** Accepted  
**Date:** 2025-12-10  
**Context:** Migrate from hybrid JWT plus sessions to fully stateless JWT authentication for horizontal scaling.

## **Context**

The application initially used a hybrid authentication system:
- Backend services used JWT-based authentication (stateless)
- UI modules used session-based authentication (stateful)
- Both mechanisms coexisted, with sessions taking precedence for UI

This hybrid approach created several issues:
- Required sticky sessions or shared session stores for horizontal scaling
- Increased complexity with two authentication mechanisms
- Session state management overhead
- Inconsistent authentication patterns across modules

The system needed to support true horizontal scaling without session affinity requirements.

---

## **Decision**

We will use **fully stateless JWT-based authentication** across all modules (backend services and frontend UI modules).

All authentication flows return JWT tokens that are:
- Stored client-side (localStorage for web applications)
- Included in API requests via `Authorization: Bearer <token>` header
- Validated server-side by `TokenAuthenticationFilter`
- Enforced by `@Secured` annotation via `UserTokenAuthorizationInterceptor`

### **Key Changes**

1. **Removed Session Support**:
   - Deleted `security-web` module
   - Removed `SessionAuthenticationFilter` ✅ (completed 2025-12-11)
   - Removed `SessionAuthResource` ✅ (completed 2025-12-11)
   - Removed `CognitoFormLoginResource` ✅ (completed 2025-12-11)
   - Removed `SessionAttribute` enum ✅ (completed 2025-12-11)
   - Removed `PageProtectionRouteHandler`
   - Removed session-related CORS configuration

2. **Unified Authentication Model**:
   - Single `@Secured` annotation for all authentication
   - JWT tokens for all modules (backend and frontend)
   - Client-side token storage and management
   - Automatic token validation via JAX-RS filter

3. **Frontend Authentication**:
   - Login returns JWT tokens (not session cookies)
   - Tokens stored in localStorage
   - JavaScript handles page protection
   - Automatic token refresh on expiration

---

## **Consequences**

**Positive:**

- **True Stateless**: No server-side session state required
- **Horizontal Scaling**: No sticky sessions or shared session stores needed
- **Simplified Architecture**: Single authentication mechanism across all modules
- **Better Performance**: No session lookups required
- **Mobile-Friendly**: JWT tokens work well for mobile applications
- **Consistent Security Model**: Same `@Secured` annotation everywhere

**Negative / Tradeoffs:**

- **Client-Side Token Management**: Frontend must handle token storage, expiration, and refresh
- **Token Security**: Tokens stored in localStorage are vulnerable to XSS attacks (mitigated by CloudFront CSP headers in the baseline)
- **CSRF**: Bearer JWTs are not auto-attached by the browser, so classic cookie CSRF does not apply to API calls; OAuth redirect flows still require a validated `state` parameter (LinkedIn)
- **No Server-Side Revocation**: JWT tokens cannot be revoked until expiration (acceptable tradeoff for stateless design)

---

## **Implementation**

### **Backend**

All REST endpoints use `@Secured` annotation:
- `TokenAuthenticationFilter` validates tokens from `Authorization` header
- `UserTokenAuthorizationInterceptor` enforces authorization for `@Secured` methods
- Returns 401 Unauthorized if token is missing or invalid

### **Frontend**

JavaScript handles authentication:
- Login via `POST /auth/login` (returns JWT tokens)
- Store tokens in localStorage
- Include `Authorization: Bearer <token>` in all API requests
- Check token expiration and refresh automatically
- Protect pages by checking for valid token on load

### **Request Flow**

All frontend requests route through BFF on port N:
- `POST /api/session` → `bff` → `auth-service`

---
