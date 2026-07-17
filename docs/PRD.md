# Product Requirements Document (PRD)

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Status:** Approved

---

## 1. Problem Statement

Engineering organizations deploying software across multiple environments (DEV, QA, STAGING, PRODUCTION) lack a centralized, auditable system for tracking deployment lifecycles. Deployments are initiated and monitored through disconnected tools — CI/CD pipelines, Slack messages, spreadsheets — creating visibility gaps: no single source of truth exists for what is deployed where, who triggered it, whether it succeeded, and what rollbacks were performed.

When incidents occur, teams spend significant time reconstructing a timeline rather than resolving the issue. Unauthorized role changes go undetected because there is no comprehensive audit trail.

---

## 2. Solution

The Enterprise Deployment Dashboard is a role-aware REST API (with an optional React frontend) that provides:

- **Real-time deployment lifecycle tracking** across all environments under a unified API
- **One-click rollback management** with reason capture and full traceability
- **Complete append-only audit trails** recording every authentication event, deployment mutation, and privilege change
- **Role-Based Access Control (RBAC)** enforcing the principle of least privilege

---

## 3. Goals & Non-Goals

### Goals

| # | Goal |
|---|------|
| G-1 | Track the full lifecycle of every deployment: `PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL / FAILED → ROLLED_BACK` across all four environments |
| G-2 | Enforce RBAC with three roles — ADMIN, DEVOPS_ENGINEER, DEVELOPER — each limited to its responsibility level |
| G-3 | Enable rollback for any `FAILED` deployment, capturing reason and previous version |
| G-4 | Provide an append-only audit trail for all auth events, deployment mutations, and privilege changes |
| G-5 | Expose a metrics dashboard endpoint with total/successful/failed/pending counts and success rate |

### Non-Goals (v1 Out of Scope)

| # | Non-Goal | Rationale |
|---|----------|-----------|
| NG-1 | Code compilation and build pipeline execution | The dashboard tracks deployments; CI/CD tools run them |
| NG-2 | Artifact storage and management | Handled by Nexus/ECR/JFrog |
| NG-3 | Kubernetes / container orchestration | The system records status updates provided by external orchestrators |
| NG-4 | Email or push notifications | Status visibility is pull-only via API in v1 |
| NG-5 | OAuth2 / SSO | JWT with username/password is sufficient for v1 enterprise internal use |

---

## 4. User Roles & Personas

| Role | Responsibilities | Key Actions |
|------|-----------------|-------------|
| **ADMIN** | Manages system, all users, compliance | Register users; assign/update roles; view audit logs; delete deployments; view dashboard stats; delete users |
| **DEVOPS_ENGINEER** | Owns the deployment lifecycle | Create deployments; update status at each pipeline stage; trigger rollbacks; monitor pipeline |
| **DEVELOPER** | Consumes deployment information | View deployment status (read-only); view own audit entries; filter by environment and status |

---

## 5. User Stories

| ID | Story |
|----|-------|
| US-001 | As an **Admin**, I want to register new users and assign roles so team members can access the system with appropriate permissions |
| US-002 | As a **user**, I want to log in with username and password to securely access the dashboard |
| US-003 | As a **DevOps Engineer**, I want to create a new deployment for a project to track its progress |
| US-004 | As a **DevOps Engineer**, I want to update deployment status so the team sees real-time progress |
| US-005 | As a **DevOps Engineer**, I want to trigger a rollback for a failed deployment to quickly restore the previous stable version |
| US-006 | As a **Developer**, I want to view deployment history for my project to understand what versions are deployed where |
| US-007 | As a **Developer**, I want to filter deployments by environment and status to find relevant deployments quickly |
| US-008 | As an **Admin**, I want to view the full audit log to investigate unauthorized or suspicious activity |
| US-009 | As an **Admin**, I want to view dashboard statistics to track deployment success rates and identify problem areas |
| US-010 | As an **Admin**, I want to delete a deployment record to remove stale or erroneous entries |
| US-011 | As a **user**, I want meaningful error messages to understand why an operation failed |
| US-012 | As an **Admin**, I want to export audit logs *(future v2)* for compliance reporting |

