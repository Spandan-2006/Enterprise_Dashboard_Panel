# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This is a pre-implementation project. All specifications are written; no source code exists yet. Read `docs/IMPLEMENTATION_PLAN.md` for the phased build order before writing any code. The six spec documents in `docs/` are the source of truth — do not invent requirements.

## Commands

All backend commands run from the project root. On Windows, replace `./gradlew` with `gradlew.bat`.

```bash
# Build and test
./gradlew build                                  # compile + test + coverage check + JAR
./gradlew test                                   # run all tests only
./gradlew test --tests "*.DeploymentServiceTest" # run a single test class
./gradlew test --tests "*.DeploymentServiceTest.createDeployment_givenValidRequest*" # single test method
./gradlew test jacocoTestCoverageVerification    # tests + fail if line coverage < 80%

# Run locally (requires PostgreSQL)
./gradlew bootRun

# Full stack via Docker
docker-compose up --build    # app + PostgreSQL 15
docker-compose down -v       # stop and remove volumes

# Frontend (from frontend/ directory)
npm run dev      # Vite dev server
npm run build    # production build
```

Coverage reports land in `build/reports/jacoco/test/html/index.html`. Test reports in `build/reports/tests/test/index.html`.

## Architecture

Strict layered architecture — each layer communicates only with the layer directly below it:

```
Controllers → Services → Repositories → PostgreSQL
```

The cross-cutting rules that are easy to get wrong:

- **Controllers** do zero business logic — only parse requests, call one service method, and wrap the result in `ApiResponse<T>`. Every controller method returns `ResponseEntity<ApiResponse<T>>`.
- **Services** map entities to DTOs before returning — entities must never cross the service boundary into controllers.
- **`DeploymentService` owns the state machine** — `updateStatus()` enforces transitions and `triggerRollback()` is the only path to `ROLLED_BACK`. Status transitions that violate the machine throw `InvalidStatusTransitionException` (HTTP 422).
- **`AuditService.log()`** must be called within the same `@Transactional` boundary as the parent operation so audit entries roll back with failures. Exception: auth events (`LOGIN_SUCCESS`, `LOGIN_FAILURE`) use `Propagation.REQUIRES_NEW` so they are never lost to a rollback.
- **`deployments.deployed_by`** and **`audit_logs.username`** are denormalized VARCHAR copies — not foreign keys — so records survive user deletion.

## Package Structure

```
com.enterprise.dashboard
├── controller/          # @RestController — HTTP boundary only
├── service/impl/        # @Service interfaces + implementations
├── repository/          # JpaRepository interfaces
├── mapper/              # entity ↔ DTO conversion
├── model/entity/        # @Entity classes
├── model/dto/           # request and response DTOs
├── model/enums/         # Role, Environment, DeploymentStatus
├── security/            # JwtAuthFilter, CustomUserDetailsService, SecurityConfig, JwtUtil
├── config/              # CORS, Swagger, other @Configuration beans
├── exception/           # AppException hierarchy + GlobalExceptionHandler
└── audit/               # AuditLog, AuditService, AuditActions constants
```

## Deployment State Machine

```
PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL (terminal)
    ↓          ↓         ↓          ↓
  FAILED    FAILED    FAILED    FAILED (terminal)
                                   ↓
                             ROLLED_BACK (terminal — only via POST /api/deployments/{id}/rollback)
```

`endTime` is set explicitly in `DeploymentService.updateStatus()` when status reaches any terminal state. Do not use `@UpdateTimestamp` for `endTime`.

## Security Constraints

- JWT secret comes from `JWT_SECRET` env var only (min 256 bits, Base64-encoded, HS256)
- Public endpoints (no token required): `/api/auth/**`, `/swagger-ui/**`, `/v3/api-docs/**`, `/actuator/health`
- All other endpoints enforced at the `SecurityFilterChain` level (not only `@PreAuthorize`)
- CORS origin from `CORS_ALLOWED_ORIGINS` env var — never wildcard `*` in staging/prod
- Full authorization matrix (which role can call which endpoint): `docs/TRD.md` § Authorization Matrix

## Testing Conventions

- **Service tests**: `@ExtendWith(MockitoExtension.class)`, mock all repo dependencies
- **Controller tests**: `@SpringBootTest + @AutoConfigureMockMvc` for security tests (loads full filter chain); `@WebMvcTest` for logic-only slice tests
- **Repo tests**: `@DataJpaTest` with `@ActiveProfiles("test")` — uses H2 in-memory; Flyway disabled in test profile
- Test naming: `methodName_givenCondition_thenExpectedResult()`
- Each new endpoint needs four security tests: no token → 401, wrong role → 403, correct role → 2xx, expired token → 401
- Test data via `UserTestBuilder` and `DeploymentTestBuilder` builder classes

## Key Spec Documents

| Question | Document |
|----------|---------|
| What to build / user stories | `docs/PRD.md` |
| Tech stack, env vars, security design, ADRs | `docs/TRD.md` |
| User flows, screen navigation, state machine | `docs/APP_FLOW.md` |
| React components, API layer, responsive layout | `docs/UI_UX_BRIEF.md` |
| DB schema (SQL), JPA entities, DTOs, full API contract | `docs/BACKEND_SCHEMA.md` |
| Build phases, coding standards, git workflow | `docs/IMPLEMENTATION_PLAN.md` |
