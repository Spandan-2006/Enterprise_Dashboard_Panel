# AGENTS.md — Enterprise Deployment Dashboard

This file gives AI coding agents the orientation needed to work effectively in this repository without requiring human guidance for routine tasks.

---

## Repository Orientation

### Folder Layout

```
Enterprise_Dashboard_Panel/
├── src/
│   ├── main/
│   │   ├── java/com/enterprise/dashboard/   # All backend Java source code
│   │   └── resources/                        # application.yml, static assets, migrations
│   └── test/
│       └── java/com/enterprise/dashboard/   # Unit and integration tests (mirrors main structure)
├── frontend/                                 # Optional React 18 frontend (may not exist yet)
│   ├── src/
│   ├── public/
│   └── package.json
├── docker/                                   # Dockerfile, docker-compose overrides, init scripts
├── docs/                                     # All project documentation (see README Quick Links)
│   └── ADR/                                  # Architecture Decision Records
├── .github/
│   └── workflows/                            # GitHub Actions CI/CD pipeline definitions
├── build.gradle                              # Gradle build script
├── settings.gradle                           # Gradle project settings
├── docker-compose.yml                        # Full-stack local environment
├── AGENTS.md                                 # This file
└── README.md                                 # Human-facing project overview
```

### Primary Language

**Java 21** (backend). The optional frontend is **TypeScript/JavaScript** (React 18).

---

## Build and Run Commands

All backend commands are run from the project root. The Gradle wrapper (`./gradlew`) is included — no separate Gradle installation is required.

> **Windows note:** On Windows Command Prompt, replace `./gradlew` with `gradlew.bat` in the commands below.

| Command | What it does |
|---|---|
| `./gradlew build` | Compile all sources, run all tests, and package the JAR |
| `./gradlew test` | Run all unit and integration tests |
| `./gradlew bootRun` | Start the Spring Boot application locally (requires a running PostgreSQL instance) |
| `docker-compose up --build` | Build the Docker image and start the full stack (app + PostgreSQL) |
| `docker-compose down -v` | Stop and remove all containers, networks, and named volumes |

Frontend commands (run from the `frontend/` directory):

```bash
cd frontend
npm install
npm run dev    # Start the Vite dev server (hot-reload)
npm run build  # Production build to frontend/dist/
```

---

## Architecture Rules

Agents must follow these rules when writing or modifying code. Violations will be caught in code review and must be corrected before merge.

1. **Controllers must never contain business logic** — delegate everything beyond request parsing and response formatting to a service class.
2. **Services must never directly query the database** — delegate all persistence operations to repository interfaces.
3. **All API responses must use the standard `ApiResponse<T>` wrapper** — never return a bare entity, list, or primitive from a controller method.
4. **DTOs must be used at the controller boundary; entities must not be serialized directly** — map entities to DTOs in the service layer before returning them to controllers.
5. **All new endpoints must have a corresponding JUnit 5 test** — use `@WebMvcTest` for controller slice tests or `@SpringBootTest` for integration tests; both are acceptable, but coverage is required.

---

## Package Structure

All Java source lives under the base package `com.enterprise.dashboard`. The sub-package layout is:

```
com.enterprise.dashboard
├── controller/        # @RestController classes — HTTP boundary only
├── service/           # @Service classes — all business logic
├── repository/        # @Repository interfaces extending JpaRepository
├── mapper/            # MapStruct or manual mappers — entity ↔ DTO conversion
├── model/
│   ├── entity/        # @Entity classes mapped to database tables
│   ├── dto/           # Request and response DTOs (records or plain classes)
│   └── enums/         # Domain enumerations (DeploymentStatus, Environment, Role)
├── security/          # JWT filter, UserDetailsService impl, security config
├── config/            # Spring @Configuration classes (CORS, Swagger, beans)
├── exception/         # Custom exception classes and @ControllerAdvice handler
└── audit/             # Audit log entity, service, and event listeners
```

When adding a new feature, create files in the appropriate sub-packages above. Do not create top-level packages outside this structure without an ADR.

---

## Key Enumerations

