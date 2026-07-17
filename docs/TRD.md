# Technical Requirements Document (TRD)

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Status:** Approved

---

## 1. Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Runtime | Java | 21 (LTS) | Latest LTS; virtual threads available for v2 high-concurrency scenarios |
| Framework | Spring Boot | 3.x | First-class Java 21 support; auto-configured HikariCP; native Security 6 |
| Security | Spring Security | 6.x | `SecurityFilterChain` bean replaces deprecated `WebSecurityConfigurerAdapter` |
| Persistence | Spring Data JPA | 3.x | Reduces repository boilerplate; supports Specification API for dynamic queries |
| ORM | Hibernate | 6.x | JPA provider; `@BatchSize` and second-level cache ready for v2 |
| Connection Pool | HikariCP | bundled | Fastest JDBC pool; configured via `spring.datasource.hikari.*` |
| Database | PostgreSQL | 15+ | Full ACID transactions; `BIGSERIAL`, `CHECK` constraints, JSONB for future use |
| JWT | jjwt (JJWT) | 0.12.x | De-facto standard JWT library for Java; HMAC-SHA256 signing |
| Schema Migrations | Flyway | bundled with Boot | Immutable versioned SQL migrations; checksum-validated |
| API Docs | Springdoc OpenAPI | 2.x | Generates Swagger UI from `@Operation` annotations; zero XML config |
| Build | Gradle | 8.x | Kotlin DSL; faster incremental builds than Maven |
| Testing | JUnit 5 + Mockito 5 | latest | `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` |
| Frontend (Optional) | React 18 + Vite 5 | 18.x / 5.x | Component model with hooks; fast HMR dev server |
| Frontend Styling | Tailwind CSS | 3.x | Utility-first; no custom CSS files |
| Frontend HTTP | Axios | 1.x | Request/response interceptors for auth and centralized error handling |
| Frontend Charts | Chart.js + react-chartjs-2 | 4.x / 5.x | Dashboard doughnut and line charts |
| Containerization | Docker + Docker Compose | 24+ / v2 | Multi-stage image build; `postgres:15-alpine` |
| CI/CD | GitHub Actions | — | Build, test, coverage enforcement, Docker image push |

---

## 2. Architecture

### Style: Layered / Clean Architecture

```
REST Layer (Controllers + Spring Security FilterChain)
                    │
                    ▼
         Service Layer (Business Logic)
                    │
                    ▼
       Repository Layer (Spring Data JPA)
                    │
                    ▼
           PostgreSQL 15 Database
```

Each layer communicates only with the layer directly below it. A controller must never call a repository. A service must never return an entity to a controller — it maps to a DTO first.

### Component Summary

| Component | Type | Responsibility |
|-----------|------|----------------|
| `AuthController` | `@RestController` | `/api/auth` — register, login, token refresh |
| `DeploymentController` | `@RestController` | Full CRUD for deployments, status updates, rollback trigger |
| `DashboardController` | `@RestController` | Aggregated metrics and recent activity |
| `UserController` | `@RestController` | ADMIN-only user management |
| `AuditController` | `@RestController` | ADMIN-only audit log queries |
| `SecurityConfig` | `@Configuration` | `SecurityFilterChain`: permit list, JWT filter, CSRF policy, CORS |
| `JwtAuthFilter` | `OncePerRequestFilter` | Extracts and validates JWT on every request; populates `SecurityContext` |
| `CustomUserDetailsService` | `UserDetailsService` | Loads `UserDetails` from database for Spring Security |
| `AuthService` | `@Service` | Registration, login, token generation, refresh |
| `DeploymentService` | `@Service` | Deployment CRUD, state machine transitions, rollback orchestration |
| `DashboardService` | `@Service` | Aggregation queries: counts, success rate, recent list |
| `UserService` | `@Service` | User retrieval, role updates, delete |
| `AuditService` | `@Service` | Writes `AuditLog` entries; queried by `AuditController` |
| `JwtUtil` | `@Component` | Token generation, signature validation, claims extraction |
| `GlobalExceptionHandler` | `@ControllerAdvice` | Maps all domain/framework exceptions to `ApiResponse` errors |

### Request Lifecycle

```
1  HTTP request arrives at :8080
2  SecurityFilterChain → JwtAuthenticationFilter
3  Extract "Authorization: Bearer <token>" header
4  JwtUtil.validateToken() — verifies signature (HS256), expiry, claims
5  CustomUserDetailsService.loadUserByUsername() — loads User from DB
6  UsernamePasswordAuthenticationToken stored in SecurityContextHolder
7  Request reaches controller; @PreAuthorize checks role from SecurityContext
8  Controller delegates to @Service (zero business logic in controller)
9  @Service executes business logic; calls @Repository for DB operations
10 @Repository executes JPA query via Hibernate against PostgreSQL
11 @Service maps entity → DTO; returns to controller
12 Controller returns ResponseEntity<ApiResponse<T>>
13 On any exception: GlobalExceptionHandler intercepts → structured ApiResponse error
```

