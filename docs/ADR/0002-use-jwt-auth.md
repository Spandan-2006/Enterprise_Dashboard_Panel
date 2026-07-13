# ADR-0002: Use JWT for API Authentication

## Status
Accepted

## Context
The REST API must authenticate callers across multiple endpoints. Requirements:
- Stateless authentication to support future horizontal scaling (multiple Spring Boot instances)
- Role information must be available without a DB lookup on every request
- No external identity provider dependency in v1
- Compatible with both browser (React frontend) and API client (Swagger, curl, Postman) consumers

## Decision
JWT (JSON Web Tokens) with JJWT library (`io.jsonwebtoken:jjwt-api:0.12.x`), using an access token (24h) + refresh token (7d) pattern. Passwords stored with BCrypt (strength 12).

## Rationale
- Stateless: no server-side session store needed — each JWT is self-contained with claims (username, role, expiry)
- Role carried in token: `@PreAuthorize` can evaluate role from the token claims without a DB lookup on each request
- Widely understood standard (RFC 7519) with mature Java library support
- Works with both browser clients (Authorization header) and CLI tools
- BCrypt for password hashing is the Spring Security default and industry standard for password storage

## Consequences

### Positive
- No sticky session requirement — instances are interchangeable behind a load balancer
- Simple implementation with Spring Security + custom `JwtAuthenticationFilter`
- Refresh token pattern allows short-lived access tokens without constant re-login

### Negative / Trade-offs
- Token revocation is not possible without a server-side blacklist (Redis, planned for v2) — a stolen token is valid until expiry
- JWT secret key must be rotated carefully (requires re-login for all users)
- Token size is larger than a session ID (~200-400 bytes per request in Authorization header)
- `localStorage` token storage (frontend) is vulnerable to XSS — httpOnly cookie storage is more secure (v2 improvement)

## Alternatives Considered

| Alternative | Pros | Cons | Reason Rejected |
|---|---|---|---|
| Session cookies (Spring Security default) | Simple implementation, httpOnly cookie prevents XSS token theft | Stateful — requires sticky sessions or shared session store (Redis); CSRF protection needed | Violates stateless API design requirement |
| OAuth2 / OpenID Connect (Keycloak) | Industry-standard, delegated auth, supports SSO | Requires external Keycloak deployment; operational complexity exceeds v1 scope | Over-engineered for v1; planned for enterprise v2 |
| API Keys | Simple, no expiry complexity | No built-in expiry/revocation, harder to include role claims, less secure for interactive use | JWT is more flexible and secure for human + API client consumers |
| Basic Auth (HTTP) | Zero implementation complexity | Password sent on every request (even with HTTPS); no role claim support; not suitable for a browser SPA | Fundamentally insecure pattern for a production application |
