# High-Level Design (HLD)

**Project:** Enterprise Deployment Dashboard
**Version:** 1.0
**Date:** 2026-07-13
**Status:** Approved

---

## Table of Contents

1. [Module Breakdown](#1-module-breakdown)
2. [Inter-Module Dependencies](#2-inter-module-dependencies)
3. [API Surface Summary](#3-api-surface-summary)
4. [Data Flow Diagrams](#4-data-flow-diagrams)
5. [Deployment Status State Machine](#5-deployment-status-state-machine)
6. [JWT Authentication Flow](#6-jwt-authentication-flow)

---

## 1. Module Breakdown

The application is organized into seven logical modules. Each module maps to a coherent slice of business functionality and owns its controller, service, and repository layers. All modules share the `common` module for cross-cutting types.

| Module | Responsibility | Key Classes | Primary Packages |
|---|---|---|---|
| **auth** | User registration, login, JWT access-token issuance, refresh-token flow | `AuthController`, `AuthService`, `JwtUtil`, `JwtAuthenticationFilter` | `controller`, `service`, `security` |
| **deployment** | Full CRUD for deployments, status-machine transitions, search and pagination | `DeploymentController`, `DeploymentService`, `DeploymentRepository` | `controller`, `service`, `repository` |
| **rollback** | Rollback triggering against a `FAILED` deployment, rollback-history retrieval | `DeploymentController` (trigger endpoint), `RollbackService`, `RollbackRepository` | `service`, `repository` |
| **audit** | Automatic event logging on all mutating operations; admin query and pagination | `AuditController`, `AuditService`, `AuditLogRepository` | `controller`, `service`, `repository`, `audit` |
| **user** | User management for ADMIN: list users, update role, delete user | `UserController`, `UserService`, `UserRepository` | `controller`, `service`, `repository` |
| **dashboard** | Aggregated deployment metrics: counts, success rate, per-environment breakdown, recent deployments | `DashboardController`, `DashboardService` | `controller`, `service` |
| **common** | Shared DTOs, universal response wrapper, domain exceptions, enumerations | `ApiResponse<T>`, `GlobalExceptionHandler`, `AppException` subclasses, `DeploymentStatus`, `Environment`, `Role` | `config`, `exception`, `model.dto`, `model.enums` |

### Package Structure

```
com.enterprise.dashboard
├── controller
│   ├── AuthController.java
│   ├── DeploymentController.java
│   ├── DashboardController.java
│   ├── UserController.java
│   └── AuditController.java
├── service
│   ├── AuthService.java
│   ├── DeploymentService.java
│   ├── RollbackService.java
│   ├── DashboardService.java
│   ├── UserService.java
│   └── AuditService.java
├── repository
│   ├── UserRepository.java
│   ├── DeploymentRepository.java
│   ├── RollbackRepository.java
│   └── AuditLogRepository.java
├── model
│   ├── entity
│   │   ├── User.java
│   │   ├── Deployment.java
│   │   ├── Rollback.java
│   │   └── AuditLog.java
│   ├── dto
│   │   ├── request
│   │   │   ├── LoginRequest.java
│   │   │   ├── RegisterRequest.java
│   │   │   ├── DeploymentRequest.java
│   │   │   ├── StatusUpdateRequest.java
│   │   │   └── RollbackRequest.java
│   │   └── response
│   │       ├── AuthResponse.java
│   │       ├── DeploymentResponse.java
│   │       ├── DashboardStatsResponse.java
│   │       ├── UserResponse.java
│   │       └── AuditLogResponse.java
│   └── enums
│       ├── Role.java               # ADMIN, DEVOPS_ENGINEER, DEVELOPER
│       ├── DeploymentStatus.java   # PENDING, BUILDING, TESTING, DEPLOYING, SUCCESSFUL, FAILED, ROLLED_BACK
│       └── Environment.java        # DEV, QA, STAGING, PRODUCTION
├── security
│   ├── JwtUtil.java
│   ├── JwtAuthenticationFilter.java
│   ├── CustomUserDetailsService.java
│   └── SecurityConfig.java
├── exception
│   ├── AppException.java
│   ├── ResourceNotFoundException.java
│   ├── DuplicateResourceException.java
│   ├── InvalidStatusTransitionException.java
│   ├── UnauthorizedOperationException.java
│   └── GlobalExceptionHandler.java
├── config
│   ├── OpenApiConfig.java
│   └── ApplicationConfig.java
└── util
    └── ApiResponse.java
```

---

## 2. Inter-Module Dependencies

Dependency direction: `A ──→ B` means A calls B (A depends on B). All dependencies are unidirectional; there are no circular dependencies.

```
dashboard ──────→ deployment (queries deployment data)
    └───────────→ audit (queries audit data)

deployment ─────→ audit (writes audit entries)
    └───────────→ rollback (creates rollback records on FAILED → ROLLED_BACK)

auth ───────────→ audit (writes login/logout/register events)
    └───────────→ user (creates user record on registration)

user ───────────→ audit (writes role-change and delete events)

All modules use common (ApiResponse, exceptions, enums)
```

### Dependency Matrix

| Consumer ↓ / Provider → | auth | deployment | rollback | audit | user | dashboard | common |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **auth** | — | | | W | W | | Y |
| **deployment** | | — | W | W | | | Y |
| **rollback** | | R | — | W | | | Y |
| **dashboard** | | R | | R | | — | Y |
| **user** | | | | W | — | | Y |
| **audit** | | | | — | | | Y |
| **common** | | | | | | | — |

_R = reads, W = writes, Y = uses shared types_

### Key Integration Points

- **`DeploymentService` → `AuditService`:** Every call to create, update, delete, or change deployment status also calls `AuditService.log(username, action, description)` within the same `@Transactional` boundary.
- **`AuthService` → `UserService` (register):** `AuthService.register()` delegates user entity persistence to `UserService.createUser()` to avoid duplicating user-creation logic.
- **`RollbackService` → `DeploymentService`:** After recording the rollback, `RollbackService` calls `DeploymentService.updateStatus(id, ROLLED_BACK)` to keep the deployment's status consistent.
- **All modules → `ApiResponse`:** Every controller endpoint returns `ResponseEntity<ApiResponse<T>>`, where `T` is the module-specific response DTO.

---

## 3. API Surface Summary

All endpoints are prefixed with `/api`. All authenticated endpoints require `Authorization: Bearer <accessToken>` header.

| Module | Endpoint Prefix | HTTP Methods | Roles | Auth Required |
|---|---|---|---|:---:|
| **auth** | `/api/auth` | POST | All (public) | No — register, login, refresh are unauthenticated |
| **deployments** | `/api/deployments` | GET, POST, PUT, DELETE | ADMIN + DEVOPS_ENGINEER (write); ALL roles (read) | Yes |
| **rollback** | `/api/deployments/{id}/rollback` | POST (trigger), GET (history) | ADMIN, DEVOPS_ENGINEER | Yes |
| **dashboard** | `/api/dashboard` | GET | All authenticated roles | Yes |
| **users** | `/api/users` | GET, PUT, DELETE | ADMIN only | Yes |
| **audit** | `/api/audit-logs` | GET | ADMIN only | Yes |

### Endpoint Inventory

#### Auth (`/api/auth`)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/api/auth/register` | Register a new user | No |
| POST | `/api/auth/login` | Login; returns `accessToken` and `refreshToken` | No |
| POST | `/api/auth/refresh` | Exchange `refreshToken` for new `accessToken` | No |

#### Deployments (`/api/deployments`)

| Method | Path | Description | Roles |
|---|---|---|---|
| GET | `/api/deployments` | List all deployments (paginated, filterable by environment/status/project/date) | ALL |
| POST | `/api/deployments` | Create a new deployment | ADMIN, DEVOPS_ENGINEER |
| GET | `/api/deployments/{id}` | Get deployment by ID | ALL |
| PUT | `/api/deployments/{id}` | Update deployment metadata | ADMIN, DEVOPS_ENGINEER |
| DELETE | `/api/deployments/{id}` | Delete deployment | ADMIN |
| PATCH | `/api/deployments/{id}/status` | Advance or fail deployment status | ADMIN, DEVOPS_ENGINEER |
| POST | `/api/deployments/{id}/rollback` | Trigger rollback (FAILED → ROLLED_BACK) | ADMIN, DEVOPS_ENGINEER |
| GET | `/api/deployments/{id}/rollbacks` | Get rollback history for a deployment | ADMIN, DEVOPS_ENGINEER |

#### Dashboard (`/api/dashboard`)

| Method | Path | Description | Roles |
|---|---|---|---|
| GET | `/api/dashboard/stats` | Returns total, successful, failed, pending counts + success rate | ALL |
| GET | `/api/dashboard/recent` | Returns the N most recent deployments | ALL |
| GET | `/api/dashboard/by-environment` | Deployment counts grouped by environment | ALL |

#### Users (`/api/users`)

| Method | Path | Description | Roles |
|---|---|---|---|
| GET | `/api/users` | List all users (paginated) | ADMIN |
| GET | `/api/users/{id}` | Get user by ID | ADMIN |
| PUT | `/api/users/{id}/role` | Update user's role | ADMIN |
| DELETE | `/api/users/{id}` | Delete user | ADMIN |

#### Audit Logs (`/api/audit-logs`)

| Method | Path | Description | Roles |
|---|---|---|---|
| GET | `/api/audit-logs` | Query audit logs (paginated; filter by username, action, date range) | ADMIN |

---

## 4. Data Flow Diagrams

### DFD Level 0 — System as Black Box

```
[Admin]            ──────────┐
[DevOps Engineer]  ──────────┼──→ [ Enterprise Deployment Dashboard ] ──→ [Deployment Records]
[Developer]        ──────────┘              ↑↓                           [Audit Logs]
                                       [PostgreSQL]                       [Dashboard Metrics]
```

The system has one external data store (PostgreSQL 15) and three categories of external actors (differentiated by role). All actors interact through the same REST API boundary; authorization logic inside the system enforces role-specific access.

### DFD Level 1 — Internal Processes

```
External User
    │
    ├──→ [1. Authentication Process]
    │         Input:  credentials (username + password) OR refresh token
    │         Output: JWT accessToken (24 hours) + refreshToken (7 days)
    │         Reads:  users table
    │         Writes: users table (register), audit_logs (login/register events)
    │
    ├──→ [2. Deployment Management]
    │         Input:  JWT + deployment data / status update / rollback request
    │         Output: DeploymentResponse / paginated DeploymentResponse list
    │         Reads:  deployments table, rollbacks table
    │         Writes: deployments table, rollbacks table, audit_logs
    │
    ├──→ [3. Dashboard Aggregation]
    │         Input:  JWT
    │         Output: DashboardStatsResponse (counts, success rate, recent deployments,
    │                 per-environment breakdown)
    │         Reads:  deployments table (COUNT, GROUP BY queries — read-only)
    │         Writes: (none)
    │
    ├──→ [4. User Management]
    │         Input:  JWT (Admin only) + optional role update / delete request
    │         Output: UserResponse / paginated UserResponse list
    │         Reads:  users table
    │         Writes: users table, audit_logs (role change / delete events)
    │
    └──→ [5. Audit Log Query]
              Input:  JWT (Admin role required) + filter params (username, action, dateFrom, dateTo)
              Output: Paginated AuditLogResponse list
              Reads:  audit_logs table
              Writes: (none)
```

### DFD Level 2 — Deployment Status Change (Detail)

```
Client
  │
  │  PATCH /api/deployments/{id}/status
  │  Body: { "status": "BUILDING" }
  │  Header: Authorization: Bearer <token>
  │
  ▼
JwtAuthenticationFilter
  │  Validate token → extract username, roles
  ▼
DeploymentController.updateStatus()
  │  @PreAuthorize("hasAnyRole('ADMIN','DEVOPS_ENGINEER')")
  ▼
DeploymentService.updateStatus(id, newStatus)
  │
  ├── DeploymentRepository.findById(id)
  │     → Deployment entity (or ResourceNotFoundException)
  │
  ├── StatusTransitionValidator.validate(current, new)
  │     → (or InvalidStatusTransitionException)
  │
  ├── deployment.setStatus(newStatus)
  │   deployment.setUpdatedAt(now)
  │
  ├── DeploymentRepository.save(deployment)
  │
  ├── AuditService.log(username, DEPLOYMENT_STATUS_UPDATED, description)
  │     → AuditLogRepository.save(auditLog)
  │
  └── Return DeploymentResponse DTO
  │
  ▼
DeploymentController
  │  Return ResponseEntity<ApiResponse<DeploymentResponse>> 200 OK
  ▼
Client receives updated deployment with new status
```

---

## 5. Deployment Status State Machine

### State Diagram

```
                           ┌─────────┐
                           │ PENDING │
                           └────┬────┘
                                │ ADMIN/DevOps: advance to build
                                ▼
                           ┌──────────┐
                           │ BUILDING │
                           └─────┬────┘
                                 │ build succeeded
                                 ▼
                           ┌─────────┐
                           │ TESTING │
                           └────┬────┘
                                │ tests passed
                                ▼
                           ┌───────────┐
                           │ DEPLOYING │
                           └─────┬─────┘
                                 │ deployment succeeded
                                 ▼
                           ┌────────────┐
                           │ SUCCESSFUL │
                           └────────────┘

   PENDING   ──────────────────────────────────────────────────→ FAILED
   BUILDING  ──────────────────────────────────────────────────→ FAILED
   TESTING   ──────────────────────────────────────────────────→ FAILED
   DEPLOYING ──────────────────────────────────────────────────→ FAILED

   FAILED    ──────────────────────────────────────────────────→ ROLLED_BACK
             (ADMIN/DevOps trigger rollback)
```

### Valid Transitions

| From | To | Who Can Trigger | API Endpoint |
|---|---|---|---|
| `PENDING` | `BUILDING` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `BUILDING` | `TESTING` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `TESTING` | `DEPLOYING` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `DEPLOYING` | `SUCCESSFUL` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `PENDING` | `FAILED` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `BUILDING` | `FAILED` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `TESTING` | `FAILED` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `DEPLOYING` | `FAILED` | ADMIN, DEVOPS_ENGINEER | `PATCH /api/deployments/{id}/status` |
| `FAILED` | `ROLLED_BACK` | ADMIN, DEVOPS_ENGINEER | `POST /api/deployments/{id}/rollback` |

### Invalid Transitions

Any transition not listed above is rejected with `HTTP 400 Bad Request` and the error message `"Invalid status transition from {current} to {requested}"`. Notable invalid transitions:

- `SUCCESSFUL → *` — a successful deployment cannot be re-entered into the pipeline (create a new deployment instead)
- `ROLLED_BACK → *` — terminal state; no further transitions permitted
- `FAILED → BUILDING` — a failed deployment must be rolled back before retry; create a new deployment for the retry

### State Machine Implementation

```java
// InvalidStatusTransitionException is thrown for illegal transitions
public enum DeploymentStatus {
    PENDING, BUILDING, TESTING, DEPLOYING, SUCCESSFUL, FAILED, ROLLED_BACK;

    public boolean canTransitionTo(DeploymentStatus next) {
        return switch (this) {
            case PENDING   -> next == BUILDING  || next == FAILED;
            case BUILDING  -> next == TESTING   || next == FAILED;
            case TESTING   -> next == DEPLOYING || next == FAILED;
            case DEPLOYING -> next == SUCCESSFUL || next == FAILED;
            case FAILED    -> next == ROLLED_BACK;
            case SUCCESSFUL, ROLLED_BACK -> false; // terminal
        };
    }
}
```

---

## 6. JWT Authentication Flow

### Token Structure

| Token | Lifetime | Claims | Storage (client-side) |
|---|---|---|---|
| Access token | 24 hours (86400000ms, configurable via JWT_EXPIRY_MS) | `sub` (username), `roles`, `iat`, `exp` | Memory / Authorization header |
| Refresh token | 7 days | `sub` (username), `iat`, `exp` | Secure HTTP-only cookie or local storage |

### Login Sequence

```
Client                JwtAuthFilter          AuthController          DB
  │                        │                      │                   │
  │─── POST /api/auth/login ──────────────────────→│                   │
  │    { username, password }                      │                   │
  │                        │                      │                   │
  │          (no JWT check — public endpoint)      │                   │
  │                        │                      │                   │
  │                        │                      │── findByUsername()→│
  │                        │                      │←─ User entity ─────│
  │                        │                      │                   │
  │                        │                      │  BCrypt.matches(   │
  │                        │                      │    rawPassword,    │
  │                        │                      │    storedHash)     │
  │                        │                      │                   │
  │                        │                      │  JwtUtil.generate  │
  │                        │                      │  AccessToken()     │
  │                        │                      │  JwtUtil.generate  │
  │                        │                      │  RefreshToken()    │
  │                        │                      │                   │
  │                        │                      │── AuditService     │
  │                        │                      │   .log(USER_LOGIN)→│
  │                        │                      │                   │
  │←── 200 OK ─────────────────────────────────────│                   │
  │    { accessToken,                              │                   │
  │      refreshToken,                             │                   │
  │      expiresIn: 86400 }                         │                   │
```

### Authenticated Request Sequence

```
Client                JwtAuthFilter          Controller          DB
  │                        │                      │               │
  │─── GET /api/deployments ──────────────────────→│               │
  │    Authorization: Bearer <accessToken>         │               │
  │                        │                      │               │
  │                        │  extractToken()       │               │
  │                        │  JwtUtil.validate()   │               │
  │                        │  (sig + expiry)       │               │
  │                        │                      │               │
  │                        │  loadUserByUsername() │               │
  │                        │──────────────────────────────────────→│
  │                        │←─────── UserDetails ──────────────────│
  │                        │                      │               │
  │                        │  setAuthentication() │               │
  │                        │  (SecurityContext)    │               │
  │                        │                      │               │
  │                        │── continue filter chain ─────────────→│
  │                        │                      │               │
  │                        │                      │  @PreAuthorize │
  │                        │                      │  role check    │
  │                        │                      │               │
  │                        │                      │── service()    │
  │                        │                      │── repo query() │
  │                        │                      │←── results ────│
  │                        │                      │               │
  │←── 200 OK { data: [...] } ────────────────────│               │
```

### Token Refresh Sequence

```
Client                JwtAuthFilter          AuthController       DB
  │                        │                      │               │
  │─── POST /api/auth/refresh ────────────────────→│               │
  │    { refreshToken: "..." }                     │               │
  │                        │                      │               │
  │          (no JWT check — public endpoint)      │               │
  │                        │                      │               │
  │                        │                      │  JwtUtil       │
  │                        │                      │  .validateToken│
  │                        │                      │  (refresh)     │
  │                        │                      │               │
  │                        │                      │  extract sub   │
  │                        │                      │── findUser() ─→│
  │                        │                      │←─ User ────────│
  │                        │                      │               │
  │                        │                      │  JwtUtil       │
  │                        │                      │  .generateNew  │
  │                        │                      │  AccessToken() │
  │                        │                      │               │
  │←── 200 OK { accessToken, expiresIn } ─────────│               │
```

### Security Properties of the JWT Implementation

| Property | Implementation |
|---|---|
| Signature algorithm | HMAC-SHA256 (HS256) — symmetric; secret shared only within the application |
| Key source | `JWT_SECRET` environment variable; never hardcoded |
| Token expiry enforcement | `exp` claim validated by `JwtUtil.validateToken()` on every request |
| Token revocation (v1) | Not supported — tokens are valid until expiry (acceptable for 24-hour access tokens) |
| Token revocation (v2) | Redis blacklist: token `jti` stored on logout; `JwtAuthFilter` checks blacklist before trusting token |
| Claims carried | `sub` (username), `roles` (list), `iat`, `exp` — no sensitive PII in token payload |
| Refresh token rotation | Out of scope for v1; planned for v2 (issue new refresh token on each refresh, invalidate previous) |