These enums are defined in `com.enterprise.dashboard.model.enums`. Use them — do not use raw strings to represent these concepts anywhere in the codebase.

### `DeploymentStatus`

Represents the lifecycle state of a deployment.

| Value | Meaning |
|---|---|
| `PENDING` | Deployment has been created but not yet started |
| `BUILDING` | Artifact build is in progress |
| `TESTING` | Automated tests are running against the built artifact |
| `DEPLOYING` | Artifact is being pushed to the target environment |
| `SUCCESSFUL` | Deployment completed without errors |
| `FAILED` | Deployment encountered a terminal error |
| `ROLLED_BACK` | A rollback was performed after a failed deployment |

Valid forward transitions: `PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL`.
Failure path:
```
PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL
              ↓           ↓          ↓
            FAILED      FAILED    FAILED → ROLLED_BACK
```

### `Environment`

Represents the target deployment environment.

| Value | Notes |
|---|---|
| `DEV` | Developer sandbox; no approval required |
| `QA` | Quality assurance environment; requires DEVOPS_ENGINEER or ADMIN role |
| `STAGING` | Pre-production mirror; requires DEVOPS_ENGINEER or ADMIN role |
| `PRODUCTION` | Live environment; requires ADMIN role |

### `Role`

Controls access to API endpoints via Spring Security.

| Value | Permissions summary |
|---|---|
| `ADMIN` | Full access to all endpoints including user management and environment configuration |
| `DEVOPS_ENGINEER` | Can create, update, and rollback deployments; cannot manage users |
| `DEVELOPER` | Read-only access to deployments; can create deployments to DEV only |

---

## Testing Requirements

- Every **service method** must have at least one **unit test** using JUnit 5 and Mockito. Mock all repository dependencies.
- Every **controller endpoint** must have at least one test using `@WebMvcTest` (preferred for fast feedback) or `@SpringBootTest` (for integration scenarios).
- The project enforces a minimum **80% line coverage** threshold via the JaCoCo Gradle plugin. The build will fail if coverage drops below this threshold.
- Tests live in `src/test/java/` and mirror the package structure of the class under test.
- Use `@DisplayName` on test methods to produce readable test reports.
- Do not use `@SpringBootTest` when `@WebMvcTest` or `@DataJpaTest` slices are sufficient — full context startup is slow.

---

## Security Rules

These rules are non-negotiable. Any code that violates them must not be merged.

- **Never log passwords, tokens, or PII.** This includes debug-level log statements. Scrub or mask sensitive fields before logging.
- **The JWT signing secret must always come from the `JWT_SECRET` environment variable.** It must never be hardcoded in source code, `application.yml`, or any committed configuration file.
- **All endpoints except `/api/auth/**`, `/swagger-ui/**`, and `/actuator/health` require authentication.** The Spring Security configuration must enforce this at the filter chain level; do not rely solely on `@PreAuthorize` for public endpoints.
- Passwords must always be hashed with BCrypt before persistence. Never store or compare plaintext passwords.
- CORS must be explicitly configured; do not use a wildcard origin (`*`) in `staging` or `production` profiles.
- Validate all user-supplied input using Jakarta Bean Validation annotations (`@NotBlank`, `@Size`, etc.) on DTO fields. The global `@ControllerAdvice` handler catches `MethodArgumentNotValidException` and returns a structured error response.

---

## Docs Maintenance Rules

Keep documentation in sync with code changes. Specifically:

- **When adding or modifying an API endpoint**, update `docs/06-api-spec.md` with the new/changed path, method, request body, response body, and required role.
- **When changing the database schema** (adding a table, column, index, or constraint), update `docs/07-database.md` and add the corresponding Flyway migration script under `src/main/resources/db/migration/`.
- **When making a significant architectural decision** (new framework, design pattern, data store, or deviation from existing conventions), create a new ADR under `docs/ADR/` following the existing naming convention (`ADR-NNN-short-title.md`). An ADR should document: context, decision, consequences, and alternatives considered.
- Documentation updates must be included in the same pull request as the code change they describe. PRs that change behavior without updating relevant docs will be blocked in review.
