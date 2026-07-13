# 09 — Security Design Document

## 1. Security Architecture Overview

The application enforces security at three distinct layers, ensuring that threats are intercepted as early as possible in the request lifecycle.

### Spring Security FilterChain — Stateless Configuration

The `SecurityFilterChain` is configured with `SessionCreationPolicy.STATELESS`. No HTTP session is created or used. Every request must carry its own JWT for authentication. The `JwtAuthenticationFilter` runs before `UsernamePasswordAuthenticationFilter` in the filter chain.

```
Inbound Request
    ↓
Network Layer (HTTPS + CORS)
    ↓
Authentication Layer (JwtAuthenticationFilter → BCrypt verification at login)
    ↓
Authorization Layer (@PreAuthorize on controllers/services)
    ↓
Controller / Business Logic
```

### Layer 1 — Network Layer

- **HTTPS**: All production traffic is served over HTTPS only. HTTP is redirected to HTTPS at the load-balancer or reverse-proxy level.
- **CORS**: Controlled via `CorsConfigurationSource` bean (see CORS Configuration below). Only explicitly whitelisted origins can make cross-origin requests.

### Layer 2 — Authentication Layer

- **JWT Filter**: Every request passes through `JwtAuthenticationFilter`, which extracts and validates the Bearer token.
- **BCrypt Verification**: At login time, the submitted password is verified against the BCrypt hash stored in the database using `PasswordEncoder.matches()`.

### Layer 3 — Authorization Layer

- **@PreAuthorize**: Method-level security annotations on controller methods enforce role-based access control. Spring Security evaluates these after the `SecurityContext` has been populated by the JWT filter.
- **Global rule**: All endpoints require authentication unless explicitly listed in the permit-all configuration.

### CORS Configuration

| Property | Value |
|---|---|
| `allowedOrigins` | Read from env var `CORS_ALLOWED_ORIGINS` (comma-separated) |
| `allowedMethods` | GET, POST, PUT, PATCH, DELETE, OPTIONS |
| `allowedHeaders` | `Authorization`, `Content-Type` |
| `allowCredentials` | `false` |

`allowCredentials` is false because the API uses `Authorization: Bearer` headers (not cookies). Setting this to true would require restricted origins and is not needed for this stateless JWT design.

`CORS_ALLOWED_ORIGINS` must be set in production. In local development it defaults to `http://localhost:3000`.

---

## 2. JWT Implementation Details

### Library

```
io.jsonwebtoken:jjwt-api:0.12.x
io.jsonwebtoken:jjwt-impl:0.12.x  (runtime)
io.jsonwebtoken:jjwt-jackson:0.12.x  (runtime)
```

### Token Structure

**Access Token**

| Claim | Value | Description |
|---|---|---|
| `sub` | username | Subject — the authenticated user |
| `role` | e.g. `"ADMIN"` | User's role (single role per user) |
| `iat` | Unix epoch seconds | Issued-at timestamp |
| `exp` | Unix epoch seconds | Expiry — 24 hours after `iat` |

**Refresh Token**

Same claim structure (`sub`, `role`, `iat`, `exp`) but `exp` is set 7 days after `iat`. The refresh token is used exclusively at `POST /api/auth/refresh` to issue a new access token. It must not be used to access protected resources directly.

### Secret Key

- Minimum size: **256 bits (32 bytes)**
- Encoding: Base64-encoded string stored in environment variable `JWT_SECRET`
- The key is decoded at startup via `Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret))`
- Algorithm: `HS256` (HMAC-SHA256)

If `JWT_SECRET` is absent or shorter than 32 bytes after decoding, the application must fail to start with a clear error message.

### JwtUtil Class — Public API

| Method | Signature | Description |
|---|---|---|
| `generateAccessToken` | `String generateAccessToken(UserDetails userDetails)` | Creates a signed JWT with `sub`, `role`, `iat`, `exp` (24h). Uses `JWT_SECRET` for signing. |
| `generateRefreshToken` | `String generateRefreshToken(UserDetails userDetails)` | Same structure but `exp` set to 7 days. |
| `extractUsername` | `String extractUsername(String token)` | Extracts the `sub` claim. Throws `JwtException` on malformed token. |
| `extractRole` | `String extractRole(String token)` | Extracts the `role` claim from the token body. |
| `isTokenValid` | `boolean isTokenValid(String token, UserDetails userDetails)` | Returns `true` if: signature is valid, token is not expired, and `sub` matches `userDetails.getUsername()`. |
| `isTokenExpired` | `boolean isTokenExpired(String token)` | Returns `true` if `exp` is before `new Date()`. |
| `extractExpiration` | `Date extractExpiration(String token)` | Returns the `Date` value of the `exp` claim. |

