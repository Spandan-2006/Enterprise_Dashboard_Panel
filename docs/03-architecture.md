# Architecture Document

**Project:** Enterprise Deployment Dashboard
**Version:** 1.0
**Date:** 2026-07-13
**Status:** Approved

---

## Table of Contents

1. [Architecture Style](#1-architecture-style)
2. [Component Diagram](#2-component-diagram)
3. [Request Lifecycle](#3-request-lifecycle)
4. [Cross-Cutting Concerns](#4-cross-cutting-concerns)
5. [Technology Decisions](#5-technology-decisions)
6. [Configuration Management](#6-configuration-management)
7. [Scalability Considerations (v2 Roadmap)](#7-scalability-considerations-v2-roadmap)

---

## 1. Architecture Style

### Chosen Style: Layered / Clean Architecture

The system is structured as a single deployable Spring Boot application following a strict layered architecture:

```
REST Layer (Controllers + Security)
           │
           ▼
   Service Layer (Business Logic)
           │
           ▼
 Repository Layer (Data Access)
           │
           ▼
   PostgreSQL 15 (Database)
```

Each layer has a single, well-defined responsibility and may only communicate with the layer directly below it. No layer may bypass another — for example, a controller must never call a repository directly.

### Rationale

| Criterion | Justification |
|---|---|
| Team familiarity | Layered architecture is the dominant pattern in the Spring ecosystem; every Spring Boot developer understands `@Controller` → `@Service` → `@Repository` |
| Testability | Each layer can be unit-tested in isolation using Mockito: controllers with `@WebMvcTest`, services with mocked repositories, repositories with `@DataJpaTest` |
| Simplicity | The application is a single-domain problem (deployment lifecycle management); all business logic fits naturally into a small set of services |
| Maintainability | Strict layer separation prevents the codebase from collapsing into a ball of mud as the team grows |
| Onboarding speed | A new engineer can understand the codebase structure within hours, not days |

### Why NOT Microservices for v1

Microservices are explicitly deferred for v1 for the following reasons:

- **Deployment complexity:** Each service requires its own CI/CD pipeline, container image, health check, and network configuration. For a team bootstrapping a product this is wasteful overhead.
- **Service discovery:** A microservices topology requires a service registry (Eureka, Consul) or a service mesh (Istio, Linkerd), neither of which is warranted at this scale.
- **Network latency and partial failure:** Synchronous inter-service calls introduce network hops and distributed failure modes (timeouts, circuit breakers). In a single process these are simple method calls.
- **Single business domain:** All features — authentication, deployments, rollbacks, audit logs, dashboard — share the same core entities (`User`, `Deployment`, `AuditLog`). There is no natural seam along which to split them into independent services without artificial data duplication.
- **Premature optimization:** Microservices solve scaling and team-ownership problems that do not yet exist. Introducing them now would be premature optimization at the cost of real development velocity.

### Scalability Path to v2

The architecture is deliberately designed so that horizontal scaling in v2 requires only infrastructure changes, not application refactoring:

- **Stateless JWT authentication** means every application instance can authenticate any request independently — no shared session state is needed.
- **Multiple identical instances** can be placed behind a load balancer (AWS ALB, NGINX, Kubernetes Ingress) without sticky sessions.
- **HikariCP connection pooling** is configured per-instance and prevents DB connection exhaustion under load.
- Should true microservice decomposition become warranted (distinct teams owning distinct bounded contexts, independent scaling requirements), the existing service boundaries map cleanly to candidate services: `AuthService`, `DeploymentService`, `AuditService`.

---

## 2. Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Spring Boot Application                          │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │  REST Controllers│  │ Spring Security  │  │   Swagger UI     │   │
│  │                 │  │                 │  │   (OpenAPI 3.0)  │   │
│  │ AuthController  │  │ SecurityConfig  │  │                  │   │
│  │ DeploymentCtrl  │  │ JwtAuthFilter   │  └──────────────────┘   │
│  │ DashboardCtrl   │  │ UserDetailsImpl │                          │
│  │ UserController  │  └────────┬────────┘                          │
│  │ AuditController │           │ (filters every request)           │
│  └────────┬────────┘           │                                   │
│           └────────────────────┘                                   │
│                       ↓                                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Service Layer                              │  │
│  │   AuthService  │  DeploymentService  │  AuditService         │  │
│  │   UserService  │  DashboardService   │  RollbackService      │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           ↓                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  Repository Layer                             │  │
│  │  UserRepository  │  DeploymentRepository  │  RollbackRepo    │  │
│  │                  │  AuditLogRepository    │                  │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
└───────────────────────────│─────────────────────────────────────────┘
                            │ JPA / Hibernate
                            ▼
                 ┌──────────────────────┐
                 │   PostgreSQL 15      │
                 │   (Port 5432)        │
                 └──────────────────────┘
```

### Component Descriptions

| Component | Type | Responsibility |
|---|---|---|
| `AuthController` | `@RestController` | Handles `/api/auth` endpoints: register, login, token refresh |
| `DeploymentController` | `@RestController` | Full CRUD for deployments, status updates, rollback trigger |
| `DashboardController` | `@RestController` | Serves aggregated metrics and statistics |
| `UserController` | `@RestController` | Admin-only user management: list, update role, delete |
| `AuditController` | `@RestController` | Admin-only audit log queries with filtering and pagination |
| `SecurityConfig` | `@Configuration` | Defines `SecurityFilterChain`: permit list, JWT filter, CSRF policy, CORS |
| `JwtAuthFilter` | `OncePerRequestFilter` | Extracts and validates JWT on every request, populates `SecurityContext` |
| `CustomUserDetailsService` | `UserDetailsService` | Loads `UserDetails` from the database for Spring Security |
| `AuthService` | `@Service` | Registration, login, token generation and refresh logic |
| `DeploymentService` | `@Service` | Deployment CRUD, status machine transitions, rollback orchestration |
| `DashboardService` | `@Service` | Aggregation queries: total, success rate, environment breakdown |
| `UserService` | `@Service` | User retrieval, role updates, soft or hard delete |
| `AuditService` | `@Service` | Writes `AuditLog` entries; queried by `AuditController` |
| `RollbackService` | `@Service` | Creates `Rollback` record, updates deployment status to `ROLLED_BACK` |
| `UserRepository` | `JpaRepository` | CRUD + `findByUsername`, `findByEmail` |
| `DeploymentRepository` | `JpaRepository` | CRUD + paged queries + projection queries for dashboard |
| `AuditLogRepository` | `JpaRepository` | Insert + paged queries with filtering by username, action, date range |
| `RollbackRepository` | `JpaRepository` | Insert + `findByDeploymentId` for rollback history |
| `JwtUtil` | `@Component` | Token generation, signature validation, claims extraction |
| `GlobalExceptionHandler` | `@ControllerAdvice` | Catches all domain and framework exceptions, returns `ApiResponse` errors |
| `ApiResponse<T>` | POJO | Universal response wrapper: `success`, `message`, `data`, `timestamp` |

---

## 3. Request Lifecycle

The following is a step-by-step trace of a typical authenticated request — for example, `GET /api/deployments` with a valid JWT.

```
Step  Component                         Action
────  ────────────────────────────────  ──────────────────────────────────────────────────
  1   Network                           HTTP Request arrives at port 8080
  2   SecurityFilterChain               Passes request to JwtAuthenticationFilter
  3   JwtAuthenticationFilter           Extracts Authorization: Bearer <token> header
  4   JwtUtil.validateToken()           Verifies signature (HMAC-SHA256), expiry, and claims
  5   CustomUserDetailsService          loadUserByUsername() loads User entity from DB
  6   SecurityContextHolder             UsernamePasswordAuthenticationToken stored in context
  7   @RestController                   Request reaches controller; @PreAuthorize checks role
  8   Controller → @Service             Controller delegates; contains zero business logic
  9   @Service                          Business logic executes; calls @Repository for DB ops
 10   @Repository                       JPA query executed via Hibernate against PostgreSQL
 11   @Service                          Builds response DTO from entity
 12   @RestController                   Returns ResponseEntity<ApiResponse<T>> with HTTP status
 13   GlobalExceptionHandler            If any step throws an exception, @ControllerAdvice
                                        intercepts it and maps to a standard ApiResponse error
```

**JWT validation failure path:** If step 4 fails (expired token, invalid signature, missing header), `JwtAuthenticationFilter` immediately returns `HTTP 401 Unauthorized` without invoking any controller — no business logic executes.

**Authorization failure path:** If step 7 fails (authenticated user lacks the required role), Spring Security throws `AccessDeniedException`, which `GlobalExceptionHandler` maps to `HTTP 403 Forbidden`.

---

## 4. Cross-Cutting Concerns

### 4.1 Security

Security is centralized and enforced at two levels:

| Level | Mechanism | Configuration |
|---|---|---|
| Transport | HTTPS (TLS) | Configured at load balancer / reverse proxy level |
| Authentication | JWT via `JwtAuthenticationFilter` | Applied globally via `SecurityFilterChain`; every request except `/api/auth/**` requires a valid token |
| Authorization | `@PreAuthorize("hasRole('ADMIN')")` | Applied at method level on `@Service` and `@RestController` methods requiring elevated privilege |
| Password storage | BCrypt (strength 12) | `PasswordEncoder` bean in `SecurityConfig` |
| CORS | Configured in `SecurityConfig` | Restricts origins in production; permissive in dev profile |
| CSRF | Disabled | Stateless JWT architecture; no browser-session-based CSRF risk |

Role-to-permission matrix:

| Endpoint Category | ADMIN | DEVOPS_ENGINEER | DEVELOPER |
|---|:---:|:---:|:---:|
| Register / Login | Yes | Yes | Yes |
| View deployments | Yes | Yes | Yes |
| Create / update deployment | Yes | Yes | No |
| Update deployment status | Yes | Yes | No |
| Trigger rollback | Yes | Yes | No |
| View dashboard | Yes | Yes | Yes |
| Manage users | Yes | No | No |
| View audit logs | Yes | No | No |

### 4.2 Audit Logging

Every mutating operation emits an audit event. Audit entries are written by `AuditService.log()` and share the same `@Transactional` boundary as the originating operation — if the business operation fails and rolls back, the audit entry rolls back too, preventing phantom audit records.

Events captured:

| Originating Service | Events |
|---|---|
| `AuthService` | `USER_REGISTERED`, `USER_LOGIN`, `USER_LOGOUT`, `TOKEN_REFRESHED` |
| `DeploymentService` | `DEPLOYMENT_CREATED`, `DEPLOYMENT_UPDATED`, `DEPLOYMENT_STATUS_CHANGED`, `DEPLOYMENT_DELETED` |
| `RollbackService` | `ROLLBACK_TRIGGERED`, `ROLLBACK_COMPLETED` |
| `UserService` | `USER_ROLE_UPDATED`, `USER_DELETED` |

### 4.3 Exception Handling

`GlobalExceptionHandler` (`@ControllerAdvice`) is the single point for translating exceptions to HTTP responses. This keeps controllers and services completely free of HTTP status code logic.

| Exception Class | HTTP Status | Scenario |
|---|---|---|
| `ResourceNotFoundException` | 404 Not Found | Entity not found by ID |
| `DuplicateResourceException` | 409 Conflict | Username or email already registered |
| `InvalidStatusTransitionException` | 400 Bad Request | Illegal deployment status transition |
| `UnauthorizedOperationException` | 403 Forbidden | Business-level permission check failed |
| `AccessDeniedException` (Spring) | 403 Forbidden | `@PreAuthorize` role check failed |
| `AuthenticationException` (Spring) | 401 Unauthorized | Invalid or missing JWT |
| `MethodArgumentNotValidException` | 400 Bad Request | Bean Validation failure on request DTO |
| `Exception` (fallback) | 500 Internal Server Error | Unexpected runtime error |

All error responses use the standard `ApiResponse` envelope:

```json
{
  "success": false,
  "message": "Deployment with ID 42 not found",
  "data": null,
  "timestamp": "2026-07-13T10:30:00Z"
}
```

### 4.4 Validation

Jakarta Bean Validation is applied to all inbound request DTOs before the request reaches the service layer.

| Annotation | Usage |
|---|---|
| `@Valid` | On `@RequestBody` parameters in controllers |
| `@NotBlank` | Required string fields (username, projectName, version) |
| `@NotNull` | Required object fields (environment, status enums) |
| `@Email` | Email format enforcement on registration |
| `@Size` | Min/max length constraints (passwords, names) |

Validation failures throw `MethodArgumentNotValidException`, handled by `GlobalExceptionHandler` into a structured `400` response listing all violated constraints.

---

## 5. Technology Decisions

Full rationale is captured in the Architecture Decision Records:

| ADR | Decision | Summary |
|---|---|---|
| ADR-0001 | PostgreSQL 15 over MySQL / MongoDB | See `docs/ADR-0001` — relational model with ACID guarantees fits deployment tracking; native JSON support available for extensions |
| ADR-0002 | JWT over session cookies / OAuth2 for v1 | See `docs/ADR-0002` — stateless tokens enable horizontal scaling without shared session store; OAuth2 added complexity deferred to v2 |
| ADR-0003 | Gradle over Maven | See `docs/ADR-0003` — faster incremental builds, Kotlin DSL for type-safe build scripts, better fit for Java 21 toolchains |

Additional technology selections:

| Technology | Version | Justification |
|---|---|---|
| Java | 21 (LTS) | Latest LTS; virtual threads (Project Loom) available for high-concurrency v2 scenarios |
| Spring Boot | 3.x | First-class Java 21 support; native `SecurityFilterChain` API; auto-configured HikariCP |
| Spring Security | 6.x | Included with Spring Boot 3; `SecurityFilterChain` bean replaces deprecated `WebSecurityConfigurerAdapter` |
| Spring Data JPA | 3.x | Reduces repository boilerplate; supports Specification API for dynamic queries |
| Hibernate | 6.x | JPA provider; supports `@BatchSize` and second-level cache for future optimization |
| HikariCP | bundled | Fastest JDBC connection pool; configured via `spring.datasource.hikari.*` |
| jjwt (JJWT) | 0.12.x | De-facto standard JWT library for Java; supports HMAC-SHA256 and RS256 |
| Springdoc OpenAPI | 2.x | Generates interactive Swagger UI from `@Operation` annotations; zero XML config |
| JUnit 5 + Mockito | latest | Standard Spring Boot testing stack; `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` |

---

## 6. Configuration Management

### Spring Profiles

The application ships with three profiles, activated via the `SPRING_PROFILES_ACTIVE` environment variable:

| Profile | Purpose | Database URL |
|---|---|---|
| `dev` | Local development; verbose logging; H2 or local PostgreSQL | `localhost:5432/dashboard_dev` |
| `staging` | Pre-production validation; mirrors prod config | Injected via environment variable |
| `prod` | Production; minimal logging; strict CORS; SSL enforced | Injected via environment variable |

Profile-specific property files:

```
src/main/resources/
  application.yml             # Shared defaults
  application-dev.yml         # Dev overrides
  application-staging.yml     # Staging overrides
  application-prod.yml        # Prod overrides (no secrets)
```

### Secrets Management

All sensitive values are injected via environment variables at runtime. No secrets are committed to version control.

| Secret | Environment Variable | Used In |
|---|---|---|
| Database URL | `DB_URL` | `spring.datasource.url` |
| Database username | `DB_USERNAME` | `spring.datasource.username` |
| Database password | `DB_PASSWORD` | `spring.datasource.password` |
| JWT signing key | `JWT_SECRET` | `JwtUtil` |
| JWT expiry (access) | `JWT_EXPIRY_MS` | `JwtUtil` |
| JWT expiry (refresh) | `JWT_REFRESH_EXPIRY_MS` | `JwtUtil` |

Full environment variable documentation: `docs/18-environments.md`

### application.yml Structure

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
      ddl-auto: validate          # Never create/drop in prod
    show-sql: false               # Enabled only in dev profile
    properties:
      hibernate.format_sql: true

jwt:
  secret: ${JWT_SECRET}
  expiry-ms: ${JWT_EXPIRY_MS:900000}        # 15 minutes
  refresh-expiry-ms: ${JWT_REFRESH_EXPIRY_MS:604800000}  # 7 days

server:
  port: 8080
```

---

## 7. Scalability Considerations (v2 Roadmap)

The v1 design is deliberately stateless, making horizontal scaling a pure infrastructure concern in v2.

### Horizontal Scaling

```
                    ┌───────────────────┐
                    │   Load Balancer   │
                    │  (NGINX / ALB)    │
                    └─────────┬─────────┘
                              │
               ┌──────────────┼──────────────┐
               │              │              │
        ┌──────┴──────┐ ┌─────┴──────┐ ┌────┴───────┐
        │  Instance 1 │ │ Instance 2 │ │ Instance N │
        │ Spring Boot │ │ Spring Boot│ │ Spring Boot│
        └──────┬──────┘ └─────┬──────┘ └────┬───────┘
               └──────────────┼──────────────┘
                              │
                    ┌─────────┴─────────┐
                    │   PostgreSQL 15   │
                    │  (Primary + Read  │
                    │     Replicas)     │
                    └───────────────────┘
```

**Why this works without redesign:** JWT tokens are self-contained — the signing secret is shared via the `JWT_SECRET` environment variable, so any instance can validate any token. No session affinity (sticky sessions) is required at the load balancer.

### Connection Pool Sizing

HikariCP is configured with `maximum-pool-size: 20` per instance. With N instances, the total connection count to PostgreSQL is `N * 20`. For PostgreSQL's default `max_connections: 100`, deploy no more than 5 instances without tuning `max_connections` or introducing a connection pooler (PgBouncer).

### v2 Planned Enhancements

| Enhancement | Problem Solved | Technology |
|---|---|---|
| Redis JWT blacklist | Token revocation (logout invalidates token immediately) | Spring Data Redis; check blacklist in `JwtAuthFilter` |
| Redis dashboard cache | Repeated aggregation queries against large deployment tables | `@Cacheable` on `DashboardService` methods; TTL 60 seconds |
| Async notifications | Deployment status changes trigger email/Slack without blocking the request | Spring Events or Kafka; `DeploymentService` publishes event, `NotificationService` consumes |
| Read replicas | Dashboard and audit queries are read-heavy; offload to replicas | Spring Data JPA routing datasource; `@Transactional(readOnly=true)` |
| Database migrations | Controlled schema evolution in production | Flyway (already planned); `spring.flyway.enabled=true` |
| Kubernetes deployment | Orchestrate N instances with health checks and rolling updates | Kubernetes `Deployment` resource; liveness and readiness probes on `/actuator/health` |
