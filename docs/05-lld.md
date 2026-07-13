# Low-Level Design (LLD)

**Project:** Enterprise Deployment Dashboard
**Version:** 1.0
**Date:** 2026-07-13
**Status:** Approved

---

## Table of Contents

1. [Entity Class Diagrams](#1-entity-class-diagrams)
2. [Service Interface Definitions](#2-service-interface-definitions)
3. [DTO Specifications](#3-dto-specifications)
4. [Repository Method Definitions](#4-repository-method-definitions)
5. [Exception Hierarchy](#5-exception-hierarchy)
6. [GlobalExceptionHandler Mapping Table](#6-globalexceptionhandler-mapping-table)
7. [AuditLog Trigger Points](#7-auditlog-trigger-points)

---

## 1. Entity Class Diagrams

All entities reside in the `com.enterprise.dashboard.model.entity` package. Hibernate/JPA manages DDL via `spring.jpa.hibernate.ddl-auto`. Timestamps are supplied by Hibernate's `@CreationTimestamp` and never set manually.

### 1.1 User

```
@Entity
@Table(name = "users")
class User {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id

  @Column(nullable=false, unique=true, length=50)
  String username

  @Column(nullable=false, unique=true, length=100)
  String email

  @Column(nullable=false)
  String password          // BCrypt hash — @JsonIgnore, never serialized in any response

  @Enumerated(EnumType.STRING)
  @Column(nullable=false, length=20)
  Role role                // Enum: ADMIN | DEVOPS_ENGINEER | DEVELOPER

  @CreationTimestamp
  @Column(name="created_at", updatable=false)
  LocalDateTime createdAt
}
```

**Column notes:**
- `username`: max 50 chars; serves as the principal name stored in the JWT subject claim
- `password`: stored as BCrypt ($2a$) hash; never returned in any API response
- `role`: persisted as string (e.g. `"ADMIN"`) — avoids ordinal instability on enum reordering

**Indexes:** unique index on `username`, unique index on `email` (enforced at DB level in addition to `unique=true`)

---

### 1.2 Deployment

```
@Entity
@Table(name = "deployments")
class Deployment {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id

  @Column(name="project_name", nullable=false, length=100)
  String projectName

  @Column(nullable=false, length=50)
  String version

  @Enumerated(EnumType.STRING)
  @Column(nullable=false, length=20)
  Environment environment  // Enum: DEV | QA | STAGING | PRODUCTION

  @Enumerated(EnumType.STRING)
  @Column(nullable=false, length=30)
  DeploymentStatus status  // Enum (default: PENDING). See state machine in HLD §5.

  @Column(name="deployed_by", nullable=false, length=50)
  String deployedBy        // Username of the creator — denormalized for query efficiency

  @CreationTimestamp
  @Column(name="start_time", updatable=false)
  LocalDateTime startTime

  @Column(name="end_time")
  LocalDateTime endTime    // Nullable; set when a terminal state is reached
                           // Terminal states: SUCCESSFUL | FAILED | ROLLED_BACK

  @Column(columnDefinition="TEXT")
  String remarks           // Nullable free-text field; updatable via PUT /api/deployments/{id}
}
```

**Column notes:**
- `deployedBy` stores the username string rather than a FK to `users.id`. This allows audit records to survive user deletion without cascading side effects.
- `endTime` is set by `DeploymentService.updateStatus()` whenever the new status is a terminal state (`SUCCESSFUL`, `FAILED`, `ROLLED_BACK`). It is never set on creation.
- `status` defaults to `PENDING` in the service layer before persistence; no `@ColumnDefault` annotation is used.

**Indexes:** composite index on `(environment, status)` for the most common filter combination; individual indexes on `deployed_by`, `status`, `start_time`.

---

### 1.3 Rollback

```
@Entity
@Table(name = "rollbacks")
class Rollback {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id

  @ManyToOne(fetch=FetchType.LAZY)
  @JoinColumn(name="deployment_id", nullable=false)
  Deployment deployment    // FK to deployments.id — ON DELETE RESTRICT (prevents orphan)

  @Column(name="previous_version", nullable=false, length=50)
  String previousVersion   // Copied from deployment.version at rollback time

  @Column(name="rollback_reason", nullable=false, columnDefinition="TEXT")
  String rollbackReason    // Operator-supplied reason

  @CreationTimestamp
  @Column(name="rollback_time", updatable=false)
  LocalDateTime rollbackTime
}
```

**Column notes:**
- `deployment` uses `FetchType.LAZY` to avoid N+1 issues when listing rollbacks.
- `previousVersion` is copied from `deployment.version` at the moment of rollback; the deployment's version is not changed.
- The ON DELETE RESTRICT constraint is the mechanism that causes `ConflictException` when `DELETE /api/deployments/{id}` is attempted and rollback records exist.

**Indexes:** index on `deployment_id` (implicit from FK); composite index on `(deployment_id, rollback_time DESC)` for `findByDeploymentIdOrderByRollbackTimeDesc`.

---

### 1.4 AuditLog

```
@Entity
@Table(name = "audit_logs")
class AuditLog {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id

  @Column(nullable=false, length=50)
  String username          // Actor — may be "SYSTEM" for automated events

  @Column(nullable=false, length=50)
  String action            // Constant string, e.g. "LOGIN_SUCCESS", "DEPLOYMENT_CREATED"

  @CreationTimestamp
  @Column(name="timestamp", updatable=false)
  LocalDateTime timestamp

  @Column(columnDefinition="TEXT")
  String description       // Nullable human-readable detail, e.g. "Deployed payment-service v1.4.2 to STAGING"
}
```

**Column notes:**
- `action` values are string constants, not an enum, to allow adding new audit events without a schema migration.
- `AuditLog` entries are **insert-only** — never updated or deleted via the application API.
- `AuditService.log()` is called within the `@Transactional` boundary of the calling service method; if the parent transaction rolls back, the audit entry is also rolled back (ensuring consistency).

**Indexes:** index on `username`; index on `action`; index on `timestamp`; composite index on `(username, timestamp)` for date-range queries per user.

---

### Entity Relationship Summary

```
users (1) ────────── (N) deployments
                          deployedBy = users.username (denormalized)

deployments (1) ─────────── (N) rollbacks
                          rollbacks.deployment_id → deployments.id  [ON DELETE RESTRICT]

audit_logs — standalone (no FK to users or deployments; username stored as string)
```

---

## 2. Service Interface Definitions

All service interfaces reside in `com.enterprise.dashboard.service`. Implementations are in the `impl` subpackage (e.g., `AuthServiceImpl`). Spring's `@Service` annotation is applied on the implementation class. All mutating operations are `@Transactional`; read-only operations carry `@Transactional(readOnly=true)`.

---

### 2.1 AuthService

```java
package com.enterprise.dashboard.service;

/**
 * Handles user registration, authentication, and token lifecycle.
 * All mutating operations are @Transactional.
 */
public interface AuthService {

    /**
     * Registers a new user account.
     *
     * <p>Steps:
     * <ol>
     *   <li>Validates that username and email are not already taken
     *       (throws {@link ConflictException} on duplicate).</li>
     *   <li>Hashes the plain-text password with BCrypt.</li>
     *   <li>Persists the new {@code User} entity.</li>
     *   <li>Generates access and refresh JWTs for the new user.</li>
     *   <li>Calls {@code AuditService.log(username, "USER_REGISTERED", ...)}.</li>
     * </ol>
     *
     * @param request validated registration payload
     * @return {@link AuthResponse} containing access/refresh tokens and user metadata
     * @throws ConflictException if username or email already exists
     */
    AuthResponse register(RegisterRequest request);

    /**
     * Authenticates a user and issues JWT tokens.
     *
     * <p>Steps:
     * <ol>
     *   <li>Loads the user by username (throws {@link UnauthorizedException} if not found).</li>
     *   <li>Verifies the BCrypt password match
     *       (throws {@link UnauthorizedException} on mismatch).</li>
     *   <li>On failure, logs {@code LOGIN_FAILURE} before throwing.</li>
     *   <li>On success, generates access and refresh JWTs.</li>
     *   <li>Logs {@code LOGIN_SUCCESS}.</li>
     * </ol>
     *
     * @param request login credentials
     * @return {@link AuthResponse} containing access/refresh tokens and user metadata
     * @throws UnauthorizedException if credentials are invalid
     */
    AuthResponse login(LoginRequest request);

    /**
     * Issues a new access token using a valid refresh token.
     *
     * <p>Validates the refresh token's signature and expiry. If valid, extracts the
     * subject (username), loads the user, and issues a fresh access token.
     * The refresh token itself is not rotated in v1.
     *
     * @param refreshToken the JWT refresh token string (without "Bearer " prefix)
     * @return {@link AuthResponse} containing the new access token and its expiry
     * @throws UnauthorizedException if the refresh token is expired, malformed, or invalid
     */
    AuthResponse refreshToken(String refreshToken);
}
```

---

### 2.2 DeploymentService

```java
package com.enterprise.dashboard.service;

/**
 * Core business logic for deployment lifecycle management.
 * Rollback operations are co-located here to keep the status machine in one place.
 */
public interface DeploymentService {

    /**
     * Creates a new deployment record with {@code PENDING} status.
     *
     * <p>The {@code deployedBy} field is set to the authenticated user's username.
     * Logs {@code DEPLOYMENT_CREATED} via {@link AuditService}.
     *
     * @param request   validated creation payload (projectName, version, environment, remarks)
     * @param username  authenticated user's username (extracted from JWT in controller)
     * @return {@link DeploymentResponse} for the newly created deployment
     * @throws ValidationException if the environment or version is invalid
     */
    DeploymentResponse createDeployment(CreateDeploymentRequest request, String username);

    /**
     * Returns a paginated, optionally filtered list of deployments.
     *
     * <p>All filter fields in {@link DeploymentFilterRequest} are nullable; omitting a
     * field means no filter is applied for that dimension. Filtering is implemented via
     * JPA {@code Specification} to avoid combinatorial query method explosion.
     *
     * @param filter   optional filter criteria (environment, status, projectName, date range)
     * @param pageable Spring {@link Pageable} (page, size, sort)
     * @return page of {@link DeploymentResponse}
     */
    Page<DeploymentResponse> getDeployments(DeploymentFilterRequest filter, Pageable pageable);

    /**
     * Retrieves a single deployment by its primary key.
     *
     * @param id deployment primary key
     * @return {@link DeploymentResponse}
     * @throws ResourceNotFoundException if no deployment exists with the given id
     */
    DeploymentResponse getDeploymentById(Long id);

    /**
     * Updates the mutable fields of a deployment (currently only {@code remarks}).
     *
     * <p>Status, version, environment, and deployedBy are immutable after creation.
     * Logs {@code DEPLOYMENT_UPDATED}.
     *
     * @param id      deployment primary key
     * @param request payload containing updated remarks (nullable to clear)
     * @return {@link DeploymentResponse} reflecting the update
     * @throws ResourceNotFoundException if no deployment exists with the given id
     */
    DeploymentResponse updateDeployment(Long id, UpdateDeploymentRequest request);

    /**
     * Transitions a deployment to a new status, enforcing the state machine rules.
     *
     * <p>Valid transitions:
     * <ul>
     *   <li>{@code PENDING} → {@code BUILDING}</li>
     *   <li>{@code BUILDING} → {@code TESTING}</li>
     *   <li>{@code TESTING} → {@code DEPLOYING}</li>
     *   <li>{@code DEPLOYING} → {@code SUCCESSFUL}</li>
     *   <li>Any of {@code PENDING/BUILDING/TESTING/DEPLOYING} → {@code FAILED}</li>
     *   <li>{@code FAILED} → {@code ROLLED_BACK} (via triggerRollback only)</li>
     * </ul>
     *
     * <p>When transitioning to a terminal state ({@code SUCCESSFUL}, {@code FAILED},
     * {@code ROLLED_BACK}), sets {@code endTime} to the current UTC timestamp.
     * Logs {@code DEPLOYMENT_STATUS_UPDATED}.
     *
     * @param id        deployment primary key
     * @param newStatus target status
     * @param username  actor's username (for audit log)
     * @return {@link DeploymentResponse} with updated status
     * @throws ResourceNotFoundException          if deployment not found
     * @throws InvalidStatusTransitionException   if the transition is not permitted
     */
    DeploymentResponse updateStatus(Long id, DeploymentStatus newStatus, String username);

    /**
     * Permanently deletes a deployment record. Restricted to ADMIN role (enforced at
     * the controller level via {@code @PreAuthorize}).
     *
     * @param id deployment primary key
     * @throws ResourceNotFoundException if no deployment exists with the given id
     * @throws ConflictException         if one or more {@link Rollback} records reference this deployment
     */
    void deleteDeployment(Long id);

    /**
     * Triggers a rollback for a deployment in {@code FAILED} status.
     *
     * <p>Steps:
     * <ol>
     *   <li>Loads the deployment; throws {@link ResourceNotFoundException} if absent.</li>
     *   <li>Validates that current status is {@code FAILED}
     *       (throws {@link InvalidStatusTransitionException} otherwise).</li>
     *   <li>Creates a {@link Rollback} record with the deployment's current version
     *       as {@code previousVersion}.</li>
     *   <li>Transitions the deployment status to {@code ROLLED_BACK} and sets
     *       {@code endTime}.</li>
     *   <li>Logs {@code ROLLBACK_TRIGGERED}.</li>
     * </ol>
     *
     * @param deploymentId deployment primary key
     * @param rollbackReason operator-supplied justification
     * @param username       actor's username
     * @return {@link RollbackResponse} for the new rollback record
     * @throws ResourceNotFoundException        if deployment not found
     * @throws InvalidStatusTransitionException if deployment is not in {@code FAILED} status
     */
    RollbackResponse triggerRollback(Long deploymentId, String rollbackReason, String username);

    /**
     * Returns the paginated rollback history for a specific deployment.
     *
     * @param deploymentId deployment primary key
     * @param pageable     Spring {@link Pageable}
     * @return page of {@link RollbackResponse} ordered by {@code rollbackTime} desc
     * @throws ResourceNotFoundException if no deployment exists with the given id
     */
    Page<RollbackResponse> getRollbackHistory(Long deploymentId, Pageable pageable);
}
```

---

### 2.3 AuditService

```java
package com.enterprise.dashboard.service;

/**
 * Provides append-only audit trail operations.
 */
public interface AuditService {

    /**
     * Creates an {@link AuditLog} entry in the current transaction.
     *
     * <p>This method MUST be called within an active {@code @Transactional} boundary
     * so that audit entries are rolled back if the parent operation fails. The caller
     * (e.g., {@link DeploymentService}) is responsible for providing the transaction.
     *
     * @param username    actor performing the action (use "SYSTEM" for automated events)
     * @param action      constant string identifying the event (e.g. "DEPLOYMENT_CREATED")
     * @param description nullable human-readable description providing context
     */
    void log(String username, String action, String description);

    /**
     * Returns a paginated, optionally filtered view of audit logs. Restricted to ADMIN
     * role (enforced at the controller level).
     *
     * <p>All filter fields are nullable. When all are null, returns all audit logs
     * ordered by {@code timestamp} descending. Uses JPA {@code Specification} for
     * dynamic filtering.
     *
     * @param filter   optional filter criteria (username, action, date range)
     * @param pageable Spring {@link Pageable}
     * @return page of {@link AuditLogResponse}
     */
    Page<AuditLogResponse> getAuditLogs(AuditLogFilterRequest filter, Pageable pageable);
}
```

---

### 2.4 UserService

```java
package com.enterprise.dashboard.service;

/**
 * Manages user accounts. All endpoints require ADMIN role.
 */
public interface UserService {

    /**
     * Returns a paginated list of all users. Passwords are never included.
     *
     * @param pageable Spring {@link Pageable}
     * @return page of {@link UserResponse} (no password field)
     */
    Page<UserResponse> getUsers(Pageable pageable);

    /**
     * Retrieves a single user by their primary key.
     *
     * @param id user primary key
     * @return {@link UserResponse}
     * @throws ResourceNotFoundException if no user exists with the given id
     */
    UserResponse getUserById(Long id);

    /**
     * Updates the role of an existing user.
     *
     * <p>Logs {@code USER_ROLE_CHANGED} with the old and new role values in the
     * description for audit trail completeness.
     *
     * @param id      user primary key
     * @param newRole the target role to assign
     * @return {@link UserResponse} reflecting the updated role
     * @throws ResourceNotFoundException if no user exists with the given id
     */
    UserResponse updateRole(Long id, Role newRole);

    /**
     * Permanently deletes a user account.
     *
     * <p>Logs {@code USER_DELETED} before deletion. Note: deployments referencing this
     * user's username via the denormalized {@code deployedBy} field are unaffected.
     *
     * @param id user primary key
     * @throws ResourceNotFoundException if no user exists with the given id
     */
    void deleteUser(Long id);
}
```

---

### 2.5 DashboardService

```java
package com.enterprise.dashboard.service;

/**
 * Provides aggregated deployment metrics for the dashboard.
 * All methods are read-only and suitable for caching.
 */
public interface DashboardService {

    /**
     * Computes a full statistics snapshot.
     *
     * <p>Executes the following count queries against {@link DeploymentRepository}:
     * <ul>
     *   <li>{@code countByStatus(PENDING)}</li>
     *   <li>{@code countByStatus(SUCCESSFUL)}</li>
     *   <li>{@code countByStatus(FAILED)}</li>
     *   <li>{@code countByStatus(ROLLED_BACK)}</li>
     *   <li>Total: sum of all statuses (single {@code count()} call)</li>
     * </ul>
     * Also populates {@code recentDeployments} from {@link #getRecentDeployments(int)}
     * with a limit of 5.
     *
     * @return {@link DashboardStatsResponse} with all metrics populated
     */
    DashboardStatsResponse getStats();

    /**
     * Calculates the deployment success rate as a percentage.
     *
     * <p>Formula: {@code (successfulDeployments / totalEvaluated) * 100.0}, where
     * {@code totalEvaluated} is the count of all deployments that have left the
     * {@code PENDING} state (i.e., total minus PENDING). Returns {@code 0.0} if
     * {@code totalEvaluated} is zero to avoid division by zero.
     *
     * @return success rate as a {@code double} in the range [0.0, 100.0]
     */
    double getSuccessRate();

    /**
     * Returns the most recent deployments ordered by start time descending.
     *
     * @param limit maximum number of results to return (typically 5 or 10)
     * @return list of {@link DeploymentResponse}, length at most {@code limit}
     */
    List<DeploymentResponse> getRecentDeployments(int limit);
}
```

---

## 3. DTO Specifications

All DTOs reside in `com.enterprise.dashboard.model.dto`. Request DTOs carry Jakarta Validation annotations (`@NotBlank`, `@NotNull`, `@Size`, `@Email`). Response DTOs are plain POJOs with no validation annotations.

---

### 3.1 Request DTOs

#### RegisterRequest

```java
public class RegisterRequest {
    @NotBlank
    @Size(min = 3, max = 50)
    private String username;

    @NotBlank
    @Email
    private String email;

    @NotBlank
    @Size(min = 8, max = 100)
    private String password;

    @NotNull
    private Role role;
}
```

#### LoginRequest

```java
public class LoginRequest {
    @NotBlank
    private String username;

    @NotBlank
    private String password;
}
```

#### CreateDeploymentRequest

```java
public class CreateDeploymentRequest {
    @NotBlank
    @Size(max = 100)
    private String projectName;

    @NotBlank
    @Size(max = 50)
    private String version;

    @NotNull
    private Environment environment;

    private String remarks;   // nullable
}
```

#### UpdateDeploymentRequest

```java
public class UpdateDeploymentRequest {
    private String remarks;   // nullable — pass null to clear, pass value to update
}
```

#### UpdateStatusRequest

```java
public class UpdateStatusRequest {
    @NotNull
    private DeploymentStatus status;
}
```

#### RollbackRequest

```java
public class RollbackRequest {
    @NotBlank
    private String rollbackReason;
}
```

#### UpdateRoleRequest

```java
public class UpdateRoleRequest {
    @NotNull
    private Role role;
}
```

#### DeploymentFilterRequest (query params — no validation annotations)

```java
public class DeploymentFilterRequest {
    private Environment environment;   // nullable
    private DeploymentStatus status;   // nullable
    private String projectName;        // nullable — partial match
    private LocalDate startDate;       // nullable — inclusive lower bound on startTime
    private LocalDate endDate;         // nullable — inclusive upper bound on startTime
}
```

#### AuditLogFilterRequest (query params — no validation annotations)

```java
public class AuditLogFilterRequest {
    private String username;           // nullable — exact match
    private String action;             // nullable — exact match
    private LocalDateTime startDate;   // nullable — inclusive lower bound on timestamp
    private LocalDateTime endDate;     // nullable — inclusive upper bound on timestamp
}
```

---

### 3.2 Response DTOs

#### AuthResponse

```java
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    private String tokenType = "Bearer";
    private long expiresIn;       // seconds — 86400 for access token
    private String username;
    private Role role;
}
```

#### DeploymentResponse

```java
public class DeploymentResponse {
    private Long id;
    private String projectName;
    private String version;
    private Environment environment;
    private DeploymentStatus status;
    private String deployedBy;
    private LocalDateTime startTime;
    private LocalDateTime endTime;    // null if not yet in terminal state
    private String remarks;           // null if not set
    private Long durationSeconds;     // computed: endTime - startTime in seconds;
                                      // null if either field is null
}
```

**Computed field note:** `durationSeconds` is populated in the service/mapper layer using `ChronoUnit.SECONDS.between(startTime, endTime)`. It is not stored in the database.

#### RollbackResponse

```java
public class RollbackResponse {
    private Long id;
    private Long deploymentId;
    private String previousVersion;
    private String rollbackReason;
    private LocalDateTime rollbackTime;
}
```

#### UserResponse

```java
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private Role role;
    private LocalDateTime createdAt;
    // NOTE: no password field — never included in any response
}
```

#### DashboardStatsResponse

```java
public class DashboardStatsResponse {
    private long totalDeployments;
    private long successfulDeployments;
    private long failedDeployments;
    private long pendingDeployments;
    private long rolledBackDeployments;
    private double successRate;                       // 0.0 to 100.0
    private List<DeploymentResponse> recentDeployments; // last 5
}
```

#### AuditLogResponse

```java
public class AuditLogResponse {
    private Long id;
    private String username;
    private String action;
    private LocalDateTime timestamp;
    private String description;   // null if not recorded
}
```

---

### 3.3 Universal Response Wrapper

```java
// com.enterprise.dashboard.model.dto.ApiResponse<T>
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private LocalDateTime timestamp;
    private List<ErrorDetail> errors;   // null on success

    // Static factory helpers
    public static <T> ApiResponse<T> success(String message, T data) { ... }
    public static <T> ApiResponse<T> error(String message, List<ErrorDetail> errors) { ... }
}

public class ErrorDetail {
    private String field;    // null for non-field-level errors
    private String message;
}
```

---

## 4. Repository Method Definitions

All repositories extend `JpaRepository<Entity, Long>` and reside in `com.enterprise.dashboard.repository`. The standard CRUD methods (`save`, `findById`, `findAll`, `deleteById`, `count`, `existsById`) are inherited and not re-listed. Only custom methods are documented below.

---

### 4.1 UserRepository

```java
public interface UserRepository extends JpaRepository<User, Long> {

    /** Used by JwtAuthenticationFilter and AuthService to load user by login identifier. */
    Optional<User> findByUsername(String username);

    /** Used during registration to check for duplicate email before persisting. */
    Optional<User> findByEmail(String email);

    /** Fast existence check — avoids loading the full entity. */
    boolean existsByUsername(String username);

    /** Fast existence check — avoids loading the full entity. */
    boolean existsByEmail(String email);
}
```

---

### 4.2 DeploymentRepository

```java
public interface DeploymentRepository
        extends JpaRepository<Deployment, Long>, JpaSpecificationExecutor<Deployment> {

    /** Single-field paginated queries — used as fallback when only one filter is active. */
    Page<Deployment> findByEnvironment(Environment environment, Pageable pageable);

    Page<Deployment> findByStatus(DeploymentStatus status, Pageable pageable);

    Page<Deployment> findByProjectName(String projectName, Pageable pageable);

    Page<Deployment> findByDeployedBy(String username, Pageable pageable);

    /** Two-field combo query — most common dashboard filter (environment + status). */
    Page<Deployment> findByEnvironmentAndStatus(
            Environment environment,
            DeploymentStatus status,
            Pageable pageable);

    /**
     * Count deployments in a specific status — used by DashboardService for stats.
     * Runs a single COUNT(*) WHERE status = ? query.
     */
    long countByStatus(DeploymentStatus status);

    /**
     * Returns the 5 most recent deployments ordered by start time descending.
     * Used by DashboardService.getStats() and DashboardService.getRecentDeployments(5).
     */
    List<Deployment> findTop5ByOrderByStartTimeDesc();

    /**
     * Returns the N most recent deployments for the /dashboard/recent endpoint.
     * Spring Data derives the query; the controller passes limit=10.
     */
    List<Deployment> findTop10ByOrderByStartTimeDesc();

    /**
     * Dynamic multi-field filtering via Specification.
     *
     * This is the primary query method used by DeploymentService.getDeployments().
     * DeploymentSpecification builds predicates from DeploymentFilterRequest fields:
     *   - environment: exact match
     *   - status: exact match
     *   - projectName: LIKE '%value%' (case-insensitive)
     *   - startDate: startTime >= startDate (start of day)
     *   - endDate:   startTime <= endDate   (end of day)
     *
     * Inherited from JpaSpecificationExecutor<Deployment>:
     *   Page<Deployment> findAll(Specification<Deployment> spec, Pageable pageable);
     */
}
```

---

### 4.3 RollbackRepository

```java
public interface RollbackRepository extends JpaRepository<Rollback, Long> {

    /** Returns paginated rollback history for a deployment (newest first via Pageable). */
    Page<Rollback> findByDeploymentId(Long deploymentId, Pageable pageable);

    /**
     * Existence check used by DeploymentService.deleteDeployment().
     * If true, deletion is blocked and ConflictException is thrown.
     */
    boolean existsByDeploymentId(Long deploymentId);

    /**
     * Returns all rollbacks for a deployment ordered by most recent first.
     * Used for detail views where a full list (not paginated) is acceptable.
     */
    List<Rollback> findByDeploymentIdOrderByRollbackTimeDesc(Long deploymentId);
}
```

---

### 4.4 AuditLogRepository

```java
public interface AuditLogRepository
        extends JpaRepository<AuditLog, Long>, JpaSpecificationExecutor<AuditLog> {

    /** Single-field query — used when only username filter is provided. */
    Page<AuditLog> findByUsername(String username, Pageable pageable);

    /** Single-field query — used when only action filter is provided. */
    Page<AuditLog> findByAction(String action, Pageable pageable);

    /**
     * Username + date-range query — used when username and both date bounds are provided.
     * Timestamp bounds are inclusive.
     */
    Page<AuditLog> findByUsernameAndTimestampBetween(
            String username,
            LocalDateTime start,
            LocalDateTime end,
            Pageable pageable);

    /**
     * Dynamic multi-field filtering via Specification.
     *
     * AuditLogSpecification builds predicates from AuditLogFilterRequest fields:
     *   - username:   exact match (case-sensitive)
     *   - action:     exact match
     *   - startDate:  timestamp >= startDate
     *   - endDate:    timestamp <= endDate
     *
     * Inherited from JpaSpecificationExecutor<AuditLog>:
     *   Page<AuditLog> findAll(Specification<AuditLog> spec, Pageable pageable);
     */
}
```

---

## 5. Exception Hierarchy

All application exceptions reside in `com.enterprise.dashboard.exception`. They extend `AppException`, which is an abstract base class carrying the HTTP status code. The `GlobalExceptionHandler` maps each exception type to the appropriate HTTP response.

```
java.lang.RuntimeException
└── AppException                       (abstract; fields: HttpStatus status, String message)
    ├── ResourceNotFoundException       HTTP 404 — entity not found by ID or identifier
    ├── UnauthorizedException           HTTP 401 — invalid/expired credentials or JWT
    ├── ForbiddenException              HTTP 403 — valid token but role lacks permission
    ├── ConflictException               HTTP 409 — unique constraint or FK constraint violation
    ├── ValidationException             HTTP 400 — business rule violation (distinct from Bean Validation)
    └── InvalidStatusTransitionException HTTP 422 — illegal deployment status transition
```

### AppException (abstract base)

```java
public abstract class AppException extends RuntimeException {
    private final HttpStatus status;

    protected AppException(HttpStatus status, String message) {
        super(message);
        this.status = status;
    }

    public HttpStatus getStatus() { return status; }
}
```

### ResourceNotFoundException

```java
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resourceType, Long id) {
        super(HttpStatus.NOT_FOUND,
              String.format("Resource not found: %s with id %d", resourceType, id));
    }
}
```

### UnauthorizedException

```java
public class UnauthorizedException extends AppException {
    public UnauthorizedException(String message) {
        super(HttpStatus.UNAUTHORIZED, message);
    }
}
```

### ForbiddenException

```java
public class ForbiddenException extends AppException {
    public ForbiddenException(String message) {
        super(HttpStatus.FORBIDDEN, message);
    }
}
```

### ConflictException

```java
public class ConflictException extends AppException {
    public ConflictException(String message) {
        super(HttpStatus.CONFLICT, message);
    }
}
```

### ValidationException

```java
public class ValidationException extends AppException {
    public ValidationException(String message) {
        super(HttpStatus.BAD_REQUEST, message);
    }
}
```

### InvalidStatusTransitionException

```java
public class InvalidStatusTransitionException extends AppException {
    public InvalidStatusTransitionException(DeploymentStatus from, DeploymentStatus to) {
        super(HttpStatus.UNPROCESSABLE_ENTITY,
              String.format("Cannot transition deployment from %s to %s", from, to));
    }
}
```

---

## 6. GlobalExceptionHandler Mapping Table

`GlobalExceptionHandler` is annotated with `@RestControllerAdvice` and resides in `com.enterprise.dashboard.exception`. Every handler method returns `ResponseEntity<ApiResponse<Void>>`.

| Exception Type | HTTP Status | Response Message Pattern |
|---|---|---|
| `MethodArgumentNotValidException` | 400 | `"Validation failed"` — `errors` array populated with `{field, message}` pairs from `BindingResult` |
| `ConstraintViolationException` | 400 | `"Validation failed"` — `errors` array populated from constraint violations |
| `ValidationException` | 400 | `exception.getMessage()` |
| `InvalidStatusTransitionException` | 422 | `"Cannot transition deployment from {from} to {to}"` |
| `ResourceNotFoundException` | 404 | `"Resource not found: {type} with id {id}"` |
| `UnauthorizedException` | 401 | `"Authentication required or credentials invalid"` |
| `AccessDeniedException` (Spring Security) | 403 | `"Insufficient permissions for this operation"` |
| `ForbiddenException` | 403 | `exception.getMessage()` |
| `ConflictException` | 409 | `exception.getMessage()` |
| `DataIntegrityViolationException` | 409 | `"Resource already exists or constraint violation"` |
| `Exception` (catch-all) | 500 | `"An unexpected error occurred. Reference: {UUID}"` — UUID written to error log for correlation |

**Handler structure example:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(
            MethodArgumentNotValidException ex) {
        List<ErrorDetail> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new ErrorDetail(fe.getField(), fe.getDefaultMessage()))
            .collect(Collectors.toList());
        return ResponseEntity.badRequest()
            .body(ApiResponse.error("Validation failed", errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        String ref = UUID.randomUUID().toString();
        log.error("Unhandled exception [ref={}]", ref, ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error(
                "An unexpected error occurred. Reference: " + ref, null));
    }
}
```

---

## 7. AuditLog Trigger Points

`AuditService.log()` is called at the end of each successful mutating operation, within the same `@Transactional` boundary. On failure, the transaction rolls back and no audit entry is persisted.

| Audit Action Constant | Description | Triggered By (Service Method) |
|---|---|---|
| `LOGIN_SUCCESS` | User authenticated successfully | `AuthService.login()` — called after JWT generation |
| `LOGIN_FAILURE` | Authentication attempt failed | `AuthService.login()` — called before throwing `UnauthorizedException` |
| `USER_REGISTERED` | New user account created | `AuthService.register()` — called after user persisted |
| `DEPLOYMENT_CREATED` | New deployment record created | `DeploymentService.createDeployment()` — called after entity saved |
| `DEPLOYMENT_UPDATED` | Deployment remarks updated | `DeploymentService.updateDeployment()` — called after field update saved |
| `DEPLOYMENT_STATUS_UPDATED` | Deployment status transitioned | `DeploymentService.updateStatus()` — description includes `{from} → {to}` |
| `ROLLBACK_TRIGGERED` | Rollback initiated for a failed deployment | `DeploymentService.triggerRollback()` — description includes deployment ID and reason |
| `USER_ROLE_CHANGED` | User's role was updated by an Admin | `UserService.updateRole()` — description includes old role and new role |
| `USER_DELETED` | User account removed by an Admin | `UserService.deleteUser()` — description includes deleted username |

**Action constant class:**

```java
// com.enterprise.dashboard.audit.AuditActions
public final class AuditActions {
    public static final String LOGIN_SUCCESS             = "LOGIN_SUCCESS";
    public static final String LOGIN_FAILURE             = "LOGIN_FAILURE";
    public static final String USER_REGISTERED           = "USER_REGISTERED";
    public static final String DEPLOYMENT_CREATED        = "DEPLOYMENT_CREATED";
    public static final String DEPLOYMENT_UPDATED        = "DEPLOYMENT_UPDATED";
    public static final String DEPLOYMENT_STATUS_UPDATED = "DEPLOYMENT_STATUS_UPDATED";
    public static final String ROLLBACK_TRIGGERED        = "ROLLBACK_TRIGGERED";
    public static final String USER_ROLE_CHANGED         = "USER_ROLE_CHANGED";
    public static final String USER_DELETED              = "USER_DELETED";

    private AuditActions() {}
}
```

**Description format conventions:**

| Action | Description Template |
|---|---|
| `LOGIN_SUCCESS` | `"User {username} logged in from session"` |
| `LOGIN_FAILURE` | `"Failed login attempt for username: {username}"` |
| `USER_REGISTERED` | `"New user registered: {username} with role {role}"` |
| `DEPLOYMENT_CREATED` | `"Deployment created: {projectName} v{version} to {environment}"` |
| `DEPLOYMENT_UPDATED` | `"Deployment {id} remarks updated"` |
| `DEPLOYMENT_STATUS_UPDATED` | `"Deployment {id} status changed from {oldStatus} to {newStatus}"` |
| `ROLLBACK_TRIGGERED` | `"Rollback triggered for deployment {id}. Reason: {rollbackReason}"` |
| `USER_ROLE_CHANGED` | `"User {username} role changed from {oldRole} to {newRole}"` |
| `USER_DELETED` | `"User account deleted: {username}"` |
