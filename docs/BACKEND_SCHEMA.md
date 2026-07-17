# Backend Schema

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Status:** Approved

This document is the single source of truth for the database schema, JPA entity mappings, service interfaces, DTOs, and the complete REST API contract. It is the primary reference for backend implementation.

---

## 1. Domain Entities

### 1.1 Entity Relationship

```
┌─────────────────────────┐
│          users          │
│  id            PK       │
│  username      UNIQUE   │
│  email         UNIQUE   │
│  password      (BCrypt) │
│  role                   │
│  created_at             │
└─────────────────────────┘
         │
         │ deployed_by = username (string copy, NOT FK)
         ▼
┌─────────────────────────┐
│       deployments       │
│  id            PK       │
│  project_name           │
│  version                │
│  environment            │
│  status                 │
│  deployed_by  VARCHAR   │
│  start_time             │
│  end_time     nullable  │
│  remarks      nullable  │
└─────────┬───────────────┘
          │ 1:N (ON DELETE RESTRICT)
          ▼
┌──────────────────┐
│    rollbacks     │
│  id        PK   │
│  deployment_id  FK│
│  previous_version│
│  rollback_reason │
│  rollback_time   │
└──────────────────┘

audit_logs — standalone (username stored as VARCHAR, NOT FK)
┌──────────────────────┐
│      audit_logs      │
│  id          PK      │
│  username   VARCHAR  │
│  action     VARCHAR  │
│  timestamp           │
│  description TEXT    │
└──────────────────────┘
```

**Key design decisions:**
- `deployments.deployed_by` and `audit_logs.username` are denormalized VARCHAR copies — not FKs — so records survive user deletion
- `rollbacks.deployment_id` uses `ON DELETE RESTRICT` to protect rollback history from being orphaned
- Audit log is **insert-only** at the application layer — no UPDATE or DELETE methods exposed

---

## 2. SQL Table Definitions

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
  id          BIGSERIAL    PRIMARY KEY,
  username    VARCHAR(50)  NOT NULL UNIQUE,
  email       VARCHAR(100) NOT NULL UNIQUE,
  password    VARCHAR(255) NOT NULL,
  role        VARCHAR(20)  NOT NULL
              CHECK (role IN ('ADMIN', 'DEVOPS_ENGINEER', 'DEVELOPER')),
  created_at  TIMESTAMP    NOT NULL DEFAULT NOW()
);

-- V2__create_deployments_table.sql
CREATE TABLE deployments (
  id            BIGSERIAL    PRIMARY KEY,
  project_name  VARCHAR(100) NOT NULL,
  version       VARCHAR(50)  NOT NULL,
  environment   VARCHAR(20)  NOT NULL
                CHECK (environment IN ('DEV', 'QA', 'STAGING', 'PRODUCTION')),
  status        VARCHAR(30)  NOT NULL DEFAULT 'PENDING'
                CHECK (status IN ('PENDING', 'BUILDING', 'TESTING', 'DEPLOYING',
                                  'SUCCESSFUL', 'FAILED', 'ROLLED_BACK')),
  deployed_by   VARCHAR(50)  NOT NULL,
  start_time    TIMESTAMP    NOT NULL DEFAULT NOW(),
  end_time      TIMESTAMP,
  remarks       TEXT
);

-- V3__create_rollbacks_table.sql
CREATE TABLE rollbacks (
  id               BIGSERIAL    PRIMARY KEY,
  deployment_id    BIGINT       NOT NULL
                   REFERENCES deployments(id) ON DELETE RESTRICT,
  previous_version VARCHAR(50)  NOT NULL,
  rollback_reason  TEXT         NOT NULL,
  rollback_time    TIMESTAMP    NOT NULL DEFAULT NOW()
);

-- V4__create_audit_logs_table.sql
CREATE TABLE audit_logs (
  id          BIGSERIAL   PRIMARY KEY,
  username    VARCHAR(50) NOT NULL,
  action      VARCHAR(50) NOT NULL,
  timestamp   TIMESTAMP   NOT NULL DEFAULT NOW(),
  description TEXT
);

