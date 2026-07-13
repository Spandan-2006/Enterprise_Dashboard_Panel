# docs/14-coding-standards.md — Coding Standards

## 1. Java Code Style

The project follows the **Google Java Style Guide** as its base standard, with the following explicit settings:

| Rule | Value |
|---|---|
| Indentation unit | 4 spaces |
| Tabs | Not permitted |
| Maximum line length | 120 characters |
| Brace style | K&R (opening brace on the same line as the declaration) |
| Trailing whitespace | Not permitted |
| Blank lines between methods | 1 |
| Blank lines at the start/end of a block | 0 |
| Import ordering | Static imports first, then non-static, each group alphabetically sorted; no wildcard imports |

### Enforcement

Code style is automatically checked and applied by the **Spotless** Gradle plugin configured in `build.gradle.kts`. The check runs during `./gradlew build` and fails the build on any style violation.

```kotlin
// build.gradle.kts (Spotless configuration excerpt)
spotless {
    java {
        googleJavaFormat("1.18.1")
        removeUnusedImports()
        trimTrailingWhitespace()
        endWithNewline()
    }
}
```

To auto-fix style violations locally before committing:

```bash
./gradlew spotlessApply
```

### IDE Setup

- **IntelliJ IDEA** — Install the **google-java-format** plugin and enable "Reformat code on save". Import the `.editorconfig` from the project root to align indentation and line-ending settings.
- **VS Code** — Install the **Language Support for Java** extension and configure `editor.formatOnSave: true`.

The `.editorconfig` at the project root is the single source of truth for whitespace rules and takes precedence over IDE defaults.

---

## 2. Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Classes | PascalCase | `DeploymentService`, `GlobalExceptionHandler` |
| Interfaces | PascalCase, no "I" prefix | `UserRepository`, `DeploymentRepository` |
| Methods | camelCase | `createDeployment()`, `triggerRollback()` |
| Local variables | camelCase | `deployedBy`, `rollbackReason` |
| Instance fields | camelCase | `deploymentRepository`, `jwtService` |
| Constants (`static final`) | UPPER_SNAKE_CASE | `JWT_EXPIRATION_MS`, `MAX_PAGE_SIZE` |
| Packages | all lowercase, dot-separated | `com.enterprise.dashboard.service` |
| Database tables | snake_case, plural nouns | `audit_logs`, `deployment_rollbacks` |
| Database columns | snake_case | `project_name`, `deployed_by` |
| REST endpoints | kebab-case plural nouns | `/api/deployments`, `/api/audit-logs` |
| Environment variables | UPPER_SNAKE_CASE | `JWT_SECRET`, `POSTGRES_PASSWORD` |
| DTO classes | Suffix with `Request` or `Response` | `CreateDeploymentRequest`, `DeploymentResponse` |
| Exception classes | Suffix with `Exception` | `ResourceNotFoundException`, `ForbiddenException` |
| Test classes | Suffix with `Test` | `DeploymentServiceTest`, `AuthControllerTest` |
| Spring configuration classes | Suffix with `Config` | `SecurityConfig`, `CorsConfig` |

**Interface naming rationale:** The "I" prefix convention (e.g., `IDeploymentService`) is a C# convention and is not idiomatic Java. Spring dependency injection uses the concrete type for injection; clients program to the interface type, making the "I" prefix redundant.

---

## 3. Spring Boot Layer Conventions

The application uses a three-tier layered architecture: Controller → Service → Repository. Each layer has strict responsibilities. Violating these boundaries creates tight coupling and makes testing significantly harder.

### Controller (`@RestController`)

- **Allowed:** Parse HTTP request data, call a single service method, map the service result to an HTTP response, return `ResponseEntity<T>`.
- **Not allowed:** Any business logic, direct repository access, exception handling (let `GlobalExceptionHandler` handle it), `@Transactional`.
- No constructor injection of repositories — inject services only.
- Method bodies should typically be 3–5 lines.

### Service (`@Service`)

- **Allowed:** Business logic, orchestration of multiple repository calls, throwing domain-specific exceptions, mapping entities to DTOs.
- `@Transactional` at **method level**, not class level. Apply it only to methods that actually modify state. Read-only methods should use `@Transactional(readOnly = true)`.
- Never return JPA `@Entity` objects from public service methods — always map to a DTO before returning.
- Services must be unit-testable without a running Spring context. Constructor injection of all dependencies enables this.