---

## 3. Password Security

### BCrypt Configuration

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}
```

**Cost factor 12** results in approximately 300 ms per hash on modern hardware. This is intentional — it makes brute-force and credential-stuffing attacks computationally expensive for an attacker while remaining acceptable for normal login flows.

### Password Handling Rules

| Rule | Implementation |
|---|---|
| Passwords are NEVER logged | MDC context excludes `password` fields; no logging statement references raw password |
| Passwords are NEVER returned in API responses | `UserResponse` DTO contains no `password` field |
| Passwords are stored as BCrypt hashes only | `UserService.createUser()` calls `passwordEncoder.encode()` before persisting |
| No plaintext passwords in the codebase | No hardcoded credentials; test passwords are in `application-test.yml` only |

### Password Change

Not implemented in v1. The endpoint is excluded from the API specification. Future implementation must enforce old-password verification before accepting a new password.

---

## 4. Authorization Matrix

`Y` = permitted, `N` = denied, `(open)` = no authentication required.

| Endpoint | ADMIN | DEVOPS_ENGINEER | DEVELOPER |
|---|---|---|---|
| POST /api/auth/register | Y (open) | Y (open) | Y (open) |
| POST /api/auth/login | Y (open) | Y (open) | Y (open) |
| POST /api/auth/refresh | Y (open) | Y (open) | Y (open) |
| GET /api/deployments | Y | Y | Y |
| POST /api/deployments | Y | Y | N |
| GET /api/deployments/{id} | Y | Y | Y |
| PUT /api/deployments/{id} | Y | Y | N |
| DELETE /api/deployments/{id} | Y | N | N |
| PATCH /api/deployments/{id}/status | Y | Y | N |
| POST /api/deployments/{id}/rollback | Y | Y | N |
| GET /api/deployments/{id}/rollbacks | Y | Y | Y |
| GET /api/dashboard/stats | Y | Y | Y |
| GET /api/dashboard/success-rate | Y | Y | Y |
| GET /api/dashboard/recent | Y | Y | Y |
| GET /api/users | Y | N | N |
| GET /api/users/{id} | Y | N | N |
| PUT /api/users/{id}/role | Y | N | N |
| DELETE /api/users/{id} | Y | N | N |
| GET /api/audit-logs | Y | N | N |

**Key rules derived from the matrix:**

- All three roles can read deployments and dashboard data (read-only access is broad).
- Only ADMIN and DEVOPS_ENGINEER can create, modify, or trigger deployments.
- Only ADMIN can delete deployments, manage users, and view audit logs.
- Auth endpoints are fully open — registration is self-service. An ADMIN role cannot be self-assigned; role assignment requires `PUT /api/users/{id}/role` which is ADMIN-only.

---

## 5. JwtAuthenticationFilter Flow

The filter extends `OncePerRequestFilter` and executes once per request.

```
Request arrives
    |
    v
Extract "Authorization" header
    |
    v
Does header start with "Bearer "?
  |                        |
  NO                       YES
  |                        |
  v                        v
Continue filter chain  Extract token = header.substring(7)
(Spring Security will      |
reject with 401 if         v
endpoint requires      JwtUtil.extractUsername(token) --> username
auth)                      |
                           v
           SecurityContextHolder.getContext().getAuthentication() == null?
             |                                        |
             NO                                       YES
             |                                        |
             v                                        v
       Token already                  UserDetailsService.loadUserByUsername(username)
       processed,                          --> UserDetails
       continue chain                          |
                                               v
                                   JwtUtil.isTokenValid(token, userDetails)?
                                     |                          |
                                     NO                         YES
                                     |                          |
                                     v                          v
                               Continue chain          Create UsernamePasswordAuthenticationToken(
                               without setting             userDetails, null,
                               authentication              userDetails.getAuthorities())
                               (results in 401)            |
                                                           v
                                                   Set token in SecurityContextHolder
                                                           |
                                                           v
                                                   Continue filter chain
                                                           |
                                                           v
                                           Reaches controller
                                                           |
                                                           v
                                           @PreAuthorize checks role
                                           from SecurityContext