-- V5__create_indexes.sql
CREATE INDEX idx_deployments_status       ON deployments(status);
CREATE INDEX idx_deployments_env_status   ON deployments(environment, status);
CREATE INDEX idx_deployments_deployed_by  ON deployments(deployed_by);
CREATE INDEX idx_deployments_start_time   ON deployments(start_time DESC);
CREATE INDEX idx_rollbacks_deployment_id  ON rollbacks(deployment_id);
CREATE INDEX idx_audit_logs_username_ts   ON audit_logs(username, timestamp DESC);
CREATE INDEX idx_audit_logs_action        ON audit_logs(action);
```

### Flyway Migration Files

| File | Purpose |
|------|---------|
| `V1__create_users_table.sql` | Creates `users` |
| `V2__create_deployments_table.sql` | Creates `deployments` |
| `V3__create_rollbacks_table.sql` | Creates `rollbacks` with FK |
| `V4__create_audit_logs_table.sql` | Creates `audit_logs` |
| `V5__create_indexes.sql` | Creates all performance indexes |
| `V6__seed_initial_data.sql` | Seeds admin user + sample data |

**Rules:** Never modify an existing migration file — Flyway checksums them. Always add a new `VN+1__` file for schema changes. Use `IF NOT EXISTS` and `ON CONFLICT DO NOTHING` for idempotency.

---

## 3. JPA Entity Classes

All entities live in `com.enterprise.dashboard.model.entity`.

### User

```java
@Entity @Table(name = "users")
class User {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id;

  @Column(nullable=false, unique=true, length=50)
  String username;

  @Column(nullable=false, unique=true, length=100)
  String email;

  @Column(nullable=false)
  @JsonIgnore
  String password; // BCrypt hash — NEVER serialized

  @Enumerated(EnumType.STRING)
  @Column(nullable=false, length=20)
  Role role;

  @CreationTimestamp
  @Column(name="created_at", updatable=false)
  LocalDateTime createdAt;
}
```

### Deployment

```java
@Entity @Table(name = "deployments")
class Deployment {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id;

  @Column(name="project_name", nullable=false, length=100)
  String projectName;

  @Column(nullable=false, length=50)
  String version;

  @Enumerated(EnumType.STRING)
  @Column(nullable=false, length=20)
  Environment environment; // DEV | QA | STAGING | PRODUCTION

  @Enumerated(EnumType.STRING)
  @Column(nullable=false, length=30)
  DeploymentStatus status; // PENDING default

  @Column(name="deployed_by", nullable=false, length=50)
  String deployedBy; // denormalized — NOT a FK

  @CreationTimestamp
  @Column(name="start_time", updatable=false)
  LocalDateTime startTime;

  @Column(name="end_time")
  LocalDateTime endTime; // null until terminal state; set by service layer

  @Column(columnDefinition="TEXT")
  String remarks; // nullable
}
```

**Notes:**
- `endTime` is set explicitly by `DeploymentService.updateStatus()` when transitioning to `SUCCESSFUL`, `FAILED`, or `ROLLED_BACK` — NOT via `@UpdateTimestamp`
- `deployedBy` survives user deletion because it has no FK constraint

### Rollback

```java
@Entity @Table(name = "rollbacks")
class Rollback {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id;

  @ManyToOne(fetch=FetchType.LAZY)
  @JoinColumn(name="deployment_id", nullable=false)
  Deployment deployment; // LAZY to avoid N+1

  @Column(name="previous_version", nullable=false, length=50)
  String previousVersion; // copied from deployment.version at rollback time

  @Column(name="rollback_reason", nullable=false, columnDefinition="TEXT")
  String rollbackReason;

  @CreationTimestamp
  @Column(name="rollback_time", updatable=false)
  LocalDateTime rollbackTime;
}
```

### AuditLog

```java
@Entity @Table(name = "audit_logs")
class AuditLog {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id;

  @Column(nullable=false, length=50)
  String username; // may be "SYSTEM" for automated events

  @Column(nullable=false, length=50)
  String action; // constant string (see AuditActions class)

  @CreationTimestamp
  @Column(name="timestamp", updatable=false)
  LocalDateTime timestamp;

  @Column(columnDefinition="TEXT")
  String description; // nullable human-readable detail
}
```

### JPA Notes

- All enums use `@Enumerated(EnumType.STRING)` — never `ORDINAL` (ordinal shifts corrupt data on enum reordering)
- `@CreationTimestamp` sets insert-time fields; they are `updatable=false`
- `ddl-auto: validate` in production — Flyway owns all schema changes

---

## 4. Enumerations

Package: `com.enterprise.dashboard.model.enums`

```java
public enum Role { ADMIN, DEVOPS_ENGINEER, DEVELOPER }