### Repository (`@Repository`)

- Extend `JpaRepository<Entity, IdType>` or `PagingAndSortingRepository`.
- **Allowed:** Custom JPQL or Spring Data query methods.
- **Not allowed:** Business logic, exception handling, DTO mapping.
- All custom query methods must be documented with Javadoc (see Section 6).

### Configuration (`@Configuration`)

- No XML configuration files. All configuration is Java-based using `@Configuration` and `@Bean` methods.
- Each `@Configuration` class should have a focused concern (e.g., `SecurityConfig`, `CorsConfig`, `JwtConfig`).
- Do not scatter `@Bean` definitions across `@Service` or `@Controller` classes.

### Dependency Injection

Constructor injection is mandatory for all Spring-managed beans. Field injection (`@Autowired` on a field) and setter injection are not permitted.

```java
// Correct
@Service
public class DeploymentService {

    private final DeploymentRepository deploymentRepository;
    private final UserRepository userRepository;
    private final AuditLogService auditLogService;

    public DeploymentService(DeploymentRepository deploymentRepository,
                             UserRepository userRepository,
                             AuditLogService auditLogService) {
        this.deploymentRepository = deploymentRepository;
        this.userRepository = userRepository;
        this.auditLogService = auditLogService;
    }
}

// Also correct — Lombok @RequiredArgsConstructor generates the constructor
@RequiredArgsConstructor
@Service
public class DeploymentService {
    private final DeploymentRepository deploymentRepository;
    private final UserRepository userRepository;
    private final AuditLogService auditLogService;
}
```

**Rationale for constructor injection:**
1. Dependencies are explicit and visible in the constructor signature.
2. The class can be instantiated in unit tests with mock objects without a Spring context.
3. `final` fields guarantee the dependency is set exactly once, preventing accidental null references.

---

## 4. DTO and Mapping Rules

### DTO Naming

- Request DTOs carry data from the client to the server. Suffix: `Request`.
  - `CreateDeploymentRequest`, `UpdateDeploymentStatusRequest`, `LoginRequest`
- Response DTOs carry data from the server to the client. Suffix: `Response`.
  - `DeploymentResponse`, `UserResponse`, `AuthResponse`
- Pagination wrapper: `PageResponse<T>` — a generic wrapper that includes `content`, `totalElements`, `totalPages`, `page`, and `size`.

### Entity Exposure Rule

JPA `@Entity` objects must never be returned from `@RestController` methods. Entities are mutable, may contain lazy-loaded collections, and expose the full database schema to API consumers. Always map to a DTO before returning from the controller.

Violations of this rule cause:
- `LazyInitializationException` at serialization time (Jackson tries to serialize a lazy collection outside a Hibernate session).
- Unintended data exposure (password hashes, internal IDs, audit metadata).
- Tight coupling between the API contract and the database schema.

### Mapping Location

In v1, entity-to-DTO mapping is performed in the service layer using private helper methods. MapStruct is not a v1 dependency; it will be evaluated for v2 if manual mapping becomes a maintenance burden.

```java
// In DeploymentService.java
private DeploymentResponse toResponse(Deployment deployment) {
    return new DeploymentResponse(
        deployment.getId(),
        deployment.getProjectName(),
        deployment.getEnvironment(),
        deployment.getStatus(),
        deployment.getVersion(),
        deployment.getDeployedBy(),
        deployment.getCreatedAt()
    );
}
```

The mapping method is `private` — it is an implementation detail of the service and should not be tested in isolation.

---

## 5. Exception Handling Rules

### Rule 1 — No empty catch blocks

```java
// Forbidden
try {
    deploymentRepository.save(deployment);
} catch (Exception e) {}

// Required minimum — log the exception
try {
    deploymentRepository.save(deployment);
} catch (DataAccessException e) {
    log.error("Failed to persist deployment: deploymentId={}", deployment.getId(), e);
    throw new AppException("Deployment could not be saved", e);
}
```

### Rule 2 — Throw domain-specific exceptions from the service layer

The application defines a hierarchy of domain exceptions in `com.enterprise.dashboard.exception`:

| Exception Class | HTTP Status | Use Case |
|---|---|---|
| `ResourceNotFoundException` | 404 Not Found | Entity looked up by ID does not exist |
| `ForbiddenException` | 403 Forbidden | Authenticated user lacks the required role |
| `ConflictException` | 409 Conflict | Business rule violation (e.g., duplicate name) |
| `BadRequestException` | 400 Bad Request | Invalid input that passes bean validation but fails business validation |
| `AppException` | 500 Internal Server Error | Unexpected infrastructure failure |

