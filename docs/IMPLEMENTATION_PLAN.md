# Implementation Plan

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Status:** Approved

This document describes the phased build order, definition of done for each phase, and the coding standards and workflow rules that govern the implementation.

---

## 1. Build Phases

### Phase 0 — Project Scaffold

**Goal:** A runnable empty Spring Boot application connected to PostgreSQL.

**Tasks:**
1. Initialize Gradle project with Kotlin DSL (`build.gradle.kts`)
2. Add dependencies: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-security`, `spring-boot-starter-validation`, `postgresql`, `flyway-core`, `jjwt-api/impl/jackson`, `springdoc-openapi-starter-webmvc-ui`, `lombok` (optional)
3. Create `application.yml` with Spring profiles (`dev`, `test`, `staging`, `prod`) and HikariCP config
4. Create `application-test.yml` with H2 in-memory DB, Flyway disabled, `ddl-auto: create-drop`
5. Create Docker Compose file: `spring-boot-app` + `postgres:15-alpine` services
6. Verify: `docker-compose up --build` → `curl localhost:8080/actuator/health` returns `{"status":"UP"}`

**Definition of Done:**
- `./gradlew build` succeeds
- `GET /actuator/health` returns 200
- Swagger UI accessible at `http://localhost:8080/swagger-ui/index.html`

---

### Phase 1 — Database Schema

**Goal:** All four tables created and verified in PostgreSQL via Flyway.

**Tasks:**
1. Write Flyway migrations in `src/main/resources/db/migration/`:
   - `V1__create_users_table.sql`
   - `V2__create_deployments_table.sql`
   - `V3__create_rollbacks_table.sql`
   - `V4__create_audit_logs_table.sql`
   - `V5__create_indexes.sql`
   - `V6__seed_initial_data.sql`
2. Create JPA entity classes: `User`, `Deployment`, `Rollback`, `AuditLog`
3. Create enum classes: `Role`, `Environment`, `DeploymentStatus`
4. Configure `ddl-auto: validate` in production profile; `create-drop` in test profile
5. Verify schema with `psql` or a DB client

**Definition of Done:**
- `./gradlew bootRun` applies all migrations without errors
- `\dt` in psql shows all four tables with correct columns
- `@DataJpaTest` for `UserRepository.findByUsername()` passes

---

### Phase 2 — Authentication API

**Goal:** `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh` working end-to-end.

**Tasks:**
1. Implement `JwtUtil` — `generateAccessToken`, `generateRefreshToken`, `extractUsername`, `isTokenValid`
2. Implement `JwtAuthenticationFilter extends OncePerRequestFilter`
3. Implement `CustomUserDetailsService implements UserDetailsService`
4. Configure `SecurityFilterChain` in `SecurityConfig`:
   - `SessionCreationPolicy.STATELESS`
   - Permit: `/api/auth/**`, `/actuator/health`, `/swagger-ui/**`, `/v3/api-docs/**`
   - Authenticate all other paths
   - CORS configuration from `CORS_ALLOWED_ORIGINS` env var
5. Implement DTOs: `RegisterRequest`, `LoginRequest`, `RefreshTokenRequest`, `AuthResponse`
6. Implement `AuthService` / `AuthServiceImpl` — register (BCrypt), login, refresh
7. Implement `AuthController`
8. Implement `AuditService` — `log()` method (use `REQUIRES_NEW` propagation for auth events)
9. Write `GlobalExceptionHandler` — at minimum handles `MethodArgumentNotValidException`, `UnauthorizedException`, catch-all 500

**Tests to write:**
- `AuthServiceTest` — unit tests for register (duplicate username/email), login (valid/invalid credentials), refresh (valid/expired token)
- `AuthControllerIntegrationTest` — POST /register 201, POST /login 200/401, POST /refresh 200/401
- Security: GET /api/deployments without token → 401

**Definition of Done:**
- All auth integration tests pass
- JWT tokens parseable (decode online or via test assertion)
- `LOGIN_SUCCESS` and `LOGIN_FAILURE` records appear in `audit_logs` table after login attempts

---

### Phase 3 — Deployment Lifecycle API

**Goal:** Full CRUD for deployments with state machine enforcement.

**Tasks:**
1. Implement DTOs: `CreateDeploymentRequest`, `UpdateDeploymentRequest`, `UpdateStatusRequest`, `DeploymentResponse`, `DeploymentFilterRequest`
2. Implement `DeploymentService` / `DeploymentServiceImpl`:
   - `createDeployment` — sets `status=PENDING`, `deployedBy=username`, logs `DEPLOYMENT_CREATED`
   - `getDeployments` — paginated; uses `Specification` for dynamic multi-field filtering
   - `getDeploymentById`
   - `updateDeployment` — only updates `remarks`; logs `DEPLOYMENT_UPDATED`
   - `updateStatus` — enforces state machine transitions; sets `endTime` on terminal states; logs `DEPLOYMENT_STATUS_UPDATED`
   - `deleteDeployment` — checks `rollbackRepository.existsByDeploymentId()`; logs `DEPLOYMENT_DELETED`