public enum Environment { DEV, QA, STAGING, PRODUCTION }

public enum DeploymentStatus { PENDING, BUILDING, TESTING, DEPLOYING, SUCCESSFUL, FAILED, ROLLED_BACK }
```

### Valid Status Transitions

| From | Permitted To |
|------|-------------|
| PENDING | BUILDING, FAILED |
| BUILDING | TESTING, FAILED |
| TESTING | DEPLOYING, FAILED |
| DEPLOYING | SUCCESSFUL, FAILED |
| FAILED | ROLLED_BACK (via rollback endpoint only) |
| SUCCESSFUL / ROLLED_BACK | — (terminal, no further transitions) |

---

## 5. DTOs

All DTOs live in `com.enterprise.dashboard.model.dto`.

### Request DTOs

```java
// RegisterRequest
class RegisterRequest {
  @NotBlank @Size(min=3, max=50) String username;
  @NotBlank @Email               String email;
  @NotBlank @Size(min=8, max=100) String password;
  @NotNull                       Role role;
}

// LoginRequest
class LoginRequest {
  @NotBlank String username;
  @NotBlank String password;
}

// RefreshTokenRequest
class RefreshTokenRequest {
  @NotBlank String refreshToken;
}

// CreateDeploymentRequest
class CreateDeploymentRequest {
  @NotBlank @Size(max=100) String projectName;
  @NotBlank @Size(max=50)  String version;
  @NotNull                 Environment environment;
  String remarks; // nullable
}

// UpdateDeploymentRequest
class UpdateDeploymentRequest {
  String remarks; // nullable — pass null to clear
}

// UpdateStatusRequest
class UpdateStatusRequest {
  @NotNull DeploymentStatus status;
}

// RollbackRequest
class RollbackRequest {
  @NotBlank String rollbackReason;
}

// UpdateRoleRequest
class UpdateRoleRequest {
  @NotNull Role role;
}

// DeploymentFilterRequest (query params)
class DeploymentFilterRequest {
  Environment environment;   // nullable
  DeploymentStatus status;   // nullable
  String projectName;        // nullable — LIKE '%value%'
  LocalDate startDate;       // nullable — inclusive lower bound
  LocalDate endDate;         // nullable — inclusive upper bound
}

// AuditLogFilterRequest (query params)
class AuditLogFilterRequest {
  String username;           // nullable — exact match
  String action;             // nullable — exact match
  @DateTimeFormat(iso=ISO.DATE_TIME) LocalDateTime startDate;
  @DateTimeFormat(iso=ISO.DATE_TIME) LocalDateTime endDate;
}
```

### Response DTOs

```java
// AuthResponse
class AuthResponse {
  String accessToken;
  String refreshToken;
  String tokenType = "Bearer";
  long expiresIn;     // seconds
  String username;
  Role role;
}

// DeploymentResponse
class DeploymentResponse {
  Long id;
  String projectName;
  String version;
  Environment environment;
  DeploymentStatus status;
  String deployedBy;
  LocalDateTime startTime;
  LocalDateTime endTime;      // null if not yet terminal
  String remarks;             // null if not set
  Long durationSeconds;       // computed: endTime - startTime; null if either is null
}

// RollbackResponse
class RollbackResponse {
  Long id;
  Long deploymentId;
  String previousVersion;
  String rollbackReason;
  LocalDateTime rollbackTime;
}

// UserResponse
class UserResponse {
  Long id;
  String username;
  String email;
  Role role;
  LocalDateTime createdAt;
  // NO password field — ever
}

// DashboardStatsResponse
class DashboardStatsResponse {
  long totalDeployments;
  long successfulDeployments;
  long failedDeployments;
  long pendingDeployments;
  long rolledBackDeployments;
  double successRate;                        // 0.0–100.0
  List<DeploymentResponse> recentDeployments; // last 5
}

// SuccessRateResponse
class SuccessRateResponse {
  double successRate;   // 0.0–100.0
  long totalEvaluated;  // non-PENDING deployments used in calculation
}

