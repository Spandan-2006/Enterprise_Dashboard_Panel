# AGENTS.md — Enterprise Deployment Dashboard

This file gives AI coding agents the orientation needed to work effectively in this repository without requiring human guidance for routine tasks.

---

## Documentation Map

Start here before writing any code. These six files contain the complete project specification.

| File | Purpose |
|------|---------|
| [docs/PRD.md](docs/PRD.md) | Product requirements — user stories, functional requirements, success metrics |
| [docs/TRD.md](docs/TRD.md) | Technical requirements — tech stack, architecture, security, environments, testing |
| [docs/APP_FLOW.md](docs/APP_FLOW.md) | User flows, route structure, deployment state machine |
| [docs/UI_UX_BRIEF.md](docs/UI_UX_BRIEF.md) | React component hierarchy, status badge design, API layer, responsive layout |
| [docs/BACKEND_SCHEMA.md](docs/BACKEND_SCHEMA.md) | DB schema (SQL), JPA entities, DTOs, service interfaces, full API contract |
| [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md) | Phased build order, coding standards, git workflow, definition of done |

---

## Repository Layout

```
Enterprise_Dashboard_Panel/
├── src/
│   ├── main/
│   │   ├── java/com/enterprise/dashboard/   # All backend Java source code
│   │   └── resources/
│   │       ├── application.yml               # Shared config + profile placeholders
│   │       ├── application-dev.yml
│   │       ├── application-test.yml
│   │       └── db/migration/                 # Flyway V1–V6 migration scripts
│   └── test/
│       └── java/com/enterprise/dashboard/   # Mirrors main package structure
├── frontend/                                 # Optional React 18 frontend
│   ├── src/
│   └── package.json
├── docker/                                   # Dockerfile, Docker Compose overrides
├── docs/                                     # Six project spec documents (see above)
├── .github/workflows/                        # GitHub Actions CI/CD pipeline
├── build.gradle.kts                          # Gradle Kotlin DSL build script
├── docker-compose.yml
├── AGENTS.md                                 # This file
└── README.md
```

---

## Build & Run Commands

> **Windows note:** Replace `./gradlew` with `gradlew.bat` on Windows Command Prompt.

| Command | What it does |
|---------|-------------|
| `./gradlew build` | Compile, test, coverage check, package JAR |
| `./gradlew test` | Run all unit and integration tests |
| `./gradlew test jacocoTestCoverageVerification` | Tests + enforce ≥ 80% line coverage (fails build if below) |
| `./gradlew bootRun` | Start Spring Boot app (requires running PostgreSQL) |
| `docker-compose up --build` | Build Docker image + start full stack (app + PostgreSQL 15) |
| `docker-compose down -v` | Stop and remove all containers and volumes |

Frontend commands (from `frontend/` directory):

```bash
npm install
npm run dev    # Vite dev server with HMR
npm run build  # Production build to frontend/dist/
```

---

## Architecture Rules

Violations will be caught in code review and must be corrected before merge.

1. **Controllers contain zero business logic** — delegate everything beyond request parsing and response formatting to a `@Service` class
2. **Services never call repositories from other services' domains** — `DeploymentService` does not call `UserRepository` directly
3. **All API responses use `ApiResponse<T>`** — never return a bare entity, list, or primitive from a controller method
4. **Entities never cross the service boundary** — map to DTOs before returning from any service method
5. **Every new endpoint needs tests** — minimum: no token → 401, wrong role → 403, correct role → 2xx, expired token → 401

---

## Package Structure

```
com.enterprise.dashboard
├── controller/    — @RestController (HTTP boundary only)
├── service/       — @Service interfaces + impl/ subpackage (all business logic)
├── repository/    — @Repository interfaces extending JpaRepository
├── mapper/        — entity ↔ DTO mappers
├── model/
│   ├── entity/    — @Entity classes
│   ├── dto/       — request + response DTOs
│   └── enums/     — Role, Environment, DeploymentStatus
├── security/      — JwtAuthFilter, CustomUserDetailsService, SecurityConfig, JwtUtil
├── config/        — CORS, Swagger, other @Configuration beans
├── exception/     — AppException hierarchy, GlobalExceptionHandler
└── audit/         — AuditLog entity, AuditService, AuditActions constants
```

Do not create top-level packages outside this structure.

---

## Key Enumerations

Package: `com.enterprise.dashboard.model.enums`. Use these — never raw strings.

### `DeploymentStatus`

| Value | Meaning |
|-------|---------|
| `PENDING` | Created, not yet started |
| `BUILDING` | Artifact build in progress |
| `TESTING` | Automated tests running |
| `DEPLOYING` | Artifact being pushed to environment |
| `SUCCESSFUL` | Completed without errors (terminal) |
| `FAILED` | Terminal error (terminal) |
| `ROLLED_BACK` | Rollback performed after failure (terminal) |

```
PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL
    ↓          ↓         ↓          ↓
  FAILED    FAILED    FAILED    FAILED → ROLLED_BACK (via rollback endpoint only)
```

### `Environment`

`DEV` | `QA` | `STAGING` | `PRODUCTION`

### `Role`

`ADMIN` | `DEVOPS_ENGINEER` | `DEVELOPER`

Full authorization matrix → [docs/TRD.md](docs/TRD.md#authorization-matrix)

---

## Security Rules (Non-negotiable)

- **Never log passwords, tokens, or PII.** Scrub sensitive fields before any log statement.
- **JWT signing secret from `JWT_SECRET` env var only.** Never hardcoded in source or config files.
- **All endpoints except `/api/auth/**`, `/swagger-ui/**`, `/v3/api-docs/**`, and `/actuator/health` require authentication.** Enforced at the `SecurityFilterChain` level, not only `@PreAuthorize`.
- **BCrypt before any password persist.** Call `passwordEncoder.encode()` — never store or compare plaintext.
- **CORS explicit.** No wildcard origin `*` in staging or production profiles.
- **`@Valid` on all `@RequestBody` parameters.** The `GlobalExceptionHandler` catches `MethodArgumentNotValidException` and returns a structured 400 response.

---

## Testing Requirements

- Every **service method** → at least one **unit test** (`@ExtendWith(MockitoExtension.class)`; mock all repo dependencies)
- Every **controller endpoint** → at least one integration test (`@SpringBootTest` + `@AutoConfigureMockMvc` for security tests; `@WebMvcTest` for fast slice tests)
- **Minimum 80% line coverage** enforced by JaCoCo. Build fails below this threshold.
- Tests live in `src/test/java/` and mirror the main package structure
- Naming convention: `methodName_givenCondition_thenExpectedResult()`
- Use `@DisplayName` for readable test reports
- Use `UserTestBuilder` and `DeploymentTestBuilder` for test data

---

## Docs Maintenance Rules

Include documentation updates in the same PR as the code change.

| Code Change | Required Doc Update |
|-------------|-------------------|
| New or modified API endpoint | `docs/BACKEND_SCHEMA.md` — endpoint catalog + DTOs |
| New or modified DB schema | `docs/BACKEND_SCHEMA.md` — table definitions + add Flyway migration |
| New user flow or state | `docs/APP_FLOW.md` |
| New frontend page or component | `docs/UI_UX_BRIEF.md` |
| New environment variable | `docs/TRD.md` — environment variables table |
| Significant architectural decision | Create ADR under `docs/ADR/` (format: `ADR-NNN-short-title.md`) |

PRs that change behavior without updating relevant docs will be blocked in review.