3. Implement `DeploymentSpecification` for dynamic filter
4. Implement `DeploymentController`:
   - `GET /api/deployments` — `@PreAuthorize("isAuthenticated()")`
   - `POST /api/deployments` — `@PreAuthorize("hasAnyRole('ADMIN','DEVOPS_ENGINEER')")`
   - `GET /api/deployments/{id}`
   - `PUT /api/deployments/{id}` — ADMIN, DEVOPS
   - `DELETE /api/deployments/{id}` — ADMIN only
   - `PATCH /api/deployments/{id}/status` — ADMIN, DEVOPS

**Tests to write:**
- `DeploymentServiceTest` — create, filter, status transitions (valid and invalid), delete (with/without rollbacks)
- `DeploymentControllerIntegrationTest` — security matrix (DEVELOPER tries POST → 403; no token → 401; DEVOPS → 201)
- `DeploymentRepositoryTest` — `findByStatus`, `findByEnvironmentAndStatus`, `countByStatus`

**Definition of Done:**
- All 7 deployment endpoint tests pass
- Illegal status transition (e.g. SUCCESSFUL → PENDING) returns 422
- `audit_logs` table populated after create/update/delete

---

### Phase 4 — Rollback API

**Goal:** Rollback trigger and history retrieval.

**Tasks:**
1. Implement DTOs: `RollbackRequest`, `RollbackResponse`
2. Add `triggerRollback` and `getRollbackHistory` to `DeploymentServiceImpl`:
   - `triggerRollback`: validate `status=FAILED`; create `Rollback` record; call `updateStatus` to `ROLLED_BACK`; log `ROLLBACK_TRIGGERED`; wrap in single `@Transactional`
3. Add controller endpoints:
   - `POST /api/deployments/{id}/rollback` — ADMIN, DEVOPS
   - `GET /api/deployments/{id}/rollbacks` — all roles

**Tests to write:**
- `triggerRollback_givenFailedDeployment_thenCreatesRollbackRecord()`
- `triggerRollback_givenNonFailedDeployment_thenThrows422()`
- Integration test: rollback on SUCCESSFUL deployment → 422

**Definition of Done:**
- POST rollback on FAILED deployment → 201 + deployment status changes to ROLLED_BACK
- DELETE deployment with rollback records → 409

---

### Phase 5 — Dashboard & User Management APIs

**Goal:** Metrics endpoints and admin user management.

**Tasks:**
1. Implement `DashboardService` / `DashboardController`:
   - `GET /api/dashboard/stats` — counts by status + success rate + recent 5
   - `GET /api/dashboard/success-rate`
   - `GET /api/dashboard/recent` — top 10
2. Implement `UserService` / `UserController`:
   - `GET /api/users` — paginated, ADMIN only
   - `GET /api/users/{id}`
   - `PUT /api/users/{id}/role` — logs `USER_ROLE_CHANGED`
   - `DELETE /api/users/{id}` — logs `USER_DELETED`
3. Implement `AuditController`:
   - `GET /api/audit-logs` — paginated, filtered, ADMIN only
   - Implement `AuditLogSpecification`

**Tests to write:**
- `DashboardServiceTest` — success rate formula (zero-division guard, correct percentage)
- `UserControllerIntegrationTest` — DEVOPS_ENGINEER tries GET /users → 403
- `AuditControllerIntegrationTest` — DEVELOPER tries GET /audit-logs → 403

**Definition of Done:**
- All 19 endpoints return correct data and status codes
- Full authorization matrix verified by tests

---

### Phase 6 — Coverage Gate & CI

**Goal:** CI pipeline enforces quality gates.

**Tasks:**
1. Configure JaCoCo in `build.gradle.kts`:
   - Line coverage threshold: 80% minimum
   - Branch coverage threshold: 70% minimum
   - Wire `jacocoTestCoverageVerification` into `check` task
2. Write `security_test.yml` or extend existing test suite to cover every row of the authorization matrix
3. Write test data builders: `UserTestBuilder`, `DeploymentTestBuilder`
4. Create `.github/workflows/ci.yml`:
   - Steps: checkout → setup Java 21 → `./gradlew build test jacocoTestCoverageVerification`
   - Upload test reports as artifact (`actions/upload-artifact@v4`)
5. Verify: push to branch → CI passes with green badge

**Definition of Done:**
- `./gradlew test jacocoTestCoverageVerification` passes locally
- CI pipeline green on GitHub Actions
- Coverage report shows ≥ 80% line coverage overall

---

### Phase 7 — React Frontend (Optional)

**Goal:** Functional React 18 SPA connected to the live backend.