---

## 3. Security Design

### Authentication — JWT Stateless

| Property | Value |
|----------|-------|
| Algorithm | HS256 (HMAC-SHA256) |
| Secret | Env var `JWT_SECRET`; minimum 256 bits / 32 bytes Base64-encoded |
| Access token expiry | 24 hours (configurable via `JWT_EXPIRY_MS`) |
| Refresh token expiry | 7 days (configurable via `JWT_REFRESH_EXPIRY_MS`) |
| Claims | `sub` (username), `role`, `iat`, `exp` |
| Session policy | `STATELESS` — no HTTP session created |

### JWT Token Claims

**Access Token:**

| Claim | Value |
|-------|-------|
| `sub` | username (used as Spring Security principal) |
| `role` | e.g. `"ADMIN"` — single role per user |
| `iat` | Unix epoch seconds, issued-at |
| `exp` | Unix epoch seconds, 24h after `iat` |

**JwtUtil API:**

| Method | Description |
|--------|-------------|
| `generateAccessToken(UserDetails)` | Creates signed JWT with 24h expiry |
| `generateRefreshToken(UserDetails)` | Same structure, 7d expiry |
| `extractUsername(token)` | Extracts `sub` claim |
| `isTokenValid(token, UserDetails)` | Validates signature, expiry, and subject match |

### Password Security

- BCrypt strength factor 12 (~300 ms per hash; computationally expensive for brute-force)
- Passwords **never** logged, returned in API responses, or stored as plaintext
- `UserResponse` DTO has no `password` field

### Authorization Matrix

`Y` = permitted, `N` = denied, `(open)` = no authentication required.

| Endpoint | ADMIN | DEVOPS_ENGINEER | DEVELOPER |
|----------|-------|-----------------|-----------|
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

### CORS Configuration

| Property | Value |
|----------|-------|
| `allowedOrigins` | Read from env var `CORS_ALLOWED_ORIGINS` (comma-separated); defaults to `http://localhost:3000` in dev |
| `allowedMethods` | GET, POST, PUT, PATCH, DELETE, OPTIONS |
| `allowedHeaders` | `Authorization`, `Content-Type` |
| `allowCredentials` | `false` (Bearer token auth, not cookie-based) |

### OWASP Top 10 Mitigations

| Category | Mitigation |
|----------|-----------|
| A01 Broken Access Control | `@PreAuthorize` on all write endpoints; server-side role checks; client-provided roles not trusted |
| A02 Cryptographic Failures | BCrypt-12 for passwords; HTTPS in production; JWT secret minimum 256 bits in env var |
| A03 Injection | JPA parameterized queries only; `@Valid` on all request DTOs |
| A04 Insecure Design | Business rules in service layer; rollback only permitted on `FAILED` status |
| A05 Security Misconfiguration | CORS restricted to `CORS_ALLOWED_ORIGINS`; Swagger UI disabled in production profile |
| A07 Auth Failures | JWT expiry enforced per request; BCrypt hashing; unique username/email constraints |
| A08 Data Integrity Failures | Flyway migrations immutable (checksum-validated); Docker images pinned to version tags |
| A09 Security Logging | All auth events logged: `LOGIN_SUCCESS`, `LOGIN_FAILURE`; all role changes audited |

### Security Headers (Spring Security defaults)

| Header | Value |
|--------|-------|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Cache-Control` | `no-cache, no-store, max-age=0, must-revalidate` |

---

## 4. Exception Handling

All exceptions are mapped by `GlobalExceptionHandler` (`@RestControllerAdvice`). Every response uses the `ApiResponse<T>` envelope.

### Exception Hierarchy

```
RuntimeException
└── AppException (abstract; HttpStatus + message)
    ├── ResourceNotFoundException       HTTP 404
    ├── UnauthorizedException           HTTP 401
    ├── ForbiddenException              HTTP 403
    ├── ConflictException               HTTP 409
    ├── ValidationException             HTTP 400
    └── InvalidStatusTransitionException HTTP 422