// AuditLogResponse
class AuditLogResponse {
  Long id;
  String username;
  String action;
  LocalDateTime timestamp;
  String description; // null if not recorded
}
```

### Universal Response Wrapper

```java
// ApiResponse<T> — every endpoint returns this
class ApiResponse<T> {
  boolean success;
  String message;
  T data;
  LocalDateTime timestamp;
  List<ErrorDetail> errors; // null on success

  static <T> ApiResponse<T> success(String message, T data) { ... }
  static <T> ApiResponse<T> error(String message, List<ErrorDetail> errors) { ... }
}

class ErrorDetail {
  String field;   // null for non-field errors
  String message;
}
```

---

## 6. Repository Interfaces

Package: `com.enterprise.dashboard.repository`

### UserRepository

```java
interface UserRepository extends JpaRepository<User, Long> {
  Optional<User> findByUsername(String username);
  Optional<User> findByEmail(String email);
  boolean existsByUsername(String username);
  boolean existsByEmail(String email);
}
```

### DeploymentRepository

```java
interface DeploymentRepository extends JpaRepository<Deployment, Long>, JpaSpecificationExecutor<Deployment> {
  Page<Deployment> findByEnvironment(Environment env, Pageable p);
  Page<Deployment> findByStatus(DeploymentStatus status, Pageable p);
  Page<Deployment> findByProjectName(String name, Pageable p);
  Page<Deployment> findByEnvironmentAndStatus(Environment env, DeploymentStatus status, Pageable p);
  long countByStatus(DeploymentStatus status);
  List<Deployment> findTop5ByOrderByStartTimeDesc();
  List<Deployment> findTop10ByOrderByStartTimeDesc();
  // Dynamic filter via Specification (from JpaSpecificationExecutor):
  // Page<Deployment> findAll(Specification<Deployment> spec, Pageable pageable)
}
```

### RollbackRepository

```java
interface RollbackRepository extends JpaRepository<Rollback, Long> {
  Page<Rollback> findByDeploymentId(Long deploymentId, Pageable p);
  boolean existsByDeploymentId(Long deploymentId); // used to block DELETE if rollbacks exist
  List<Rollback> findByDeploymentIdOrderByRollbackTimeDesc(Long deploymentId);
}
```

### AuditLogRepository

```java
interface AuditLogRepository extends JpaRepository<AuditLog, Long>, JpaSpecificationExecutor<AuditLog> {
  Page<AuditLog> findByUsername(String username, Pageable p);
  Page<AuditLog> findByAction(String action, Pageable p);
  Page<AuditLog> findByUsernameAndTimestampBetween(String username, LocalDateTime start, LocalDateTime end, Pageable p);
  // Dynamic filter via Specification (from JpaSpecificationExecutor)
}
```

---

## 7. Service Interfaces

Package: `com.enterprise.dashboard.service`; implementations in `impl` subpackage.

### AuthService

```java
interface AuthService {
  AuthResponse register(RegisterRequest request);   // throws ConflictException on duplicate username/email
  AuthResponse login(LoginRequest request);          // throws UnauthorizedException on bad credentials
  AuthResponse refreshToken(RefreshTokenRequest r); // throws UnauthorizedException on invalid/expired refresh token
}
```

### DeploymentService

```java
interface DeploymentService {
  DeploymentResponse createDeployment(CreateDeploymentRequest request, String username);
  Page<DeploymentResponse> getDeployments(DeploymentFilterRequest filter, Pageable pageable);
  DeploymentResponse getDeploymentById(Long id);                         // throws ResourceNotFoundException
  DeploymentResponse updateDeployment(Long id, UpdateDeploymentRequest r); // throws ResourceNotFoundException
  DeploymentResponse updateStatus(Long id, DeploymentStatus newStatus, String username);
    // throws ResourceNotFoundException, InvalidStatusTransitionException
  void deleteDeployment(Long id);
    // throws ResourceNotFoundException, ConflictException (if rollback records exist)
  RollbackResponse triggerRollback(Long deploymentId, String rollbackReason, String username);
    // throws ResourceNotFoundException, InvalidStatusTransitionException (if not FAILED)
  Page<RollbackResponse> getRollbackHistory(Long deploymentId, Pageable pageable);
}
```

### AuditService

```java
interface AuditService {
  void log(String username, String action, String description);
    // Must be called within @Transactional boundary; auth events use REQUIRES_NEW propagation
  Page<AuditLogResponse> getAuditLogs(AuditLogFilterRequest filter, Pageable pageable);
}
```

### UserService

```java
interface UserService {
  Page<UserResponse> getUsers(Pageable pageable);
  UserResponse getUserById(Long id);               // throws ResourceNotFoundException
  UserResponse updateRole(Long id, Role newRole);  // throws ResourceNotFoundException; logs USER_ROLE_CHANGED
  void deleteUser(Long id);                        // throws ResourceNotFoundException; logs USER_DELETED
}
```

### DashboardService

```java
interface DashboardService {
  DashboardStatsResponse getStats();
  SuccessRateResponse getSuccessRate();
  // Formula: (successfulDeployments / max(totalEvaluated, 1)) * 100
  // totalEvaluated = total minus PENDING; returns 0.0 if totalEvaluated == 0
  List<DeploymentResponse> getRecentDeployments(); // top 10 by startTime DESC
}
```

---

## 8. Audit Action Constants

Class: `com.enterprise.dashboard.audit.AuditActions`

```java
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
  public static final String DEPLOYMENT_DELETED        = "DEPLOYMENT_DELETED";
  private AuditActions() {}
}
```

### Description Templates

| Action | Description |
|--------|-------------|
| `LOGIN_SUCCESS` | `"User {username} logged in"` |
| `LOGIN_FAILURE` | `"Failed login attempt for username: {username}"` |
| `USER_REGISTERED` | `"New user registered: {username} with role {role}"` |
| `DEPLOYMENT_CREATED` | `"Deployment created: {projectName} v{version} to {environment}"` |
| `DEPLOYMENT_STATUS_UPDATED` | `"Deployment {id} status changed from {oldStatus} to {newStatus}"` |
| `ROLLBACK_TRIGGERED` | `"Rollback triggered for deployment {id}. Reason: {rollbackReason}"` |
| `USER_ROLE_CHANGED` | `"User {username} role changed from {oldRole} to {newRole}"` |
| `USER_DELETED` | `"User account deleted: {username}"` |

**Important:** `AuditService.log()` calls for auth events (`LOGIN_SUCCESS`, `LOGIN_FAILURE`) must use `@Transactional(propagation = Propagation.REQUIRES_NEW)` to ensure auth entries are persisted even if the outer transaction rolls back.

---

## 9. REST API Contract

Base URL: `http://localhost:8080/api`  
All endpoints return `ApiResponse<T>`. Paginated endpoints nest content in `data.content`.

