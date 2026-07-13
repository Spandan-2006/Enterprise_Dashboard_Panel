![Build Status](https://github.com/your-org/Enterprise_Dashboard_Panel/actions/workflows/ci.yml/badge.svg)
![Java 21](https://img.shields.io/badge/Java-21-blue?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen?logo=springboot)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue?logo=postgresql)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

# Enterprise Deployment Dashboard

A full-stack enterprise-grade deployment tracking and management dashboard with role-based access control, audit logging, and rollback support.

---

## Quick Links

| Document | Description |
|---|---|
| [docs/01-prd.md](docs/01-prd.md) | Product Requirements Document — full feature specification and acceptance criteria |
| [docs/02-architecture.md](docs/02-architecture.md) | System architecture decisions, layer responsibilities, and component diagrams |
| [docs/03-domain-model.md](docs/03-domain-model.md) | Core domain entities, relationships, and business rules |
| [docs/04-data-model.md](docs/04-data-model.md) | Database schema, table definitions, indexes, and constraints |
| [docs/05-security.md](docs/05-security.md) | Authentication flow, JWT configuration, RBAC policy, and secure coding guidelines |
| [docs/06-api-spec.md](docs/06-api-spec.md) | Full REST API reference with request/response shapes and status codes |
| [docs/07-database.md](docs/07-database.md) | Database setup, migration strategy, and query optimization notes |
| [docs/08-deployment.md](docs/08-deployment.md) | Deployment procedures for dev, staging, and production environments |
| [docs/09-testing.md](docs/09-testing.md) | Testing strategy, coverage targets, and how to run the test suite |
| [docs/10-docker.md](docs/10-docker.md) | Docker and Docker Compose setup, image build instructions, and compose profiles |
| [docs/11-ci-cd.md](docs/11-ci-cd.md) | GitHub Actions pipeline stages, secrets required, and deployment gates |
| [docs/12-monitoring.md](docs/12-monitoring.md) | Actuator endpoints, logging configuration, and observability setup |
| [docs/13-rollback.md](docs/13-rollback.md) | Rollback workflow, trigger conditions, and history retention policy |
| [docs/14-coding-standards.md](docs/14-coding-standards.md) | Java style guide, naming conventions, and code review checklist |
| [docs/15-git-workflow.md](docs/15-git-workflow.md) | Branching strategy, commit message format, and PR process |
| [docs/16-error-handling.md](docs/16-error-handling.md) | Global exception handling, error response format, and status code mapping |
| [docs/17-audit-logging.md](docs/17-audit-logging.md) | Audit log schema, captured events, and retention rules |
| [docs/18-environments.md](docs/18-environments.md) | Environment variable reference for all configuration properties |
| [docs/ADR/](docs/ADR/) | Architecture Decision Records — one file per significant technical decision |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Java 21 |
| Framework | Spring Boot 3.x |
| Persistence | Spring Data JPA + PostgreSQL |
| Security | Spring Security + JWT + BCrypt |
| Build | Gradle 8.x |
| Containerization | Docker + Docker Compose |
| CI/CD | GitHub Actions |
| API Docs | Swagger / OpenAPI 3.0 |
| Testing | JUnit 5 + Mockito |
| Frontend (Optional) | React 18 + Tailwind CSS + Chart.js + Axios |

---

## Architecture Overview

The application follows a strict layered (clean) architecture. Each layer has a single responsibility and communicates only with the layer immediately below it.

```
[React Frontend (Optional)] --> HTTP/JSON --> [Spring MVC Controllers]
                                                        |
                                                        v
                                          [Service Layer (Business Logic)]
                                                        |
                                                        v
                                          [Repository Layer (Spring Data JPA)]
                                                        |
                                                        v
                                          [PostgreSQL Database]
```

- **Controllers** handle HTTP concerns only: request parsing, validation delegation, and response formatting via `ApiResponse<T>`.
- **Services** contain all business logic and orchestrate calls to repositories and other services.
- **Repositories** are Spring Data JPA interfaces; custom queries use JPQL or named queries, never raw SQL in service code.
- **Security** is enforced at the controller boundary via Spring Security filter chains and method-level `@PreAuthorize` annotations.

---

## Getting Started

### Prerequisites

- **Java 21** — [Download](https://adoptium.net/)
- **Docker 24+** and **Docker Compose v2** — [Download](https://docs.docker.com/get-docker/)
- **Gradle 8.x** — included via the Gradle wrapper (`./gradlew`), no separate install needed

### Clone the Repository

```bash
git clone https://github.com/your-org/Enterprise_Dashboard_Panel.git
cd Enterprise_Dashboard_Panel
```

### Start the Full Stack

```bash
docker-compose up --build
```

This starts the Spring Boot application and a PostgreSQL 16 container. The database schema is applied automatically on first start.

### Verify the Application is Running

```bash
curl http://localhost:8080/actuator/health
# Expected: {"status":"UP"}
```

### Explore the API

Open the Swagger UI in your browser:

```
http://localhost:8080/swagger-ui.html
```

---

## Default Local Credentials

> **Warning: for local development only — never use these credentials in staging or production.**

| Username | Password | Role |
|---|---|---|
| `admin` | `admin123` | `ADMIN` |
| `devops` | `devops123` | `DEVOPS_ENGINEER` |
| `developer` | `dev123` | `DEVELOPER` |

These seed accounts are loaded by the `DataInitializer` component when the `spring.profiles.active` profile includes `local` or `dev`. They are never loaded in `staging` or `production` profiles.

---

## Environment Variables

The application is configured entirely through environment variables in non-local environments. The full reference — including variable names, default values, required/optional status, and which profile each applies to — is documented in [docs/18-environments.md](docs/18-environments.md).

The most critical variables for getting started:

| Variable | Purpose | Default (local only) |
|---|---|---|
| `DB_URL` | JDBC connection URL | `jdbc:postgresql://localhost:5432/dashboard` |
| `DB_USERNAME` | Database username | `dashboard` |
| `DB_PASSWORD` | Database password | `dashboard` |
| `JWT_SECRET` | HS256 signing secret (min 256 bits) | *(must be set explicitly)* |
| `JWT_EXPIRATION_MS` | Access token TTL in milliseconds | `86400000` (24 h) |

---

## Contributing

Before opening a pull request, review:

- **[docs/14-coding-standards.md](docs/14-coding-standards.md)** — Java style conventions, package layout, DTO/entity boundaries, and the code review checklist.
- **[docs/15-git-workflow.md](docs/15-git-workflow.md)** — Branching model (feature branches off `main`), commit message format (`type(scope): message`), and PR requirements (tests must pass, coverage must not regress).

All pull requests require at least one approving review and a green CI pipeline before merge.

---

## License

This project is licensed under the [MIT License](LICENSE).