**Tasks:**
1. Scaffold with Vite: `npm create vite@latest frontend -- --template react`
2. Install: `tailwindcss`, `axios`, `react-router-dom`, `chart.js`, `react-chartjs-2`
3. Implement API layer (`src/api/`): `axiosInstance.js`, `authApi.js`, `deploymentApi.js`, `dashboardApi.js`, `userApi.js`, `auditApi.js`
4. Implement `AuthContext` + `useAuth` hook + `PrivateRoute` + `RoleGuard`
5. Build pages in order: Login → Dashboard → Deployments List → Deployment Detail → New Deployment → Rollback → User Management → Audit Logs
6. Implement components: `StatusBadge`, `StatsCard`, `DeploymentStatusDonut`, `Pagination`, `FilterBar`, `ConfirmDialog`, `LoadingSpinner`
7. Wire responsive layout: `Sidebar` (off-canvas on mobile) + `TopBar`

**Definition of Done:**
- All 8 pages render without console errors
- Login/logout flow works with real backend tokens
- DEVELOPER role: create/rollback buttons hidden; direct URL to `/users` → `/forbidden`
- Dashboard charts populate from real API data

---

## 2. Coding Standards

### Java Layer Rules

| Rule | Detail |
|------|--------|
| Controllers only parse/respond | Zero business logic; delegate to `@Service` |
| Services own all logic | No direct repository calls from controllers |
| DTOs at the boundary | Never serialize `@Entity` objects; map to DTOs in service layer |
| `ApiResponse<T>` always | Every controller method returns `ResponseEntity<ApiResponse<T>>` |
| `@Transactional` | All mutating service methods; read-only methods use `@Transactional(readOnly=true)` |
| Enum columns | Always `@Enumerated(EnumType.STRING)` — never `ORDINAL` |
| No plaintext passwords | `passwordEncoder.encode()` before any persist; never log passwords |
| No hardcoded secrets | JWT secret from `JWT_SECRET` env var only |

### Package Structure

```
com.enterprise.dashboard
├── controller/    — @RestController classes
├── service/       — @Service interfaces + impl/ subpackage
├── repository/    — @Repository interfaces
├── mapper/        — entity ↔ DTO mappers
├── model/
│   ├── entity/    — @Entity classes
│   ├── dto/       — request + response DTOs
│   └── enums/     — Role, Environment, DeploymentStatus
├── security/      — JWT filter, UserDetailsService, SecurityConfig
├── config/        — CORS, Swagger, other @Configuration beans
├── exception/     — AppException hierarchy, GlobalExceptionHandler
└── audit/         — AuditLog entity, AuditService, AuditActions constants
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Classes | UpperCamelCase | `DeploymentService`, `AuthController` |
| Methods | lowerCamelCase | `createDeployment`, `triggerRollback` |
| Constants | UPPER_SNAKE_CASE | `DEPLOYMENT_CREATED`, `LOGIN_FAILURE` |
| DB columns | snake_case | `project_name`, `deployed_by` |
| Migration files | `VN__snake_case_description.sql` | `V2__create_deployments_table.sql` |

### Test Conventions

```
methodName_givenCondition_thenExpectedResult()
```

Use `@DisplayName` on all test methods. Structure tests as `// given` / `// when` / `// then`. Test data uses builder pattern (`UserTestBuilder`, `DeploymentTestBuilder`).

---

## 3. Git Workflow

### Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `feature/short-description` | `feature/rollback-api` |
| Bug fix | `fix/short-description` | `fix/jwt-expiry-check` |
| Hotfix | `hotfix/short-description` | `hotfix/prod-null-pointer` |

All branches cut from `main`. PRs target `main`. No direct commits to `main`.

### Commit Message Format (Conventional Commits)

```
type(scope): short description

Body (optional): explain the why, not the what

Footer (optional): Closes #123
```

**Types:** `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `ci`

**Examples:**
```
feat(deployment): add paginated deployment list with multi-field filter
fix(auth): guard against null refresh token on app mount
test(rollback): add 422 case for rollback on SUCCESSFUL deployment
```

### PR Requirements

- At least one approving review
- All CI checks pass (build + tests + coverage ≥ 80%)
- Relevant docs updated in the same PR as the code change
- No `--no-verify` to skip hooks

---

## 4. Documentation Maintenance Rules

Keep docs in sync with code. These rules apply to every PR:

| Code Change | Required Doc Update |
|-------------|-------------------|
| New or modified API endpoint | Update `docs/BACKEND_SCHEMA.md` — endpoint catalog + DTO |
| New or modified DB schema | Update `docs/BACKEND_SCHEMA.md` — table definition + add Flyway migration |
| New significant architectural decision | Add ADR under `docs/ADR/` |
| New frontend page or component | Update `docs/UI_UX_BRIEF.md` — page inventory + component hierarchy |
| New user flow or state change | Update `docs/APP_FLOW.md` |
| New env variable | Update `docs/TRD.md` — environment variables table |

PRs that change behavior without updating relevant docs will be blocked in review.

---

## 5. Definition of Done (overall)

A feature is done when:

1. Code is merged to `main` with an approved PR
2. All tests pass (unit + integration)
3. Coverage ≥ 80% overall (enforced by CI)
4. The authorization matrix for the new endpoint is fully tested (no token → 401, wrong role → 403, correct role → 2xx, expired token → 401)
5. Relevant documentation updated in the same PR
6. Audit log entries verified in the database for any mutating operation
7. No plaintext passwords, secrets, or PII in source code or logs
