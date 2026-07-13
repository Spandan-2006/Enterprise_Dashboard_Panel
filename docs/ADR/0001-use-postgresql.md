# ADR-0001: Use PostgreSQL as the Primary Database

## Status
Accepted

## Context
The Enterprise Deployment Dashboard requires a relational database supporting:
- ACID transactions (critical for the deployment-status + audit-log write atomicity requirement)
- Complex queries across 4 related entities (users, deployments, rollbacks, audit_logs)
- Enum-like value constraints for status, environment, and role fields
- Strong Java/Spring ecosystem integration
- Reliable Docker image for containerized local development

## Decision
PostgreSQL 15 with Spring Data JPA and Hibernate ORM.

## Rationale
- Strong Spring Data JPA + Hibernate integration (dialect, native enum support)
- Native `VARCHAR CHECK` constraint support for enum-like columns (no separate ENUM type required, making future value additions easier)
- JSONB data type available for future unstructured metadata
- Excellent Docker image (`postgres:15-alpine`) — small, stable, and widely used
- Better write-concurrency than MySQL on multi-user workloads (MVCC with no row-level gap locking issues)
- Superior advisory lock support for deployment coordination
- Active maintenance with 5-year LTS lifecycle for version 15

## Consequences

### Positive
- Spring Boot auto-configuration works out of the box with `spring-boot-starter-data-jpa` + `postgresql` driver
- Flyway has first-class PostgreSQL support
- Robust enum-safe migration path: VARCHAR + CHECK avoids the MySQL ENUM type migration problem

### Negative / Trade-offs
- Team needs PostgreSQL operational knowledge (vs. the more widely known MySQL)
- H2 is used for tests (profile `test`), meaning schema compatibility must be maintained across PostgreSQL 15 and H2 — avoid PostgreSQL-specific SQL in migration files
- Requires a running PostgreSQL instance for local development (Docker Compose mitigates this)

## Alternatives Considered

| Alternative | Pros | Cons | Reason Rejected |
|---|---|---|---|
| MySQL 8 | Widely known, good Spring support | Weaker JSONB support, different locking semantics, ENUM type migration pain | PostgreSQL's advisory locks, JSONB, and MVCC are better suited to this use case |
| H2 (embedded) | Zero setup, fast startup | Not suitable for production, lacks PostgreSQL features, different SQL dialect can mask bugs | Production database must be persistent and feature-complete |
| MongoDB | Schema flexibility, good for document storage | Relationships between entities are relational, not document-oriented; weaker transactional guarantees across collections | Relational model is a better fit for user/deployment/rollback/audit entities with FK constraints |