All domain exceptions extend a common `AppException` base class. `GlobalExceptionHandler` maps each to the correct HTTP status and a consistent `ErrorResponse` body.

### Rule 3 — No try-catch in controllers

Controllers must not contain try-catch blocks. All exceptions thrown by service methods are caught and handled by `GlobalExceptionHandler` (`@RestControllerAdvice`). This centralises error response formatting in one place.

### Rule 4 — Consistent 404 responses via `ResourceNotFoundException`

```java
// In any service method
Deployment deployment = deploymentRepository.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("Deployment", id));
```

`ResourceNotFoundException` produces a consistent 404 body:

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Deployment with id 42 was not found",
  "timestamp": "2025-07-13T14:32:01.456Z",
  "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

## 6. Javadoc Requirements

Javadoc is required on the following elements. The build should emit warnings (configured via Checkstyle or a custom Gradle task in v2) for missing Javadoc on required elements.

### Required Javadoc Locations

| Location | Why |
|---|---|
| All public service interface methods | Documents the contract that implementations must honour |
| All `@RestController` methods | Documents the HTTP endpoint behaviour for developers reading the code (supplements Swagger annotations) |
| All `@Entity` classes | Documents the domain concept and any non-obvious business rules embedded in the entity |
| All custom `@Repository` query methods | Documents the query intent, parameter semantics, and return value |

### Minimum Javadoc Format

```java
/**
 * Creates a new deployment record with PENDING status.
 *
 * <p>Only users with the ADMIN or MANAGER role may create deployments. Callers with the
 * DEVELOPER role will receive a {@link ForbiddenException}.
 *
 * <p>The deployment is persisted with status {@code PENDING}. A subsequent call to
 * {@link #updateDeploymentStatus} is required to advance it to {@code IN_PROGRESS} or beyond.
 *
 * @param request the deployment creation data; must not be null; all fields validated by
 *                {@link jakarta.validation.Valid}
 * @param username the username of the authenticated user triggering the deployment;
 *                 stored as the {@code deployedBy} field on the record
 * @return {@link DeploymentResponse} containing the full details of the created deployment,
 *         including the auto-generated ID and creation timestamp
 * @throws ForbiddenException if the caller has the DEVELOPER role
 * @throws ResourceNotFoundException if the project referenced in the request does not exist
 */
DeploymentResponse createDeployment(CreateDeploymentRequest request, String username);
```

### What Javadoc Should NOT Do

- Do not restate what is obvious from the method signature. `/** Returns the deployment. */` on a method named `getDeployment` adds no value.
- Do not explain implementation details in interface Javadoc. Describe the contract (what it does), not how it does it.
- Do not copy-paste the same Javadoc to both the interface method and the implementation. The implementation's Javadoc (if any) should note only differences from the interface contract; otherwise use `{@inheritDoc}`.

---

## 7. Test Code Standards

### Test Class Structure

- Naming: `{ClassUnderTest}Test` — e.g., `DeploymentServiceTest`, `AuthControllerTest`
- Location: mirror the source package under `src/test/java/`
- Use `@ExtendWith(MockitoExtension.class)` for unit tests that mock dependencies
- Use `@SpringBootTest` + `@AutoConfigureMockMvc` for integration tests that require the full application context

### Test Method Naming and Display Names

Every `@Test` method must have a `@DisplayName` annotation with a human-readable sentence describing what the test verifies:

```java
@Test
@DisplayName("createDeployment throws ForbiddenException when caller has DEVELOPER role")
void createDeployment_developerRole_throwsForbiddenException() { ... }

@Test
@DisplayName("createDeployment returns DeploymentResponse with PENDING status on success")
void createDeployment_validInput_returnsPendingStatus() { ... }
```

The method name follows the pattern `{method}_{condition}_{expectedOutcome}` (readable without camelCase parsing). The `@DisplayName` is what appears in test reports and must be written as a full sentence.

### Assertions

Use **AssertJ** (`assertThat()`), not JUnit 5's `assertEquals()`. AssertJ produces more descriptive failure messages and supports fluent chained assertions.

```java
// Preferred — AssertJ
assertThat(result.getStatus()).isEqualTo(DeploymentStatus.PENDING);
assertThat(result.getProjectName()).isEqualTo("payments-api");

// Not permitted — JUnit assertEquals
assertEquals(DeploymentStatus.PENDING, result.getStatus());
```

