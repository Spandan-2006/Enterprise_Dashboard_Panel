# Software Requirements Specification (SRS)

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Date:** 2026-07-13  
**Status:** Approved  
**Companion Document:** [docs/01-prd.md](./01-prd.md)

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Detailed Functional Requirements](#3-detailed-functional-requirements)
4. [Use Case Specifications](#4-use-case-specifications)
5. [Data Requirements](#5-data-requirements)
6. [Interface Requirements](#6-interface-requirements)
7. [Quality Attributes](#7-quality-attributes)

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification defines the complete technical requirements for the Enterprise Deployment Dashboard. It provides developers, architects, and QA engineers with sufficient detail to design, implement, and verify every component of the system. It is intended to be read alongside the PRD (`docs/01-prd.md`), which covers business motivation and user stories; this document focuses exclusively on technical specification.

### 1.2 Scope

The system is a **backend REST API** built with Java 21 and Spring Boot, backed by PostgreSQL 15+, and secured with stateless JWT authentication. An optional React frontend may consume the API but is not specified in this document. The API exposes resources for user management, deployment lifecycle management, rollback management, audit logging, and dashboard metrics.

All access to the API (except the authentication endpoints) requires a valid Bearer token. The system enforces RBAC at the method level using Spring Security.

### 1.3 Acronyms and Definitions

| Term | Definition |
|------|------------|
| **API** | Application Programming Interface — a set of HTTP endpoints exposing the system's capabilities to consumers |
| **ADR** | Architecture Decision Record — a document capturing the context, decision, and consequences of a significant architectural choice |
| **BCrypt** | A password-hashing algorithm incorporating a cost factor (work factor) to resist brute-force attacks; used to store all user passwords |
| **CI/CD** | Continuous Integration / Continuous Deployment — automated pipelines that build, test, and deploy software |
| **CRUD** | Create, Read, Update, Delete — the four fundamental data-access operations |
| **DTO** | Data Transfer Object — a plain Java object used to decouple the API request/response shape from the internal domain model |
| **JPA** | Jakarta Persistence API — the standard Java ORM specification; implemented by Hibernate in this project |
| **JWT** | JSON Web Token — a compact, URL-safe token format used for stateless authentication and authorization |
| **RBAC** | Role-Based Access Control — an authorization model where permissions are granted to roles, and roles are assigned to users |
| **REST** | Representational State Transfer — an architectural style for distributed hypermedia systems; used as the API paradigm |

---

## 2. System Overview

### 2.1 Context Diagram

The following ASCII diagram shows how the three external actor groups interact with the Spring Boot application and its database.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         External Actors                              │
│      [Admin]       [DevOps Engineer]       [Developer]              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTPS/REST (JSON)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                Spring Boot Application (Port 8080)                   │
│   Spring Security → Controllers → Services → Repositories           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ JDBC / JPA
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 PostgreSQL Database (Port 5432)                      │
│     users | deployments | rollbacks | audit_logs                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Layer Architecture

The application follows a strict layered architecture. Cross-layer dependencies are prohibited (e.g., a repository must not be injected directly into a controller).

| Layer | Package | Responsibility |
|-------|---------|----------------|
| **Controller** | `controller` | Parse HTTP requests, invoke services, serialize HTTP responses |
| **Service** | `service` | Business logic, transaction boundaries, audit log creation |
| **Repository** | `repository` | JPA data-access interfaces extending `JpaRepository` |
| **Security** | `security` | JWT filter chain, `UserDetailsService`, method-level `@PreAuthorize` |
| **Domain** | `model` | JPA entities, enums (`Role`, `DeploymentStatus`, `Environment`) |
| **DTO** | `dto` | Request/response POJOs; input validation annotations |
| **Exception** | `exception` | Custom exception classes and a global `@RestControllerAdvice` |

---

## 3. Detailed Functional Requirements

This section expands the functional requirements from the PRD with precise input contracts, processing logic, output contracts, and error conditions. FR numbers match the PRD exactly.

---

### FR-001 — User Registration

**Endpoint:** `POST /api/auth/register`  
**Access:** Admin only (ADMIN role required)

**Input:**
```json
{
  "username": "string (1–50 chars, alphanumeric + underscore)",
  "email":    "string (valid email, max 100 chars)",
  "password": "string (min 8 chars)",
  "role":     "ADMIN | DEVOPS_ENGINEER | DEVELOPER"
}
```

**Processing Logic:** Validate all fields are present and within constraints. Check that `username` and `email` are not already registered (throw `409 Conflict` if duplicate). Hash the password with BCrypt (strength 12). Persist a new `User` entity with `createdAt = now()`. Log `USER_REGISTERED` to `audit_logs`.

**Output:** `UserResponse` DTO, HTTP 201.

**Error Conditions:**

| HTTP Status | Condition |
|-------------|-----------|
| 400 | Missing or invalid fields |
| 409 | `username` or `email` already exists |
| 403 | Caller does not have ADMIN role |

---

### FR-002 — Login

**Endpoint:** `POST /api/auth/login`  
**Access:** Public (no authentication required)

**Input:**
```json
{
  "username": "string",
  "password": "string"
}
```

**Processing Logic:** Validate that both fields are non-empty. Load the `User` record by `username`. If no user is found, return HTTP 401 with the generic message `"Invalid username or password"` (no user enumeration). If found, verify the submitted password against the stored BCrypt hash using `BCryptPasswordEncoder.matches()`. If verification fails, log `LOGIN_FAILURE` to `audit_logs` and return HTTP 401. If verification succeeds, generate a JWT access token (expiry: configurable, default 86 400 s / 24 h) and a JWT refresh token (expiry: 604 800 s / 7 d), both signed with the application's HMAC-SHA256 secret. Log `LOGIN_SUCCESS` to `audit_logs`.

**Output:**
```json
{
  "accessToken":  "string (JWT)",
  "refreshToken": "string (JWT)",
  "tokenType":    "Bearer",
  "expiresIn":    86400
}
```
HTTP 200.

**Error Conditions:**

| HTTP Status | Condition |
|-------------|-----------|
| 400 | `username` or `password` field is missing or blank |
| 401 | Credentials do not match any user record |

---

### FR-003 — Token Refresh

**Endpoint:** `POST /api/auth/refresh`  
**Access:** Public (presents the refresh token as proof of identity)

**Input:**
```json
{ "refreshToken": "string (JWT)" }
```

**Processing Logic:** Validate the refresh token signature and expiry. Extract the `username` claim. Issue a new access token with the default expiry. Return the existing refresh token unchanged (no rotation in v1).

**Output:** Same shape as FR-002 response, HTTP 200.

**Error Conditions:** 401 if refresh token is expired, malformed, or has an invalid signature.

---

### FR-004 — Create Deployment

**Endpoint:** `POST /api/deployments`  
**Access:** ADMIN, DEVOPS_ENGINEER

**Input:**
```json
{
  "projectName": "string (@NotBlank, max 100 chars)",
  "version":     "string (@NotBlank, max 50 chars)",
  "environment": "DEV | QA | STAGING | PRODUCTION",
  "remarks":     "string (optional, TEXT)"
}
```

**Processing Logic:** Validate all required fields are present and non-blank. Set `status = PENDING`. Set `deployedBy = username` extracted from the JWT. Set `startTime = now()`. Leave `endTime = null`. Persist the `Deployment` entity inside a transaction. After successful persistence, write an `AuditLog` entry with `action = DEPLOYMENT_CREATED` and a description including project name, version, and environment.

**Output:** `DeploymentResponse` DTO, HTTP 201.

**Error Conditions:**

| HTTP Status | Condition |
|-------------|-----------|
| 400 | Missing or blank required fields; invalid `environment` enum value |
| 403 | Caller has DEVELOPER role |

---

### FR-005 — Update Deployment Status

**Endpoint:** `PATCH /api/deployments/{id}/status`  
**Access:** ADMIN, DEVOPS_ENGINEER

**Processing Logic:** Load deployment by ID (404 if not found). Validate the requested status transition is permitted by the state machine. Update `status` on the entity. If the new status is a terminal state (`SUCCESSFUL`, `FAILED`, `ROLLED_BACK`), set `endTime = now()`. Log `DEPLOYMENT_STATUS_UPDATED` to `audit_logs`.

**Output:** `DeploymentResponse` DTO, HTTP 200.

**Error Conditions:** 404 if deployment not found; 422 if the status transition violates the state machine; 403 if caller is DEVELOPER.

---

### FR-006 — Get Deployment by ID

**Endpoint:** `GET /api/deployments/{id}`  
**Access:** All authenticated roles

**Processing Logic:** Load deployment entity by primary key. Map to `DeploymentResponse` DTO including any associated rollback records.

**Output:** `DeploymentResponse`, HTTP 200. Returns 404 if the deployment does not exist.

---

### FR-007 — List Deployments (Paginated)

**Endpoint:** `GET /api/deployments?page=0&size=20&sort=startTime,desc`  
**Access:** All authenticated roles

**Processing Logic:** Apply pagination and sorting parameters from query string. Use Spring Data's `Pageable` abstraction. Return `Page<DeploymentResponse>` wrapped in `ApiResponse`.

**Output:** Paginated list of `DeploymentResponse` objects, HTTP 200.

---

### FR-008 — Filter Deployments

**Endpoint:** `GET /api/deployments` (same as FR-007 with additional params)  
**Access:** All authenticated roles

**Processing Logic:** Accept optional query parameters `environment`, `status`, `projectName`, `startDate` (ISO-8601), `endDate` (ISO-8601). Build a JPA `Specification` combining all provided filters with `AND` logic. Empty/absent parameters are ignored (not applied as filters). Combine with pagination.

**Output:** Filtered, paginated list of `DeploymentResponse`, HTTP 200.

---

### FR-009 — Delete Deployment

**Endpoint:** `DELETE /api/deployments/{id}`  
**Access:** ADMIN only

**Processing Logic:** Load deployment by ID (404 if not found). Delete the record. Log `DEPLOYMENT_DELETED` to `audit_logs`, capturing the deployment's project name and version in the description.

**Output:** HTTP 204 No Content.

**Error Conditions:** 404 if not found; 403 if caller is not ADMIN.

---

### FR-010 — Deployment History by Project

**Endpoint:** `GET /api/deployments/project/{projectName}`  
**Access:** All authenticated roles

**Processing Logic:** Query all `Deployment` records matching the given `projectName` (case-insensitive), ordered by `startTime` descending. Return paginated results.

**Output:** Paginated `DeploymentResponse` list, HTTP 200.

---

### FR-011 — Trigger Rollback

**Endpoint:** `POST /api/deployments/{id}/rollback`  
**Access:** ADMIN, DEVOPS_ENGINEER

**Input:**
```json
{ "rollbackReason": "string (@NotBlank, TEXT)" }
```

**Processing Logic:** Load the `Deployment` by `id` from the path variable. If not found, return 404. Verify that `deployment.status == FAILED`; if not, return 422 with message `"Rollback only allowed for deployments in FAILED status"`. Within a single transaction: create a `Rollback` entity with `deploymentId = id`, `previousVersion = deployment.version`, `rollbackReason = request.rollbackReason`, `rollbackTime = now()`. Update `deployment.status = ROLLED_BACK` and `deployment.endTime = now()`. Persist both. After the transaction commits, write `ROLLBACK_TRIGGERED` to `audit_logs` with a description capturing deployment ID and rollback reason.

**Output:** `RollbackResponse` DTO, HTTP 201.

**Error Conditions:**

| HTTP Status | Condition |
|-------------|-----------|
| 404 | Deployment ID does not exist |
| 422 | Deployment status is not FAILED |
| 400 | `rollbackReason` is blank or missing |
| 403 | Caller has DEVELOPER role |

---

### FR-012 — Rollback Record Persistence

This requirement is satisfied as part of FR-011's processing logic. The `Rollback` entity stores: `id`, `deploymentId` (FK → deployments.id), `previousVersion`, `rollbackReason`, and `rollbackTime`. No additional endpoint is required for creation.

---

### FR-013 — View Rollback History for Deployment

**Endpoint:** `GET /api/deployments/{id}/rollbacks`  
**Access:** All authenticated roles

**Processing Logic:** Load all `Rollback` records where `deploymentId = id`, ordered by `rollbackTime` descending. Return 404 if the parent deployment does not exist.

**Output:** List of `RollbackResponse` DTOs, HTTP 200.

---

### FR-014 — Audit Logging: Authentication Events

This requirement is enforced in the `AuthService`. After every login attempt (success or failure), a service call writes an `AuditLog` row. This must not be conditional on the HTTP response — even a failed authentication attempt must produce an audit record. The `LOGIN_FAILURE` record stores `username` from the request body (regardless of whether the user exists).

---

### FR-015 — Audit Logging: Deployment Mutation Events

Audit logging for deployment events is handled in the `DeploymentService` and `RollbackService` after each successful write. Supported actions: `DEPLOYMENT_CREATED`, `DEPLOYMENT_STATUS_UPDATED`, `DEPLOYMENT_DELETED`, `ROLLBACK_TRIGGERED`. The `AuditLog.description` field should be human-readable (e.g., `"Deployment #42 for project 'payments-service' status changed to FAILED"`).

---

### FR-016 — Query Audit Logs (Admin)

**Endpoint:** `GET /api/audit-logs`  
**Access:** ADMIN only

**Processing Logic:** Accept optional filters `username`, `action`, `startDate`, `endDate` as query parameters. Accept pagination params `page`, `size`, `sort`. Build a JPA `Specification` and return a paginated result. Non-Admin callers receive 403.

**Output:** Paginated `AuditLogResponse` list, HTTP 200.

---

### FR-017 — Deployment Statistics

**Endpoint:** `GET /api/dashboard/stats`  
**Access:** ADMIN (full stats); DEVOPS_ENGINEER and DEVELOPER may access this endpoint with the same response shape

**Processing Logic:** Execute aggregate count queries grouped by status. Return scalar counts: `total`, `successful`, `failed`, `pending`, `building`, `testing`, `deploying`, `rolledBack`.

**Output:**
```json
{
  "total":      50,
  "successful": 30,
  "failed":     8,
  "pending":    5,
  "building":   3,
  "testing":    2,
  "deploying":  1,
  "rolledBack": 1
}
```
HTTP 200, wrapped in `ApiResponse`.

---

### FR-018 — Success Rate and Recent Deployments

**Endpoint:** Same `GET /api/dashboard/stats` response (extended)

**Processing Logic:** Compute `successRate = (successful / total) * 100`, rounded to two decimal places (return 0.00 if total is 0). Query the 5 most recent deployments ordered by `startTime` descending.

**Output:** The `data` field in `ApiResponse` includes `successRate` (Double) and `recentDeployments` (List of up to 5 `DeploymentResponse` summaries).

---

### FR-019 — List and Update Users (Admin)

**Endpoint:** `GET /api/users` (list, paginated), `PUT /api/users/{id}/role`  
**Access:** ADMIN only

**Processing Logic for role update:** Validate the supplied `role` value is a valid `Role` enum member. Load the user (404 if not found). Update `user.role`. Log `USER_ROLE_CHANGED` to `audit_logs` with description recording both the old and new role.

**Output:** `UserResponse` DTO, HTTP 200.

---

### FR-020 — Delete User (Admin)

**Endpoint:** `DELETE /api/users/{id}`  
**Access:** ADMIN only

**Processing Logic:** Load user by ID (404 if not found). Delete the record. Log `USER_DELETED` to `audit_logs`. System should prevent deletion of the last Admin user (return 422 with message `"Cannot delete the last Admin user"`).

**Output:** HTTP 204 No Content.

---

## 4. Use Case Specifications

### UC-001 — User Login

| Field | Detail |
|-------|--------|
| **Use Case ID** | UC-001 |
| **Name** | User Login |
| **Actor** | Any registered user (Admin, DevOps Engineer, Developer) |
| **Related FRs** | FR-002 |

**Preconditions:**
- The user account exists in the `users` table.
- The user is not already authenticated (or their current token has expired).

**Main Flow:**

1. User sends `POST /api/auth/login` with JSON body `{"username": "...", "password": "..."}`.
2. System validates that both fields are present and non-empty; returns HTTP 400 if either is blank.
3. System loads the `User` record from the database by `username`.
4. System calls `BCryptPasswordEncoder.matches(submittedPassword, storedHash)` to verify the password.
5. System generates a JWT access token (24-hour expiry) and a JWT refresh token (7-day expiry), both signed with the application HMAC-SHA256 secret.
6. System writes an `AuditLog` entry with `action = LOGIN_SUCCESS`, `username = username`, `timestamp = now()`.
7. System returns HTTP 200 with body `{"accessToken": "...", "refreshToken": "...", "tokenType": "Bearer", "expiresIn": 86400}`.

**Alternate Flow A — Invalid Password:**

- Step 4 fails (BCrypt mismatch).
- System writes `AuditLog` entry with `action = LOGIN_FAILURE`.
- System returns HTTP 401 with message `"Invalid username or password"`.
- Flow ends.

**Alternate Flow B — User Not Found:**

- Step 3 returns no result.
- System returns HTTP 401 with the same message `"Invalid username or password"` (no information about whether the username exists is disclosed).
- Flow ends (no audit log is written for unknown usernames to avoid storing arbitrary input).

**Postconditions:**
- On success: the user holds a valid JWT access token usable as a Bearer token on all secured endpoints; a `LOGIN_SUCCESS` record exists in `audit_logs`.
- On failure: a `LOGIN_FAILURE` record exists in `audit_logs` (for known usernames); no token is issued.

---

### UC-002 — Trigger Deployment

| Field | Detail |
|-------|--------|
| **Use Case ID** | UC-002 |
| **Name** | Trigger Deployment |
| **Actor** | Admin or DevOps Engineer |
| **Related FRs** | FR-004, FR-015 |

**Preconditions:**
- User is authenticated with a valid, non-expired JWT.
- User's role is ADMIN or DEVOPS_ENGINEER.

**Main Flow:**

1. User sends `POST /api/deployments` with JSON body `{"projectName": "...", "version": "...", "environment": "PRODUCTION", "remarks": "..."}`.
2. System validates all required fields are present, non-blank, and that `environment` is a valid enum value.
3. System creates a `Deployment` entity: `status = PENDING`, `deployedBy = <JWT username>`, `startTime = now()`, `endTime = null`.
4. System persists the `Deployment` entity in the database.
5. System writes an `AuditLog` entry: `action = DEPLOYMENT_CREATED`, `username = <JWT username>`, description includes project, version, and environment.
6. System returns HTTP 201 with `DeploymentResponse` DTO including the generated deployment ID.

**Alternate Flow — Validation Failure:**

- Step 2 fails (missing required field or invalid enum).
- System returns HTTP 400 with a structured error body listing each failing field.
- Flow ends.

**Postconditions:**
- A `Deployment` record exists in the database with `status = PENDING`.
- A `DEPLOYMENT_CREATED` entry exists in `audit_logs`.

---

### UC-003 — Perform Rollback

| Field | Detail |
|-------|--------|
| **Use Case ID** | UC-003 |
| **Name** | Perform Rollback |
| **Actor** | Admin or DevOps Engineer |
| **Related FRs** | FR-011, FR-012, FR-015 |

**Preconditions:**
- User is authenticated with ADMIN or DEVOPS_ENGINEER role.
- The target deployment exists in the database with `status = FAILED`.

**Main Flow:**

1. User sends `POST /api/deployments/{id}/rollback` with JSON body `{"rollbackReason": "Memory leak introduced in v2.4.1"}`.
2. System loads the `Deployment` record by `{id}`.
3. System verifies `deployment.status == FAILED`; proceeds to step 4 if true.
4. Within a single transaction: system creates a `Rollback` entity (`deploymentId = id`, `previousVersion = deployment.version`, `rollbackReason = request.rollbackReason`, `rollbackTime = now()`); system updates `deployment.status = ROLLED_BACK` and `deployment.endTime = now()`.
5. System writes `AuditLog` entry: `action = ROLLBACK_TRIGGERED`, including deployment ID and rollback reason.
6. System returns HTTP 201 with `RollbackResponse` DTO.

**Alternate Flow A — Deployment Not Found:**

- Step 2 finds no record for `{id}`.
- System returns HTTP 404 with message `"Deployment not found with id: {id}"`.
- Flow ends.

**Alternate Flow B — Deployment Not in FAILED Status:**

- Step 3 finds `deployment.status != FAILED`.
- System returns HTTP 422 with message `"Rollback only allowed for deployments in FAILED status"`.
- Flow ends.

**Postconditions:**
- A `Rollback` record exists in `rollbacks` table.
- The `Deployment` record has `status = ROLLED_BACK` and a non-null `endTime`.
- A `ROLLBACK_TRIGGERED` entry exists in `audit_logs`.

---

### UC-004 — View Audit Log

| Field | Detail |
|-------|--------|
| **Use Case ID** | UC-004 |
| **Name** | View Audit Log |
| **Actor** | Admin |
| **Related FRs** | FR-016 |

**Preconditions:**
- User is authenticated with ADMIN role.

**Main Flow:**

1. Admin sends `GET /api/audit-logs` with optional query parameters: `username=alice`, `action=LOGIN_FAILURE`, `startDate=2026-07-01T00:00:00`, `endDate=2026-07-13T23:59:59`, `page=0`, `size=20`.
2. System builds a JPA `Specification` combining all supplied filter predicates with `AND` logic; absent parameters contribute no predicate.
3. System executes the query with pagination and returns a `Page<AuditLog>`.
4. System maps results to `AuditLogResponse` DTOs and wraps in `ApiResponse`.
5. System returns HTTP 200 with the paginated response.

**Alternate Flow — Non-Admin Access:**

- Step 1: Spring Security's `@PreAuthorize("hasRole('ADMIN')")` rejects the request before it reaches the controller.
- System returns HTTP 403 with message `"Access denied"`.
- Flow ends.

**Postconditions:** None — this is a read-only operation; no state is modified.

---

### UC-005 — Manage Users (Role Update)

| Field | Detail |
|-------|--------|
| **Use Case ID** | UC-005 |
| **Name** | Update User Role |
| **Actor** | Admin |
| **Related FRs** | FR-019 |

**Preconditions:**
- User is authenticated with ADMIN role.
- Target user exists in the `users` table.

**Main Flow:**

1. Admin sends `PUT /api/users/{id}/role` with JSON body `{"role": "DEVOPS_ENGINEER"}`.
2. System validates that the `role` value is a valid `Role` enum member.
3. System loads the `User` by `{id}` (404 if not found).
4. System records the `previousRole` for the audit description.
5. System updates `user.role = newRole` and persists the change.
6. System writes `AuditLog` entry: `action = USER_ROLE_CHANGED`, description `"User '{username}' role changed from {previousRole} to {newRole}"`.
7. System returns HTTP 200 with `UserResponse` DTO reflecting the updated role.

**Alternate Flow — Invalid Role Value:**

- Step 2 fails (e.g., body contains `"role": "SUPERUSER"` which is not a valid enum value).
- System returns HTTP 400 with message `"Invalid role value: SUPERUSER"`.
- Flow ends.

**Postconditions:**
- `User` record reflects the new role.
- A `USER_ROLE_CHANGED` entry exists in `audit_logs` capturing both old and new role values.

---

## 5. Data Requirements

### 5.1 Entity: User

Database table: `users`

| Field | Java Type | DB Column | Constraints | Validation |
|-------|-----------|-----------|-------------|------------|
| id | Long | id | PK, auto-generated (IDENTITY) | — |
| username | String | username | NOT NULL, UNIQUE, max 50 chars | `@NotBlank`, `@Size(max=50)` |
| email | String | email | NOT NULL, UNIQUE, max 100 chars | `@NotBlank`, `@Email`, `@Size(max=100)` |
| password | String | password | NOT NULL; stored as BCrypt hash | `@NotBlank` (applied to raw input before hashing) |
| role | Role (enum) | role | NOT NULL; values: ADMIN, DEVOPS_ENGINEER, DEVELOPER | `@NotNull` |
| createdAt | LocalDateTime | created_at | NOT NULL, default `now()` | Set programmatically; not user-supplied |

### 5.2 Entity: Deployment

Database table: `deployments`

| Field | Java Type | DB Column | Constraints | Validation |
|-------|-----------|-----------|-------------|------------|
| id | Long | id | PK, auto-generated (IDENTITY) | — |
| projectName | String | project_name | NOT NULL, max 100 chars | `@NotBlank`, `@Size(max=100)` |
| version | String | version | NOT NULL, max 50 chars | `@NotBlank`, `@Size(max=50)` |
| environment | Environment (enum) | environment | NOT NULL; values: DEV, QA, STAGING, PRODUCTION | `@NotNull` |
| status | DeploymentStatus (enum) | status | NOT NULL; default PENDING | Set by service; not user-supplied at creation |
| deployedBy | String | deployed_by | NOT NULL, max 50 chars; references users.username | Set from JWT; not user-supplied |
| startTime | LocalDateTime | start_time | NOT NULL | Set to `now()` by service |
| endTime | LocalDateTime | end_time | Nullable | Set when terminal status reached |
| remarks | String | remarks | Nullable, TEXT column | `@Size(max=1000)` if provided |

**Valid DeploymentStatus transitions:**

```
PENDING → BUILDING
BUILDING → TESTING
BUILDING → FAILED
TESTING → DEPLOYING
TESTING → FAILED
DEPLOYING → SUCCESSFUL
DEPLOYING → FAILED
FAILED → ROLLED_BACK   (via rollback endpoint only)
```

### 5.3 Entity: Rollback

Database table: `rollbacks`

| Field | Java Type | DB Column | Constraints | Validation |
|-------|-----------|-----------|-------------|------------|
| id | Long | id | PK, auto-generated (IDENTITY) | — |
| deploymentId | Long | deployment_id | NOT NULL, FK → deployments.id | Set by service |
| previousVersion | String | previous_version | NOT NULL, max 50 chars | Copied from Deployment.version by service; `@NotBlank` |
| rollbackReason | String | rollback_reason | NOT NULL, TEXT | `@NotBlank` (user-supplied in request body) |
| rollbackTime | LocalDateTime | rollback_time | NOT NULL, default `now()` | Set by service |

### 5.4 Entity: AuditLog

Database table: `audit_logs`

| Field | Java Type | DB Column | Constraints | Validation |
|-------|-----------|-----------|-------------|------------|
| id | Long | id | PK, auto-generated (IDENTITY) | — |
| username | String | username | NOT NULL, max 50 chars | Set by service from JWT or request |
| action | String | action | NOT NULL, max 50 chars | One of the defined action constants (see §5.5) |
| timestamp | LocalDateTime | timestamp | NOT NULL, default `now()` | Set by service |
| description | String | description | Nullable, TEXT | Human-readable context string |

### 5.5 Audit Action Constants

| Constant | Trigger |
|----------|---------|
| `LOGIN_SUCCESS` | Successful authentication |
| `LOGIN_FAILURE` | Failed authentication attempt (known username) |
| `DEPLOYMENT_CREATED` | New deployment record created |
| `DEPLOYMENT_STATUS_UPDATED` | Deployment status changed |
| `DEPLOYMENT_DELETED` | Deployment record deleted by Admin |
| `ROLLBACK_TRIGGERED` | Rollback executed on a FAILED deployment |
| `USER_REGISTERED` | New user registered by Admin |
| `USER_ROLE_CHANGED` | User role updated by Admin |
| `USER_DELETED` | User account deleted by Admin |

---

## 6. Interface Requirements

### 6.1 Protocol

- **Transport:** HTTP/1.1; HTTPS (TLS 1.2+) mandatory in STAGING and PRODUCTION environments
- **Port:** 8080 (configurable via `server.port` property)
- **Content-Type:** `application/json` for all request and response bodies; charset UTF-8

### 6.2 Authentication

- **Mechanism:** Bearer token in the `Authorization` header: `Authorization: Bearer <accessToken>`
- **Exclusions:** `POST /api/auth/login` and `POST /api/auth/register` do not require a token
- **Token rejection:** Expired, malformed, or unsigned tokens return HTTP 401 before the request reaches any controller

### 6.3 Pagination

Standard Spring Data query parameters apply to all list endpoints:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | int | 0 | Zero-based page index |
| `size` | int | 20 | Number of records per page (max 100) |
| `sort` | string | `startTime,desc` or `timestamp,desc` | Field and direction, e.g., `projectName,asc` |

### 6.4 Response Envelope

Every API response is wrapped in a standard `ApiResponse<T>` envelope:

```json
{
  "success":   true,
  "message":   "Deployment created successfully",
  "data":      { ... },
  "timestamp": "2026-07-13T10:30:00Z",
  "errors":    null
}
```

On error:

```json
{
  "success":   false,
  "message":   "Validation failed",
  "data":      null,
  "timestamp": "2026-07-13T10:30:01Z",
  "errors": [
    { "field": "projectName", "message": "must not be blank" }
  ]
}
```

| Field | Type | Always present | Description |
|-------|------|---------------|-------------|
| `success` | boolean | Yes | `true` for 2xx, `false` for 4xx/5xx |
| `message` | string | Yes | Human-readable summary |
| `data` | T | On success | Response payload; `null` on error |
| `timestamp` | string (ISO-8601) | Yes | Server time when response was generated |
| `errors` | array | On validation errors | Per-field error details; `null` otherwise |

### 6.5 Standard HTTP Status Codes

| Code | Meaning in This API |
|------|---------------------|
| 200 | Successful retrieval or update |
| 201 | Successful creation (deployment, rollback, user) |
| 204 | Successful deletion (no response body) |
| 400 | Request body failed validation |
| 401 | Missing or invalid JWT; bad credentials |
| 403 | Authenticated but insufficient role |
| 404 | Requested resource does not exist |
| 409 | Conflict (e.g., duplicate username/email) |
| 422 | Business rule violation (e.g., invalid status transition) |
| 500 | Unexpected server error; structured error body returned, no stack trace |

---

## 7. Quality Attributes

### 7.1 Availability

| Criterion | Target |
|-----------|--------|
| Production uptime | 99.9% (less than ~8.7 hours downtime per year) |
| Health endpoint | `GET /actuator/health` always returns 200 with liveness data; must not be secured behind JWT |
| Database unavailability | Application returns structured 503 responses rather than unhandled exceptions; circuit-breaker pattern recommended for future v2 |

Implementation notes: Spring Boot Actuator's health endpoint must be exposed and excluded from the JWT filter chain. All uncaught exceptions in the `@RestControllerAdvice` must produce a structured `ApiResponse` with `success: false` and a generic message.

### 7.2 Maintainability

| Criterion | Target |
|-----------|--------|
| Test coverage | Minimum 80% line coverage across `service` and `controller` packages, measured with JaCoCo |
| Documentation | All `public` methods in service interfaces annotated with Javadoc describing parameters, return values, and thrown exceptions |
| Architecture enforcement | No repository interface injected directly into a controller class; controllers interact only with service interfaces |
| Naming conventions | Entities in `model`, request DTOs suffixed `Request`, response DTOs suffixed `Response`, services suffixed `Service`, repositories suffixed `Repository` |

### 7.3 Security

| Criterion | Target |
|-----------|--------|
| Password storage | BCrypt strength 12; passwords never logged or returned in any API response |
| Token lifecycle | Access tokens expire in 24 hours (configurable); tokens are stateless (no server-side session) |
| Privilege auditability | Every role change (user creation, role update, user deletion) generates an `AuditLog` entry — zero unaudited privilege escalations |
| SQL injection prevention | All database access via JPA parameterized queries; no native queries with string concatenation |
| Sensitive data exposure | Stack traces, SQL queries, and internal exception messages must never appear in API responses in any environment |
| CORS | `Access-Control-Allow-Origin` must be explicitly configured; wildcard `*` is not acceptable in PRODUCTION |

### 7.4 Testability

| Criterion | Target |
|-----------|--------|
| Unit testability | All service classes have corresponding unit test classes using Mockito for repository mocking; no Spring context required for service unit tests |
| Integration testability | An `application-test.properties` profile configures an H2 in-memory database, allowing all repository and controller integration tests to run without PostgreSQL |
| Controller testability | All REST endpoints tested via `MockMvc` with `@WebMvcTest`; security context stubbed with `@WithMockUser` |
| Audit log verification | Integration tests assert that the correct `AuditLog` entries are written after each mutating operation |
