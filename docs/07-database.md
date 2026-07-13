# Database Design Document

**Project:** Enterprise Deployment Dashboard
**Version:** 1.0
**Date:** 2026-07-13
**Status:** Approved

---

## Table of Contents

1. [Database Choice Rationale](#1-database-choice-rationale)
2. [Entity-Relationship Diagram](#2-entity-relationship-diagram)
3. [Table Definitions](#3-table-definitions)
4. [Indexes](#4-indexes)
5. [JPA/Hibernate Mapping Notes](#5-jpahibernate-mapping-notes)
6. [Flyway Migration Strategy](#6-flyway-migration-strategy)
7. [Sample Seed Data](#7-sample-seed-data)

---

## 1. Database Choice Rationale

PostgreSQL 15 was selected as the primary data store. The authoritative decision record is documented in `docs/ADR/0001-use-postgresql.md` (Architecture Decision Record). The key reasons are summarized below.

### 1.1 ACID Transactions

PostgreSQL provides full ACID (Atomicity, Consistency, Isolation, Durability) guarantees at the row level. Deployment lifecycle transitions — for example, updating a deployment's status from `DEPLOYING` to `SUCCESSFUL` and simultaneously writing an audit log entry — must be atomic. A partial write (status updated but audit entry missing) would corrupt the audit trail. PostgreSQL's multi-version concurrency control (MVCC) ensures these operations commit together or not at all, without sacrificing read throughput.

### 1.2 Enum-Compatible VARCHAR with CHECK Constraints

Java enums (`DeploymentStatus`, `Environment`, `Role`) map cleanly to `VARCHAR` columns with `CHECK` constraints. This approach was preferred over PostgreSQL's native `CREATE TYPE ... AS ENUM` for two reasons:

- **ALTER flexibility:** Adding a new enum value (e.g., a new `PAUSED` status in v2) requires only an `ALTER TABLE ... ADD CHECK` migration, not a DDL `ALTER TYPE` that can be disruptive in a live database.
- **Hibernate compatibility:** `@Enumerated(EnumType.STRING)` on a `VARCHAR` column works without a custom `AttributeConverter`, keeping JPA mapping straightforward.

### 1.3 Spring Data JPA Integration

PostgreSQL is a first-class target for Spring Data JPA. The `spring-boot-starter-data-jpa` autoconfiguration works out of the box with the `postgresql` driver. Named queries, `@Query` JPQL, Criteria API, and Spring Data `Specification` all behave correctly without workarounds. The `org.postgresql:postgresql` driver is actively maintained and tested against every Spring Boot 3.x release.

### 1.4 Official Docker Image (postgres:15-alpine)

The `postgres:15-alpine` image is the official PostgreSQL image built on Alpine Linux. It is:

- **Small:** ~80 MB compressed, compared to ~150 MB for the Debian-based variant.
- **Secure:** Alpine's minimal footprint reduces the attack surface in containerized deployments.
- **Deterministic:** Pinning to `15-alpine` (not `latest`) ensures every developer and CI environment runs the same database version, eliminating "works on my machine" schema drift.

### 1.5 JSONB Available for Future Unstructured Data

PostgreSQL's `JSONB` column type stores binary-encoded JSON with full indexing support (GIN indexes). While the current schema does not use `JSONB`, it is reserved for future use cases such as:

- Storing deployment pipeline metadata (build logs, artifact manifests) without a schema migration.
- Capturing feature flags or environment-specific configuration overrides.

This optionality is available without changing the database engine in v2.

---

## 2. Entity-Relationship Diagram

The system has four tables. The relationship rules are:

- `deployments.deployed_by` stores the deploying user's username as a plain `VARCHAR` — it is **not a foreign key** to `users`. This preserves deployment history if the user record is later deleted.
- `rollbacks.deployment_id` is a `BIGINT` foreign key to `deployments.id` with `ON DELETE RESTRICT`. A deployment that has one or more rollback records cannot be deleted; the rollback history must be removed first (or retained indefinitely as the intended behavior).
- `audit_logs.username` stores the acting user's username as a plain `VARCHAR` — it is **not a foreign key** to `users`. The audit log is an immutable record of events; its integrity must not depend on the continued existence of a user row.

```
┌─────────────────────────┐
│          users          │
│  id            PK       │
│  username      UNIQUE   │
│  email         UNIQUE   │
│  password               │
│  role                   │
│  created_at             │
└─────────────────────────┘
         │
         │ deployed_by = username (string reference, NOT FK)
         │ (history preserved through user deletion)
         ▼
┌─────────────────────────┐
│       deployments       │
│  id            PK       │
│  project_name           │
│  version                │
│  environment            │
│  status                 │
│  deployed_by   VARCHAR  │◄── NOT FK; string copy of username
│  start_time             │
│  end_time               │
│  remarks                │
└─────────────┬───────────┘
              │ 1
    ┌─────────┴──────────┐
    │                    │
    │ 1:N                │ 1:N
    ▼                    ▼
┌──────────────┐   ┌──────────────────┐
│  rollbacks   │   │   audit_logs     │
│  id    PK    │   │  id        PK    │
│  deployment_ │   │  username  VARCHAR│◄── NOT FK; string copy
│  id    FK    │   │  action          │
│  ON DELETE   │   │  timestamp       │
│  RESTRICT    │   │  description     │
│  previous_   │   └──────────────────┘
│  version     │
│  rollback_   │
│  reason      │
│  rollback_   │
│  time        │
└──────────────┘
```

**Key design decisions visualized:**

| Relationship | Type | Constraint | Rationale |
|---|---|---|---|
| `deployments.deployed_by` → `users.username` | String reference | None (no FK) | Preserves deployment history if user deleted |
| `rollbacks.deployment_id` → `deployments.id` | FK (BIGINT) | `ON DELETE RESTRICT` | Cannot delete a deployment with rollback history |
| `audit_logs.username` → `users.username` | String reference | None (no FK) | Audit log immutability is independent of user lifecycle |

---

## 3. Table Definitions

All tables use `BIGSERIAL` primary keys (auto-incrementing 64-bit integers). Timestamps default to `NOW()` at the database level; the application layer also sets them via JPA `@CreationTimestamp` annotations for consistency.

### 3.1 users

```sql
-- Table: users
-- Stores registered system users with their assigned role.
-- The first ADMIN is created via seed data (V6__seed_initial_data.sql).
-- Subsequent ADMINs can only be created by an existing ADMIN via the API.
CREATE TABLE users (
  id          BIGSERIAL    PRIMARY KEY,
  username    VARCHAR(50)  NOT NULL UNIQUE,
  email       VARCHAR(100) NOT NULL UNIQUE,
  password    VARCHAR(255) NOT NULL,   -- BCrypt hash (strength 12); never stored in plain text
  role        VARCHAR(20)  NOT NULL
              CHECK (role IN ('ADMIN', 'DEVOPS_ENGINEER', 'DEVELOPER')),
  created_at  TIMESTAMP    NOT NULL DEFAULT NOW()
);
```

**Column notes:**

| Column | Type | Notes |
|---|---|---|
| `id` | `BIGSERIAL` | Surrogate primary key; auto-incremented by the database sequence |
| `username` | `VARCHAR(50)` | Unique login identifier; used as a string reference in `deployments` and `audit_logs` |
| `email` | `VARCHAR(100)` | Must be unique per user; used for notifications (future) |
| `password` | `VARCHAR(255)` | BCrypt hash with strength factor 12; 255 chars accommodates all current BCrypt output lengths |
| `role` | `VARCHAR(20)` | One of `ADMIN`, `DEVOPS_ENGINEER`, `DEVELOPER`; enforced by CHECK constraint and Spring Security |
| `created_at` | `TIMESTAMP` | Set at insert by `DEFAULT NOW()`; also managed by `@CreationTimestamp` in JPA |

### 3.2 deployments

```sql
-- Table: deployments
-- Central entity. Records each deployment attempt with its lifecycle status.
-- deployed_by is a VARCHAR copy of the user's username, NOT a FK — ensures
-- deployment history survives user deletion.
CREATE TABLE deployments (
  id            BIGSERIAL    PRIMARY KEY,
  project_name  VARCHAR(100) NOT NULL,
  version       VARCHAR(50)  NOT NULL,
  environment   VARCHAR(20)  NOT NULL
                CHECK (environment IN ('DEV', 'QA', 'STAGING', 'PRODUCTION')),
  status        VARCHAR(30)  NOT NULL DEFAULT 'PENDING'
                CHECK (status IN ('PENDING', 'BUILDING', 'TESTING', 'DEPLOYING',
                                  'SUCCESSFUL', 'FAILED', 'ROLLED_BACK')),
  deployed_by   VARCHAR(50)  NOT NULL,  -- string copy of users.username; NOT a FK
  start_time    TIMESTAMP    NOT NULL DEFAULT NOW(),
  end_time      TIMESTAMP,              -- NULL until deployment reaches a terminal state
  remarks       TEXT                    -- optional free-text notes about this deployment
);
```

**Column notes:**

| Column | Type | Notes |
|---|---|---|
| `id` | `BIGSERIAL` | Surrogate primary key |
| `project_name` | `VARCHAR(100)` | Name of the project/service being deployed (e.g., `payment-service`) |
| `version` | `VARCHAR(50)` | Semantic version or tag of the artifact being deployed (e.g., `1.4.2`, `main-abc123`) |
| `environment` | `VARCHAR(20)` | Target environment; constrained to the four valid values |
| `status` | `VARCHAR(30)` | Current lifecycle status; defaults to `PENDING` on creation |
| `deployed_by` | `VARCHAR(50)` | Username of the initiating user at creation time; stored as a string to survive user deletion |
| `start_time` | `TIMESTAMP` | When the deployment was created/initiated; set by `DEFAULT NOW()` |
| `end_time` | `TIMESTAMP` | NULL while active; set explicitly by the service layer when status reaches `SUCCESSFUL`, `FAILED`, or `ROLLED_BACK` |
| `remarks` | `TEXT` | Optional notes, reason for deployment, ticket reference, etc. |

**Status transition rules** (enforced at service layer, not database):

```
PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL
                                          ↘ FAILED → (manual fix) → back to PENDING
                                          ↘ ROLLED_BACK (via rollback trigger)
```

### 3.3 rollbacks

```sql
-- Table: rollbacks
-- Records each rollback event for a deployment.
-- deployment_id uses ON DELETE RESTRICT — a deployment with rollback records
-- cannot be deleted until those rollback records are removed or the constraint is
-- explicitly relaxed by a future migration.
CREATE TABLE rollbacks (
  id               BIGSERIAL    PRIMARY KEY,
  deployment_id    BIGINT       NOT NULL
                   REFERENCES deployments(id) ON DELETE RESTRICT,
  previous_version VARCHAR(50)  NOT NULL,   -- the version being reverted TO
  rollback_reason  TEXT         NOT NULL,   -- mandatory; must not be empty
  rollback_time    TIMESTAMP    NOT NULL DEFAULT NOW()
);
```

**Column notes:**

| Column | Type | Notes |
|---|---|---|
| `id` | `BIGSERIAL` | Surrogate primary key |
| `deployment_id` | `BIGINT` | FK to the deployment that was rolled back; `ON DELETE RESTRICT` prevents orphan rollback records |
| `previous_version` | `VARCHAR(50)` | The stable version being restored (e.g., `1.4.1` when rolling back from `1.4.2`) |
| `rollback_reason` | `TEXT` | Mandatory explanation; auditable reason for why the rollback was triggered |
| `rollback_time` | `TIMESTAMP` | When the rollback was recorded; set by `DEFAULT NOW()` |

**Why `ON DELETE RESTRICT` and not `ON DELETE CASCADE`:**
Cascading deletes on rollback history would allow a deployment record and all its associated rollbacks to be silently wiped. Rollback records are part of the compliance audit trail. `RESTRICT` forces an explicit decision — an operator must delete rollback records first (or archive them) before the parent deployment can be removed.

### 3.4 audit_logs

```sql
-- Table: audit_logs
-- Append-only event log. No DELETE or UPDATE is permitted through the application
-- API. Records are written by the AuditService after every significant action.
-- username is a VARCHAR, NOT a FK — audit history is preserved if the user is deleted.
CREATE TABLE audit_logs (
  id          BIGSERIAL   PRIMARY KEY,
  username    VARCHAR(50) NOT NULL,   -- string copy of acting user's username; NOT a FK
  action      VARCHAR(50) NOT NULL,   -- constant string, e.g. 'LOGIN_SUCCESS', 'DEPLOYMENT_CREATED'
  timestamp   TIMESTAMP   NOT NULL DEFAULT NOW(),
  description TEXT                    -- human-readable summary of the event
);
```

**Column notes:**

| Column | Type | Notes |
|---|---|---|
| `id` | `BIGSERIAL` | Surrogate primary key; monotonically increasing, provides rough insertion order |
| `username` | `VARCHAR(50)` | Username of the actor at the time of the event; survives user deletion |
| `action` | `VARCHAR(50)` | Machine-readable action constant (see full list below) |
| `timestamp` | `TIMESTAMP` | When the event occurred; set by `DEFAULT NOW()` |
| `description` | `TEXT` | Optional human-readable detail, e.g. "Status changed: payment-service PENDING → BUILDING" |

**Defined action constants:**

| Action Constant | Triggered By |
|---|---|
| `LOGIN_SUCCESS` | Successful `POST /api/auth/login` |
| `LOGIN_FAILURE` | Failed `POST /api/auth/login` (wrong password or unknown user) |
| `LOGOUT` | `POST /api/auth/logout` |
| `REGISTER` | Successful `POST /api/auth/register` |
| `DEPLOYMENT_CREATED` | `POST /api/deployments` |
| `DEPLOYMENT_UPDATED` | `PUT /api/deployments/{id}` |
| `DEPLOYMENT_DELETED` | `DELETE /api/deployments/{id}` |
| `DEPLOYMENT_STATUS_UPDATED` | `PATCH /api/deployments/{id}/status` |
| `ROLLBACK_TRIGGERED` | `POST /api/deployments/{id}/rollback` |
| `USER_ROLE_UPDATED` | `PATCH /api/users/{id}/role` |
| `USER_DELETED` | `DELETE /api/users/{id}` |

**Append-only enforcement:** The `AuditService` only exposes a `log(username, action, description)` method. There is no `update` or `delete` method in the service interface. At the repository layer, the `AuditLogRepository` extends `JpaRepository` but does not expose any mutation methods beyond save. Future enforcement can add a PostgreSQL row-level security policy or a trigger that raises an error on `UPDATE`/`DELETE` against this table.

---

## 4. Indexes

Indexes are created in a dedicated Flyway migration (`V5__create_indexes.sql`) to keep DDL migration scripts focused and reversible independently.

### 4.1 Index Summary Table

| Table | Index Name | Column(s) | Type | Rationale |
|---|---|---|---|---|
| `deployments` | `idx_deployments_status` | `(status)` | BTREE | Frequent single-column filter in the deployment list endpoint (`?status=FAILED`) |
| `deployments` | `idx_deployments_env_status` | `(environment, status)` | BTREE | Compound filter — the most common query pattern on the list endpoint (`?environment=PRODUCTION&status=SUCCESSFUL`) |
| `deployments` | `idx_deployments_deployed_by` | `(deployed_by)` | BTREE | User-scoped history lookup; DEVELOPER role users see only their own deployments |
| `deployments` | `idx_deployments_start_time` | `(start_time DESC)` | BTREE | Default `ORDER BY start_time DESC` sort and date-range `WHERE start_time BETWEEN` queries |
| `rollbacks` | `idx_rollbacks_deployment_id` | `(deployment_id)` | BTREE | FK lookup performance; every rollback query is by deployment ID |
| `audit_logs` | `idx_audit_logs_username_ts` | `(username, timestamp DESC)` | BTREE | Audit queries filtered by user and ordered by recency (ADMIN audit view) |
| `audit_logs` | `idx_audit_logs_action` | `(action)` | BTREE | Filter audit log by action constant (e.g., show all `LOGIN_FAILURE` events) |

### 4.2 Index DDL

```sql
-- deployments indexes
CREATE INDEX idx_deployments_status
    ON deployments(status);

CREATE INDEX idx_deployments_env_status
    ON deployments(environment, status);

CREATE INDEX idx_deployments_deployed_by
    ON deployments(deployed_by);

CREATE INDEX idx_deployments_start_time
    ON deployments(start_time DESC);

-- rollbacks indexes
CREATE INDEX idx_rollbacks_deployment_id
    ON rollbacks(deployment_id);

-- audit_logs indexes
CREATE INDEX idx_audit_logs_username_ts
    ON audit_logs(username, timestamp DESC);

CREATE INDEX idx_audit_logs_action
    ON audit_logs(action);
```

### 4.3 Index Design Notes

**Why no index on `deployments.project_name`?** Project name filters are expected to be used with `LIKE '%term%'` (substring search), which a B-tree index cannot satisfy efficiently. A `pg_trgm` GIN index would help if substring search becomes a performance bottleneck in v2; this is deferred until query analysis confirms the need.

**Why a DESC index on `start_time`?** PostgreSQL can use either ascending or descending scans on a B-tree index, but explicitly creating the index as `DESC` avoids a sort step when the query plan needs `ORDER BY start_time DESC LIMIT N` (common dashboard queries for recent deployments).

**Why a composite index on `(environment, status)` rather than two separate indexes?** The list endpoint's most frequent query pattern combines both filters. PostgreSQL's index scan on `(environment, status)` satisfies both the `WHERE environment = 'PRODUCTION'` and `WHERE environment = 'PRODUCTION' AND status = 'SUCCESSFUL'` cases with a single index lookup. Two separate single-column indexes would require a bitmap index scan merge, which is less efficient.

---

## 5. JPA/Hibernate Mapping Notes

### 5.1 Enum Mapping

All Java enum fields (`Role`, `Environment`, `DeploymentStatus`) must be annotated with:

```java
@Enumerated(EnumType.STRING)
```

**Why `STRING` and not `ORDINAL`?** The default `EnumType.ORDINAL` stores the zero-based position of the enum constant. If a new constant is inserted between existing ones (e.g., adding `PAUSED` between `TESTING` and `DEPLOYING`), the ordinals of all subsequent values shift, corrupting all existing rows silently. `EnumType.STRING` stores the constant name (`"DEPLOYING"`, `"SUCCESSFUL"`) making it immune to enum reordering. The `VARCHAR CHECK` constraint in the DDL provides a second layer of protection at the database level.

### 5.2 Timestamp Annotations

| JPA Annotation | Applied To | Behavior |
|---|---|---|
| `@CreationTimestamp` | `createdAt` (User), `startTime` (Deployment), `rollbackTime` (Rollback), `timestamp` (AuditLog) | Set by Hibernate at `INSERT`; ignored on subsequent `UPDATE` |
| `@UpdateTimestamp` | **NOT used on `endTime`** | See note below |

**Why `@UpdateTimestamp` is NOT used on `endTime`:** `@UpdateTimestamp` updates the column on every `UPDATE` to the entity, regardless of whether a terminal state was reached. `endTime` must be `NULL` while the deployment is active and must be set precisely when the service determines the deployment has reached a terminal state (`SUCCESSFUL`, `FAILED`, or `ROLLED_BACK`). The `DeploymentService.updateStatus()` method sets `endTime` explicitly before calling `deploymentRepository.save()`.

### 5.3 Column Name Mapping

Spring Boot's default `ImplicitNamingStrategy` maps `camelCase` Java field names to `snake_case` column names (e.g., `projectName` → `project_name`). Where the mapping is not obvious or differs from default, explicit `@Column(name = "...")` annotations are added for clarity:

```java
@Column(name = "project_name", nullable = false, length = 100)
private String projectName;

@Column(name = "deployed_by", nullable = false, length = 50)
private String deployedBy;

@Column(name = "start_time", nullable = false)
private LocalDateTime startTime;

@Column(name = "end_time")
private LocalDateTime endTime;
```

### 5.4 Relationship Mapping

**Rollback → Deployment (ManyToOne):**

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "deployment_id", nullable = false)
private Deployment deployment;
```

Lazy fetching (`FetchType.LAZY`) is mandatory here. Loading all `Rollback` entities eagerly would trigger N+1 queries in the deployment list endpoint (one extra query per deployment row to check for rollbacks). The service layer fetches rollbacks only when the detail view explicitly requests them, using `rollbackRepository.findByDeploymentId(id)`.

### 5.5 `ddl-auto` Configuration per Environment

| Profile | `spring.jpa.hibernate.ddl-auto` | Schema Management |
|---|---|---|
| `local` (developer) | `update` | Hibernate applies incremental schema changes on startup; safe for local iteration only — see warning below |
| `test` | `create-drop` | Schema is created before each test run and dropped after; Flyway is disabled in test profile |
| `production` | `validate` | Hibernate validates entity mappings against the existing schema; Flyway manages all DDL changes |

**Never use `create` or `create-drop` in production.** `validate` will fail on startup if the database schema does not match the entity mappings, providing an early warning of a missed migration rather than silently dropping production data.

> **Warning:** Do NOT use `update` on a shared or team database — Hibernate's `update` mode can silently drop columns and does not run Flyway migrations. For team environments, use `validate` with Flyway enabled.

---

## 6. Flyway Migration Strategy

Flyway manages all production schema changes as versioned, immutable SQL scripts.

### 6.1 Migration Script Location

```
src/main/resources/db/migration/
```

Flyway's default classpath scan picks up all files in this directory automatically. No additional configuration is required beyond adding the Flyway dependency to `pom.xml`.

### 6.2 Naming Convention

```
V{version}__{description}.sql
```

- `V` — mandatory prefix (uppercase)
- `{version}` — integer version number (1, 2, 3, …); no padding required but consistent padding (01, 02, …) is acceptable
- `__` — **two underscores** (Flyway uses this as the separator between version and description)
- `{description}` — snake_case description of what the script does
- `.sql` — mandatory extension

### 6.3 Migration Files

| File | Purpose |
|---|---|
| `V1__create_users_table.sql` | Creates the `users` table |
| `V2__create_deployments_table.sql` | Creates the `deployments` table |
| `V3__create_rollbacks_table.sql` | Creates the `rollbacks` table with FK constraint |
| `V4__create_audit_logs_table.sql` | Creates the `audit_logs` table |
| `V5__create_indexes.sql` | Creates all performance indexes (see Section 4) |
| `V6__seed_initial_data.sql` | Seeds the initial ADMIN user and sample data for development (profile-guarded) |

### 6.4 Key Rules

**Rule 1: Never modify an existing migration file.**
Flyway checksums each migration on first run and stores the checksum in the `flyway_schema_history` table. If an existing file is modified, Flyway will detect the checksum mismatch and refuse to start the application. Always add a new migration file for schema changes.

**Rule 2: New migrations always get the next version number.**
If the highest existing version is `V6`, the next migration must be `V7`. Gaps are permitted but not recommended. Out-of-order versions require `spring.flyway.out-of-order=true` to be enabled; keep this setting `false` in production.

**Rule 3: Each migration script must be idempotent where possible.**
Use `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, and `INSERT ... ON CONFLICT DO NOTHING` in migration scripts to allow safe re-runs in edge cases (e.g., a partially applied migration on a clean database restart).

**Rule 4: Test profile disables Flyway.**
The test profile sets `spring.flyway.enabled=false` and uses `ddl-auto=create-drop` with H2 in-memory database. Flyway migrations use PostgreSQL-specific syntax (e.g., `BIGSERIAL`) that H2 does not support. Test schema is derived from JPA entity annotations, not SQL scripts.

### 6.5 Application Properties Configuration

```yaml
# application.yml (production)
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: false
  jpa:
    hibernate:
      ddl-auto: validate

# application-test.yml
spring:
  flyway:
    enabled: false
  jpa:
    hibernate:
      ddl-auto: create-drop
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
    driver-class-name: org.h2.Driver
```

---

## 7. Sample Seed Data

Seed data is applied by `V6__seed_initial_data.sql`. In production, this migration should be applied only once in a controlled manner. For local development, re-running is safe because all inserts use `ON CONFLICT DO NOTHING`.

### 7.1 Users

Passwords are BCrypt-hashed with strength factor 12. The plain-text values are shown in comments for local development convenience only — they must never appear in production documentation or be committed without the surrounding hash.

```sql
-- V6__seed_initial_data.sql
-- Users (passwords are BCrypt hashes; plain-text shown in comments for local dev only)
INSERT INTO users (username, email, password, role) VALUES
  ('admin',     'admin@enterprise.com',     '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewFX5L2fXhwj3Ymy', 'ADMIN'),          -- admin123
  ('devops',    'devops@enterprise.com',    '$2a$12$KmJz1c2yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewFX5L2fXhwj3Ymy', 'DEVOPS_ENGINEER'), -- devops123
  ('developer', 'developer@enterprise.com', '$2a$12$NQr3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewFX5L2fXhwj3Ymy', 'DEVELOPER')        -- dev123
ON CONFLICT (username) DO NOTHING;
```

### 7.2 Deployments

Sample deployments span all four environments and several statuses to provide a realistic dashboard view from day one.

```sql
-- Deployments (across environments and statuses)
INSERT INTO deployments (project_name, version, environment, status, deployed_by, start_time, end_time, remarks) VALUES
  ('payment-service',  '1.4.2', 'PRODUCTION', 'SUCCESSFUL', 'devops',     NOW() - INTERVAL '3 days',        NOW() - INTERVAL '3 days' + INTERVAL '12 minutes', 'Production release Q3'),
  ('auth-service',     '2.1.0', 'STAGING',    'DEPLOYING',  'devops',     NOW() - INTERVAL '1 hour',        NULL,                                               'Staging validation'),
  ('user-api',         '0.9.5', 'DEV',        'FAILED',     'developer',  NOW() - INTERVAL '2 hours',       NOW() - INTERVAL '1 hour 45 minutes',               'Test environment — build error'),
  ('notification-svc', '1.0.0', 'QA',         'TESTING',    'devops',     NOW() - INTERVAL '30 minutes',    NULL,                                               'QA regression suite'),
  ('api-gateway',      '3.2.1', 'STAGING',    'PENDING',    'devops',     NOW() - INTERVAL '5 minutes',     NULL,                                               'Scheduled maintenance window');
```

### 7.3 Rollbacks

One rollback is seeded for the `FAILED` `user-api` deployment (deployment ID 3 in the default seed order).

```sql
-- Rollbacks (references the FAILED user-api deployment, resolved by subquery to avoid hard-coded ID dependency)
INSERT INTO rollbacks (deployment_id, previous_version, rollback_reason, rollback_time) VALUES
((SELECT id FROM deployments WHERE project_name = 'user-api' AND version = '0.9.5' LIMIT 1),
 '0.9.4',
 'Build error in user-api 0.9.5: NullPointerException in UserController line 42',
 NOW() - INTERVAL '1 hour 40 minutes');
```

### 7.4 Audit Logs

Audit entries recreate a realistic event sequence covering login, deployment creation, status transitions, and a rollback trigger.

```sql
-- Audit Logs (recreates a realistic event sequence)
INSERT INTO audit_logs (username, action, timestamp, description) VALUES
  ('admin',     'LOGIN_SUCCESS',             NOW() - INTERVAL '4 days',               'User authenticated successfully'),
  ('devops',    'LOGIN_SUCCESS',             NOW() - INTERVAL '3 days 1 hour',        'User authenticated successfully'),
  ('devops',    'DEPLOYMENT_CREATED',        NOW() - INTERVAL '3 days',               'Created deployment: payment-service v1.4.2 to PRODUCTION'),
  ('devops',    'DEPLOYMENT_STATUS_UPDATED', NOW() - INTERVAL '3 days',               'Status changed: payment-service PENDING → BUILDING'),
  ('devops',    'DEPLOYMENT_STATUS_UPDATED', NOW() - INTERVAL '3 days',               'Status changed: payment-service BUILDING → TESTING'),
  ('devops',    'DEPLOYMENT_STATUS_UPDATED', NOW() - INTERVAL '3 days',               'Status changed: payment-service TESTING → DEPLOYING'),
  ('devops',    'DEPLOYMENT_STATUS_UPDATED', NOW() - INTERVAL '3 days',               'Status changed: payment-service DEPLOYING → SUCCESSFUL'),
  ('developer', 'DEPLOYMENT_CREATED',        NOW() - INTERVAL '2 hours',              'Created deployment: user-api v0.9.5 to DEV'),
  ('developer', 'DEPLOYMENT_STATUS_UPDATED', NOW() - INTERVAL '1 hour 50 minutes',   'Status changed: user-api PENDING → FAILED'),
  ('devops',    'ROLLBACK_TRIGGERED',        NOW() - INTERVAL '1 hour 40 minutes',    'Rollback initiated: user-api v0.9.5 → v0.9.4, reason: Build error');
```

### 7.5 Seed Data Integrity Checklist

Before applying seed data to any shared environment, verify:

- [ ] BCrypt hashes in the users insert are valid (test by logging in with the plain-text credentials via `POST /api/auth/login`)
- [ ] Deployment IDs in the rollbacks insert match the actual auto-generated IDs (use `SELECT id FROM deployments WHERE project_name = 'user-api' AND version = '0.9.5'` to confirm)
- [ ] All `ON CONFLICT DO NOTHING` clauses are present so re-running the migration is safe
- [ ] `V6__seed_initial_data.sql` is not applied to a production environment unless it is the initial bootstrap run
