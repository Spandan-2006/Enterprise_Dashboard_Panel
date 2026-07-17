![Build Status](https://github.com/Spandan-2006/Enterprise_Dashboard_Panel/actions/workflows/ci.yml/badge.svg)
![Java 21](https://img.shields.io/badge/Java-21-blue?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen?logo=springboot)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-blue?logo=postgresql)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

# Enterprise Deployment Dashboard

A full-stack enterprise-grade deployment tracking and management dashboard with role-based access control, audit logging, and rollback support.

---

## Documentation

| Document | What it covers |
|----------|---------------|
| [docs/PRD.md](docs/PRD.md) | Problem statement, user stories, functional & non-functional requirements, success metrics |
| [docs/TRD.md](docs/TRD.md) | Tech stack, layered architecture, security design, environment config, testing strategy, ADRs |
| [docs/APP_FLOW.md](docs/APP_FLOW.md) | Screen-by-screen user flows, route structure, deployment state machine, error handling |
| [docs/UI_UX_BRIEF.md](docs/UI_UX_BRIEF.md) | React component hierarchy, status badge design, API layer, auth context, responsive layout |
| [docs/BACKEND_SCHEMA.md](docs/BACKEND_SCHEMA.md) | DB schema (SQL), JPA entities, DTOs, service interfaces, full REST API contract |
| [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md) | Phased build order (Phase 0–7), coding standards, git workflow, definition of done |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Java 21 |
| Framework | Spring Boot 3.x |
| Persistence | Spring Data JPA + PostgreSQL 15 |
| Security | Spring Security 6 + JWT (JJWT 0.12) + BCrypt |
| Schema Migrations | Flyway |
| Build | Gradle 8.x (Kotlin DSL) |
| Containerization | Docker + Docker Compose |
| CI/CD | GitHub Actions |
| API Docs | Swagger / OpenAPI 3.0 (Springdoc) |
| Testing | JUnit 5 + Mockito + JaCoCo (80% coverage gate) |
| Frontend (Optional) | React 18 + Vite 5 + Tailwind CSS 3 + Chart.js + Axios |

---

## Architecture Overview

Strict layered (clean) architecture — each layer communicates only with the layer directly below it.

```
[React Frontend (Optional)] ──HTTP/JSON──► [Spring MVC Controllers]
                                                      │
                                                      ▼
                                         [Service Layer — Business Logic]
                                                      │
                                                      ▼
                                         [Repository Layer — Spring Data JPA]
                                                      │
                                                      ▼
                                              [PostgreSQL 15]
```

- **Controllers** — HTTP concerns only: request parsing, validation delegation, response formatting via `ApiResponse<T>`
- **Services** — all business logic; orchestrate repository calls and audit logging
- **Repositories** — Spring Data JPA interfaces; dynamic queries via `Specification` API
- **Security** — stateless JWT filter chain + method-level `@PreAuthorize` annotations

---

## Getting Started

### Prerequisites

- **Java 21** — [Download](https://adoptium.net/)
- **Docker 24+** and **Docker Compose v2** — [Download](https://docs.docker.com/get-docker/)
- **Gradle 8.x** — included via wrapper (`./gradlew`), no separate install needed

### Clone & Run

```bash
git clone https://github.com/Spandan-2006/Enterprise_Dashboard_Panel.git
cd Enterprise_Dashboard_Panel
docker-compose up --build
```

This starts the Spring Boot application and a PostgreSQL 15 container. Flyway applies all migrations automatically on first start.

To run tests without Docker:

```bash
./gradlew test
```

### Verify the Application

```bash
curl http://localhost:8080/actuator/health
# Expected: {"status":"UP"}
```

### Explore the API

```
http://localhost:8080/swagger-ui/index.html
```

---

## Default Local Credentials

> **Warning: for local development only — never use in staging or production.**

| Username | Password | Role |
|----------|---------|------|
| `admin` | `admin123` | `ADMIN` |
| `devops` | `devops123` | `DEVOPS_ENGINEER` |
| `developer` | `dev123` | `DEVELOPER` |

Seed accounts are loaded by `V6__seed_initial_data.sql` on first start. They are never loaded in `staging` or `production` profiles.

---

## Key Environment Variables

The full reference is in [docs/TRD.md](docs/TRD.md#5-configuration--environments).

| Variable | Purpose | Default (dev only) |
|----------|---------|-------------------|
| `DB_URL` | JDBC connection URL | `jdbc:postgresql://localhost:5432/dashboard` |
| `DB_USERNAME` | Database username | `dashboard` |
| `DB_PASSWORD` | Database password | `dashboard` |
| `JWT_SECRET` | HS256 signing secret (min 256 bits, Base64-encoded) | *(must be set explicitly)* |
| `JWT_EXPIRY_MS` | Access token TTL in milliseconds | `86400000` (24h) |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed origins | `http://localhost:3000` |

---

## Roles & Permissions

| Action | ADMIN | DEVOPS_ENGINEER | DEVELOPER |
|--------|-------|-----------------|-----------|
| View deployments & dashboard | Yes | Yes | Yes |
| Create / update deployments | Yes | Yes | No |
| Trigger rollbacks | Yes | Yes | No |
| Delete deployments | Yes | No | No |
| Manage users | Yes | No | No |
| View audit logs | Yes | No | No |

Full authorization matrix: [docs/TRD.md — Authorization Matrix](docs/TRD.md#authorization-matrix)

---

## Contributing

Before opening a pull request, review:

- **[docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md)** — coding standards, package structure, naming conventions, git workflow
- **[docs/TRD.md](docs/TRD.md)** — architecture rules and security requirements
- All PRs require an approving review, a green CI pipeline (tests + ≥ 80% coverage), and documentation updated in the same PR as the code change

---

## License

This project is licensed under the [MIT License](LICENSE).