Use `assertAll()` to group related assertions so that all failures are reported in a single test run, not just the first:

```java
assertAll(
    "deployment response fields",
    () -> assertThat(result.getId()).isNotNull(),
    () -> assertThat(result.getStatus()).isEqualTo(DeploymentStatus.PENDING),
    () -> assertThat(result.getDeployedBy()).isEqualTo("johndoe")
);
```

### Asynchronous Testing

Never use `Thread.sleep()` in tests. Use **Awaitility** for assertions that require waiting for an asynchronous condition:

```java
await().atMost(5, SECONDS)
       .until(() -> deploymentRepository.findById(id).isPresent());
```

### Mock Verification Rules

Do not write test methods whose sole assertion is that a mock was called:

```java
// Forbidden — verifies mock interaction but asserts nothing about the outcome
verify(auditLogRepository).save(any());

// Required — verify the outcome; mock verification is a secondary check if needed
assertThat(savedDeployment.getStatus()).isEqualTo(DeploymentStatus.PENDING);
verify(auditLogRepository).save(argThat(log -> log.getAction().equals("DEPLOYMENT_CREATED")));
```

### Test Data Builders

Do not construct test entities inline with `new Entity()` and twenty setter calls. Use fluent test builder classes located in `src/test/java/com/enterprise/dashboard/builder/`:

```java
// Fluent test builders
User testUser = UserTestBuilder.aUser()
    .withUsername("johndoe")
    .withRole(Role.MANAGER)
    .build();

Deployment testDeployment = DeploymentTestBuilder.aDeployment()
    .withProject("payments-api")
    .withEnvironment(Environment.STAGING)
    .withStatus(DeploymentStatus.PENDING)
    .build();
```

Builder classes have sensible defaults for all fields so tests can specify only the fields relevant to the scenario being tested.

---

## 8. Logging Standards

### Logger Declaration

Use the SLF4J API exclusively. Declare the logger as a `private static final` field:

```java
private static final Logger log = LoggerFactory.getLogger(DeploymentService.class);
```

Or, if Lombok is already used in the class, use the `@Slf4j` annotation (which generates the equivalent `log` field at compile time):

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class DeploymentService {
    // 'log' is available without explicit declaration
}
```

Do not mix both styles within the same class.

### Log Level Guidelines

| Level | When to Use | Example |
|---|---|---|
| `DEBUG` | Entry and exit of service methods, intermediate computed values, query parameters | `log.debug("Finding deployments for user: username={}", username)` |
| `INFO` | Significant business events that represent successful state transitions | `log.info("Deployment created: id={}, project={}, env={}, by={}", id, project, env, user)` |
| `WARN` | Recoverable issues that do not fail the current operation but may indicate a problem | `log.warn("JWT refresh token near expiry: username={}, expiresAt={}", username, expiry)` |
| `ERROR` | Exceptions that propagate to the caller and result in a non-200 response | `log.error("Failed to connect to database during deployment save: deploymentId={}", id, ex)` |

**Always pass the exception as the last argument to `log.error()` or `log.warn()`** — this enables Logback to include the full stack trace in the log output.

### Data Masking Rules

The following fields must never appear unmasked in any log statement at any level:

| Field | Required Masking | Example |
|---|---|---|
| `password` | Do not log at all | Omit from log statements entirely |
| JWT token (full) | Truncate to first 20 characters + `...` | `eyJhbGciOiJIUzI1Ni...` |
| Email address | Mask local part: `em***@domain.com` | `jo***@example.com` |

```java
// Forbidden
log.debug("Login attempt: username={}, password={}", username, password);

// Correct
log.debug("Login attempt: username={}", username);
// Do not log the password at all — not even at TRACE level
```

### Structured Log Fields

For log statements that are likely to be searched or aggregated (INFO and above in the service layer), use key-value pairs in the message rather than interpolated strings:

```java
// Preferred — key=value pairs are easy to parse with log aggregation tools
log.info("Deployment status updated: id={}, oldStatus={}, newStatus={}, by={}",
         deployment.getId(), oldStatus, newStatus, username);

// Acceptable for brief messages
log.info("User logged in: username={}", username);
```

The `requestId`, `username`, and `endpoint` MDC fields are injected automatically by the `RequestContextFilter` — do not include them redundantly in the message.