### Pagination Parameters (all list endpoints)

| Param | Default | Description |
|-------|---------|-------------|
| `page` | `0` | Zero-based page index |
| `size` | `20` | Results per page (max 100) |
| `sort` | endpoint-specific | e.g. `startTime,desc` |

### Endpoint Catalog

#### Authentication

| Method | Path | Auth | Role | Success |
|--------|------|------|------|---------|
| POST | `/api/auth/register` | None | open | 201 — `AuthResponse` |
| POST | `/api/auth/login` | None | open | 200 — `AuthResponse` |
| POST | `/api/auth/refresh` | None | open | 200 — `AuthResponse` (new access token) |

#### Deployments

| Method | Path | Auth | Role | Success | Notes |
|--------|------|------|------|---------|-------|
| GET | `/api/deployments` | Bearer | ALL | 200 — `Page<DeploymentResponse>` | Supports `environment`, `status`, `projectName`, `startDate`, `endDate` query params |
| POST | `/api/deployments` | Bearer | ADMIN, DEVOPS | 201 — `DeploymentResponse` | `deployedBy` set from JWT |
| GET | `/api/deployments/{id}` | Bearer | ALL | 200 — `DeploymentResponse` | |
| PUT | `/api/deployments/{id}` | Bearer | ADMIN, DEVOPS | 200 — `DeploymentResponse` | Only updates `remarks` |
| DELETE | `/api/deployments/{id}` | Bearer | ADMIN | 204 No Content | 409 if rollback records exist |
| PATCH | `/api/deployments/{id}/status` | Bearer | ADMIN, DEVOPS | 200 — `DeploymentResponse` | 422 on invalid transition |
| POST | `/api/deployments/{id}/rollback` | Bearer | ADMIN, DEVOPS | 201 — `RollbackResponse` | 422 if deployment not FAILED |
| GET | `/api/deployments/{id}/rollbacks` | Bearer | ALL | 200 — `Page<RollbackResponse>` | |