---

## 6. Functional Requirements

### Authentication (FR-001 – FR-003)

- **FR-001** — Admin-only user registration: accepts `username`, `email`, `password`, `role`; password hashed with BCrypt before persistence
- **FR-002** — Login endpoint returns a JWT access token (24h) and refresh token (7d)
- **FR-003** — Token-refresh endpoint issues a new access token from a valid refresh token

### Deployment Lifecycle (FR-004 – FR-010)

- **FR-004** — Admin/DevOps creates deployment with `projectName`, `version`, `environment`; created with `status=PENDING`, `deployedBy=authenticated username`
- **FR-005** — Admin/DevOps updates deployment status through the valid state machine
- **FR-006** — Any authenticated user retrieves full detail of a single deployment
- **FR-007** — Paginated deployment list, default page 0 / size 20 / sorted by `startTime DESC`
- **FR-008** — Deployment list supports filtering by `environment`, `status`, `projectName`, date range
- **FR-009** — Admin permanently deletes a deployment; deletion logged to audit trail
- **FR-010** — Deployment history endpoint by `projectName`, reverse chronological order

### Rollback Management (FR-011 – FR-013)

- **FR-011** — Admin/DevOps triggers rollback for a `FAILED` deployment, supplying `rollbackReason`
- **FR-012** — Rollback persists a `Rollback` record (`deploymentId`, `previousVersion`, `rollbackReason`, `rollbackTime`); deployment status set to `ROLLED_BACK`
- **FR-013** — Endpoint returns all rollback records for a given deployment ID

### Audit Logging (FR-014 – FR-016)

- **FR-014** — Audit entry auto-created for every auth event: `username`, `action`, `timestamp`, `description`
- **FR-015** — Audit entry auto-created for every deployment mutation and rollback event
- **FR-016** — Admin-only paginated audit log query with filters: `username`, `action`, date range

### Dashboard Metrics (FR-017 – FR-018)

- **FR-017** — Metrics endpoint returns counts by status: total, successful, failed, pending, rolled back
- **FR-018** — Metrics endpoint also returns overall success rate and list of 5 most recent deployments

### User Management (FR-019 – FR-020)

- **FR-019** — Admin lists all users (paginated); updates any user's role; every role change logged as `USER_ROLE_CHANGED`
- **FR-020** — Admin deletes user account; logged as `USER_DELETED`

---

## 7. Non-Functional Requirements

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-001 | API response time (p95) | < 200 ms under 100 concurrent users |
| NFR-002 | JWT access token expiry | Configurable via property; default 24 hours |
| NFR-003 | Password hashing | BCrypt strength (cost factor) 12 |
| NFR-004 | Data integrity | Multi-step DB operations wrapped in single `@Transactional` boundary |
| NFR-005 | Availability | 99.9% uptime target in production |
| NFR-006 | Error responses | JSON body with `success: false`, `message`, `errors`; no raw stack traces |
| NFR-007 | Auth enforcement | Every endpoint except `/api/auth/*` and `/actuator/health` requires valid Bearer JWT |

---

## 8. Constraints

| Constraint | Detail |
|------------|--------|
| Runtime | Java 21 LTS only |
| Database | PostgreSQL 15+ only in v1 |
| Messaging | No external message queue in v1; all operations synchronous |
| Audit log immutability | Audit records are append-only; no delete/update via API |

---

## 9. Success Metrics

| Metric | Target |
|--------|--------|
| Deployment coverage | 100% of deployments tracked — zero production changes outside the dashboard |
| Incident traceability | Any rollback event fully traceable within 5 minutes |
| Privilege auditability | Zero unaudited privilege escalations |
| API performance | p95 < 200 ms under 100 concurrent users |
| Dashboard page load | < 2 seconds including frontend rendering |