```

**Error handling within the filter:**

- If `extractUsername` throws a `JwtException` (malformed token), the filter catches the exception, does not set the `SecurityContext`, and continues the chain. The downstream security interceptor returns `401 Unauthorized`.
- If `loadUserByUsername` throws `UsernameNotFoundException`, same behavior — no `SecurityContext` populated, chain continues to 401.

---

## 6. OWASP Top 10 Mitigations

| OWASP Category | Risk | Mitigation Implemented |
|---|---|---|
| **A01 — Broken Access Control** | Users accessing resources or performing actions beyond their role | `@PreAuthorize` on all write endpoints; role check enforced server-side; client-provided role claims are never trusted without server-side verification |
| **A02 — Cryptographic Failures** | Sensitive data exposed via weak encryption or poor key management | BCrypt strength 12 for passwords; HTTPS enforced in production; JWT payload contains no sensitive PII beyond role; `JWT_SECRET` minimum 256 bits, stored in env var not source code |
| **A03 — Injection** | SQL injection or other injection attacks through user-supplied input | JPA parameterized queries only — no native queries with string concatenation; `@Valid` on all request DTOs with Bean Validation annotations |
| **A04 — Insecure Design** | Business logic flaws baked into the architecture | Business rules enforced in service layer, not controllers; rollback is only permitted on `FAILED` deployments (service-level guard); ADMIN role assignment requires an existing ADMIN to call the role endpoint |
| **A05 — Security Misconfiguration** | Overly permissive defaults left in place | CORS restricted to `CORS_ALLOWED_ORIGINS`; Swagger UI disabled in production profile (or secured behind auth); Spring Security default security headers enabled; no default passwords |
| **A06 — Vulnerable and Outdated Components** | Using outdated library versions with known CVEs | Dependency versions pinned in `build.gradle.kts`; OWASP Dependency-Check Gradle plugin runs in CI (`./gradlew dependencyCheckAnalyze`); Dependabot enabled on GitHub repository for automated PR alerts on vulnerable dependencies |
| **A07 — Identification and Authentication Failures** | Weak authentication mechanisms or poor session management | JWT expiry enforced on every request; BCrypt for password hashing; unique constraint on `username` and `email` columns; short-lived access tokens (24h) + refresh token pattern |
| **A08 — Software and Data Integrity Failures** | Tampered dependencies or migration scripts | Flyway migrations are immutable — once applied, they must never be modified (checksum validation fails on tamper); Docker images reference specific version tags, not `latest` |
| **A09 — Security Logging and Monitoring Failures** | Security events not logged, preventing incident detection | All authentication events logged: `LOGIN_SUCCESS`, `LOGIN_FAILURE`; all role changes audited via `AuditService`; `AuditService` uses `@Transactional(propagation = REQUIRES_NEW)` to ensure auth events are persisted even if the outer transaction rolls back |
| **A10 — Server-Side Request Forgery (SSRF)** | Application making outbound HTTP requests to attacker-controlled endpoints | No outbound HTTP calls from the application in v1 — there is no SSRF attack surface |

---

## 7. Sensitive Data Handling

### Fields Never Returned in API Responses

| Field | Reason |
|---|---|
| `password` | BCrypt hash must never be exposed; `UserResponse` DTO has no password field |
| Raw JWT tokens | Auth response intentionally returns `accessToken` and `refreshToken` at login — this is expected. All other endpoints never echo back tokens. |

### Fields Masked in Logs

| Field | Log Representation |
|---|---|
| Email address | `em***@domain.com` pattern (prefix truncated, domain preserved) |
| Password | Never logged under any circumstances |
| JWT token | Never logged; only the username extracted from the token is logged |

### MDC (Mapped Diagnostic Context) Fields

Fields that ARE included in MDC for structured logging:

| MDC Key | Content |
|---|---|
| `requestId` | UUID generated per request by request-scoped filter |
| `username` | Authenticated username from `SecurityContext` |
| `endpoint` | HTTP method + path (e.g., `POST /api/deployments`) |

Fields that are NEVER placed in MDC: `password`, `token`, `email_full`.

### Database Storage

- Passwords stored as BCrypt hashes only (60-character hash string).
- No plaintext passwords exist anywhere in the system — not in application logs, not in database dumps, not in config files.

---

## 8. Security Headers

Spring Security provides the following headers by default on every response:

| Header | Value | Purpose |
|---|---|---|
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing by the browser |
| `X-Frame-Options` | `DENY` | Prevents the application from being embedded in iframes (clickjacking protection) |
| `Cache-Control` | `no-cache, no-store, max-age=0, must-revalidate` | Prevents sensitive API responses from being cached by browsers or proxies |
| `X-XSS-Protection` | `0` | Browser XSS auditors are deprecated and can introduce vulnerabilities; set to 0 to disable. XSS protection is handled by CSP. |
| `Content-Security-Policy` | Custom (applied to Swagger UI pages) | Restricts sources for scripts and styles on Swagger UI to prevent XSS on documentation pages |

These headers are active because the default Spring Security header configuration is not disabled in the `SecurityFilterChain`. If you ever call `http.headers(headers -> headers.disable())`, these protections are removed — do not do this.