#### Dashboard

| Method | Path | Auth | Role | Success |
|--------|------|------|------|---------|
| GET | `/api/dashboard/stats` | Bearer | ALL | 200 — `DashboardStatsResponse` |
| GET | `/api/dashboard/success-rate` | Bearer | ALL | 200 — `SuccessRateResponse` |
| GET | `/api/dashboard/recent` | Bearer | ALL | 200 — `List<DeploymentResponse>` (max 10) |

#### Users

| Method | Path | Auth | Role | Success |
|--------|------|------|------|---------|
| GET | `/api/users` | Bearer | ADMIN | 200 — `Page<UserResponse>` |
| GET | `/api/users/{id}` | Bearer | ADMIN | 200 — `UserResponse` |
| PUT | `/api/users/{id}/role` | Bearer | ADMIN | 200 — `UserResponse` |
| DELETE | `/api/users/{id}` | Bearer | ADMIN | 204 No Content |

#### Audit Logs

| Method | Path | Auth | Role | Success |
|--------|------|------|------|---------|
| GET | `/api/audit-logs` | Bearer | ADMIN | 200 — `Page<AuditLogResponse>` |

### Error Code Reference

| Code | Meaning | When |
|------|---------|------|
| 400 | Bad Request | Validation failure, missing fields, invalid enum value |
| 401 | Unauthorized | Missing/invalid/expired JWT, bad login credentials |
| 403 | Forbidden | Valid JWT but insufficient role |
| 404 | Not Found | Entity not found by ID |
| 409 | Conflict | Duplicate username/email; delete blocked by FK constraint |
| 422 | Unprocessable Entity | Invalid status transition; rollback on non-FAILED deployment |
| 500 | Internal Server Error | Unexpected error; response includes reference UUID |

---

## 10. Seed Data (V6__seed_initial_data.sql)

```sql
-- Users (BCrypt hashes for local dev passwords)
INSERT INTO users (username, email, password, role) VALUES
  ('admin',     'admin@enterprise.com',     '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewFX5L2fXhwj3Ymy', 'ADMIN'),
  ('devops',    'devops@enterprise.com',    '$2a$12$KmJz1c2yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewFX5L2fXhwj3Ymy', 'DEVOPS_ENGINEER'),
  ('developer', 'developer@enterprise.com', '$2a$12$NQr3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewFX5L2fXhwj3Ymy', 'DEVELOPER')
ON CONFLICT (username) DO NOTHING;
-- Plaintext: admin=admin123, devops=devops123, developer=dev123

-- Sample deployments
INSERT INTO deployments (project_name, version, environment, status, deployed_by, start_time, end_time, remarks) VALUES
  ('payment-service',  '1.4.2', 'PRODUCTION', 'SUCCESSFUL', 'devops', NOW()-INTERVAL '3 days', NOW()-INTERVAL '3 days'+INTERVAL '12 min', 'Production release Q3'),
  ('auth-service',     '2.1.0', 'STAGING',    'DEPLOYING',  'devops', NOW()-INTERVAL '1 hour',  NULL, 'Staging validation'),
  ('user-api',         '0.9.5', 'DEV',        'FAILED',     'developer', NOW()-INTERVAL '2 hours', NOW()-INTERVAL '105 min', 'Build error'),
  ('notification-svc', '1.0.0', 'QA',         'TESTING',    'devops', NOW()-INTERVAL '30 min', NULL, 'QA regression suite'),
  ('api-gateway',      '3.2.1', 'STAGING',    'PENDING',    'devops', NOW()-INTERVAL '5 min',  NULL, NULL)
ON CONFLICT DO NOTHING;

-- Sample rollback for the FAILED user-api
INSERT INTO rollbacks (deployment_id, previous_version, rollback_reason, rollback_time)
VALUES (
  (SELECT id FROM deployments WHERE project_name='user-api' AND version='0.9.5' LIMIT 1),
  '0.9.4',
  'NullPointerException in UserController line 42',
  NOW()-INTERVAL '100 min'
);
```