```

### Handler Mapping

| Exception | HTTP Status | Message Pattern |
|-----------|-------------|-----------------|
| `MethodArgumentNotValidException` | 400 | `"Validation failed"` + `errors[]` array with `{field, message}` |
| `ValidationException` | 400 | `exception.getMessage()` |
| `InvalidStatusTransitionException` | 422 | `"Cannot transition from {from} to {to}"` |
| `ResourceNotFoundException` | 404 | `"Resource not found: {type} with id {id}"` |
| `UnauthorizedException` | 401 | `"Authentication required or credentials invalid"` |
| `AccessDeniedException` (Spring) | 403 | `"Insufficient permissions for this operation"` |
| `ConflictException` | 409 | `exception.getMessage()` |
| `DataIntegrityViolationException` | 409 | `"Resource already exists or constraint violation"` |
| `Exception` (catch-all) | 500 | `"An unexpected error occurred. Reference: {UUID}"` |

---

## 5. Configuration & Environments

### Spring Profiles

| Profile | Purpose | `ddl-auto` | Flyway |
|---------|---------|-----------|--------|
| `dev` | Local development; verbose logging; local PostgreSQL via Docker Compose | `update` | optional |
| `test` | Unit/integration tests; H2 in-memory | `create-drop` | disabled |
| `staging` | Pre-production validation; mirrors prod config | `validate` | enabled |
| `prod` | Production; minimal logging; strict CORS | `validate` | enabled |

### Required Environment Variables

| Variable | Purpose | Default (dev only) |
|----------|---------|-------------------|
| `DB_URL` | JDBC connection URL | `jdbc:postgresql://localhost:5432/dashboard` |
| `DB_USERNAME` | Database username | `dashboard` |
| `DB_PASSWORD` | Database password | `dashboard` |
| `JWT_SECRET` | HS256 signing secret (min 256 bits, Base64-encoded) | *(must be set explicitly)* |
| `JWT_EXPIRY_MS` | Access token TTL in milliseconds | `86400000` (24h) |
| `JWT_REFRESH_EXPIRY_MS` | Refresh token TTL in milliseconds | `604800000` (7d) |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed origins | `http://localhost:3000` |
| `SPRING_PROFILES_ACTIVE` | Active Spring profile | `dev` |

### application.yml Skeleton

```yaml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: false

jwt:
  secret: ${JWT_SECRET}
  expiry-ms: ${JWT_EXPIRY_MS:86400000}
  refresh-expiry-ms: ${JWT_REFRESH_EXPIRY_MS:604800000}

server:
  port: 8080
```

---

## 6. Testing Strategy

### Pyramid

| Layer | Share | Tools | Runs in CI |
|-------|-------|-------|-----------|
| Unit | ~60% | JUnit 5, Mockito 5 (`@ExtendWith(MockitoExtension.class)`) | Yes |
| Integration | ~35% | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, H2 | Yes |
| E2E | ~5% | Manual / Cypress (planned v2) | No |

### Coverage Targets (JaCoCo)

| Module | Target |
|--------|--------|
| AuthService | 90% |
| DeploymentService | 85% |
| AuditService | 80% |
| UserService | 85% |
| DashboardService | 80% |
| All Controllers | 85% |
| **Overall line coverage** | **80% minimum** |

The build fails if line coverage drops below 80% (`jacocoTestCoverageVerification` wired into `check` task).

### Test Naming Convention

```
methodName_givenCondition_thenExpectedResult()
```

### Security Test Checklist (per endpoint)

1. No token → 401
2. Valid token, wrong role → 403
3. Valid token, correct role → 2xx
4. Expired token → 401

### Test application-test.yml

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  flyway:
    enabled: false
jwt:
  secret: dGVzdC1zZWNyZXQta2V5LWZvci11bml0LXRlc3RpbmctMzItYnl0ZXM=
  expiry-ms: 86400000
  refresh-expiry-ms: 604800000
```

---

## 7. Performance Targets

| Metric | Target |
|--------|--------|
| p95 API response time | < 200 ms under 100 concurrent users |
| Dashboard page load | < 2 seconds (including frontend rendering) |
| HikariCP pool size | `maximum-pool-size: 20` per instance |
| Max horizontal instances | 5 (before PostgreSQL `max_connections: 100` is saturated without PgBouncer) |

---

## 8. Architecture Decision Records

| ADR | Decision | Reason |
|-----|----------|--------|
| ADR-0001 | PostgreSQL 15 over MySQL/MongoDB/H2 | Full ACID transactions; BIGSERIAL PKs; CHECK constraints for enum columns; JSONB available for v2 |
| ADR-0002 | JWT over session cookies / OAuth2 | Stateless tokens enable horizontal scaling without shared session store; OAuth2 complexity deferred to v2 |
| ADR-0003 | Gradle with Kotlin DSL over Maven | Faster incremental builds; type-safe build scripts; better Java 21 toolchain support |

---

## 9. v2 Roadmap

| Enhancement | Problem Solved | Technology |
|-------------|---------------|-----------|
| Redis JWT blacklist | Token revocation on logout | Spring Data Redis; check blacklist in `JwtAuthFilter` |
| Redis dashboard cache | Repeated aggregation queries on large tables | `@Cacheable` on `DashboardService`; TTL 60s |
| Async notifications | Status changes trigger email/Slack without blocking request | Spring Events or Kafka |
| Read replicas | Dashboard/audit queries are read-heavy | Spring Data JPA routing datasource; `@Transactional(readOnly=true)` |
| OAuth2 / SSO | Enterprise SSO integration | Spring Security OAuth2 resource server |
| Virtual threads | Higher concurrency at lower footprint | Java 21 Loom; `spring.threads.virtual.enabled=true` |
| Kubernetes | Rolling updates, health probes, N-instance scaling | `Deployment` resource; liveness/readiness on `/actuator/health` |
