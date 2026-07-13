# 18 — Environment Configuration

## 1. Environment Overview

| Environment | Purpose | Database | Log Level | Debug Endpoints |
|---|---|---|---|---|
| `local` | Developer machine, single developer | Local PostgreSQL via Docker Compose | DEBUG | All Actuator (no auth required) |
| `dev` | Shared dev server, integration testing | PostgreSQL container | DEBUG | Most Actuator endpoints |
| `staging` | Pre-production validation | Managed PostgreSQL | INFO | `health`, `info`, `metrics`, `loggers` |
| `production` | Live user traffic | Managed PostgreSQL HA | WARN | `health` only |

All environments use the same application JAR. Behaviour differences are controlled entirely through Spring profiles and environment variables — there are no environment-specific builds.

---

## 2. Spring Profiles

### Active Profile

The active profile is set via the `SPRING_PROFILES_ACTIVE` environment variable:

```bash
SPRING_PROFILES_ACTIVE=prod
```

When `SPRING_PROFILES_ACTIVE` is not set, Spring Boot defaults to no active profile, which loads only `application.yml`. This configuration is intentionally incomplete (no database credentials) and will fail to start — an active profile must always be specified.

### Profile File Mapping

| File | Loaded when |
|---|---|
| `application.yml` | Always (base/shared configuration) |
| `application-dev.yml` | `SPRING_PROFILES_ACTIVE=dev` |
| `application-staging.yml` | `SPRING_PROFILES_ACTIVE=staging` |
| `application-prod.yml` | `SPRING_PROFILES_ACTIVE=prod` |

Profile files override or extend the base `application.yml`. Values in a profile file take precedence over base values for the same key.

### Docker Compose

`docker-compose.yml` sets the dev profile automatically:

```yaml
environment:
  - SPRING_PROFILES_ACTIVE=dev
  - POSTGRES_HOST=postgres
  - POSTGRES_DB=enterprise_dashboard
  - POSTGRES_USER=dashuser
  - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
  - JWT_SECRET=${JWT_SECRET}
```

---

## 3. Environment Variables

| Variable | Description | Example Value | Required | Default |
|---|---|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Active Spring profile | `dev` | Yes | — |
| `POSTGRES_HOST` | PostgreSQL hostname | `localhost` | Yes | — |
| `POSTGRES_PORT` | PostgreSQL port | `5432` | No | `5432` |
| `POSTGRES_DB` | Database name | `enterprise_dashboard` | Yes | — |
| `POSTGRES_USER` | Database username | `dashuser` | Yes | — |
| `POSTGRES_PASSWORD` | Database password | — | Yes | — |
| `JWT_SECRET` | JWT signing secret (minimum 32 bytes / 256 bits, base64-encoded) | — | Yes | — |
| `JWT_EXPIRY_MS` | Access token TTL in milliseconds | `86400000` | No | `86400000` |
| `JWT_REFRESH_EXPIRY_MS` | Refresh token TTL in milliseconds | `604800000` | No | `604800000` |
| `SERVER_PORT` | HTTP server port | `8080` | No | `8080` |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed CORS origins | `http://localhost:3000` | No | `*` |
| `LOG_LEVEL` | Root log level override | `INFO` | No | Profile default |
| `SPRING_DATASOURCE_URL` | Full JDBC URL (overrides individual host/port/db variables) | `jdbc:postgresql://localhost:5432/enterprise_dashboard` | No | Built from `POSTGRES_HOST`/`PORT`/`DB` |

**Notes:**

- `POSTGRES_PASSWORD` and `JWT_SECRET` must never be committed to source control. Store them in `.env` (local), GitHub Actions Secrets (CI), or a cloud secret manager (staging/production).
- `SPRING_DATASOURCE_URL` is provided for convenience when connecting through a proxy or socket file. If set, it takes precedence over the individual `POSTGRES_*` variables.
- `CORS_ALLOWED_ORIGINS=*` is acceptable only for local development. Staging and production must enumerate specific origins.

---

## 4. Base Configuration (`application.yml`)

This file is loaded in all environments and defines shared defaults. Profile-specific files override individual keys as needed.

```yaml
server:
  port: ${SERVER_PORT:8080}
  compression:
    enabled: true
    mime-types: application/json
    min-response-size: 1024

spring:
  application:
    name: enterprise-dashboard
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://${POSTGRES_HOST}:${POSTGRES_PORT:5432}/${POSTGRES_DB}}
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: false
  flyway:
    enabled: true
    locations: classpath:db/migration

jwt:
  secret: ${JWT_SECRET}
  expiry-ms: ${JWT_EXPIRY_MS:86400000}
  refresh-expiry-ms: ${JWT_REFRESH_EXPIRY_MS:604800000}

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    root: ${LOG_LEVEL:WARN}
    com.enterprise.dashboard: INFO
```

---

## 5. Profile-Specific Overrides

### `application-dev.yml`

Used for local development and the shared dev server. Relaxed JPA settings, full Actuator access, and verbose logging.

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update        # allow schema auto-update during active development
    show-sql: true
    properties:
      hibernate:
        format_sql: true

management:
  endpoints:
    web:
      exposure:
        include: "*"          # expose all Actuator endpoints

logging:
  level:
    root: INFO
    com.enterprise.dashboard: DEBUG
```

### `application-staging.yml`

Mirrors production as closely as possible. Schema is validated (not auto-updated). Actuator exposes enough for monitoring dashboards but not administrative endpoints.

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers

logging:
  level:
    root: INFO
    com.enterprise.dashboard: INFO
```

### `application-prod.yml`

Minimum surface area. No SQL logging, no unnecessary Actuator endpoints. The `WARN` root log level is set explicitly here even though it matches the base default — this makes intent clear and prevents an accidental base-file change from affecting production.

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: false

management:
  endpoints:
    web:
      exposure:
        include: health

logging:
  level:
    root: WARN
    com.enterprise.dashboard: INFO
```

---

## 6. Secrets Management

| Environment | Secret Storage | How to Supply |
|---|---|---|
| `local` | `.env` file (gitignored) | Copy `.env.example` → `.env`; fill in values before first run |
| `dev` | `.env` on dev server or shell environment variables | Set in shell profile or CI runner environment |
| CI/CD | GitHub Actions Secrets | Reference as `${{ secrets.JWT_SECRET }}` etc. in workflow YAML |
| `staging` | Cloud secret manager (e.g., AWS Secrets Manager, Azure Key Vault) | Injected as environment variables by the container orchestrator at startup |
| `production` | Cloud secret manager, encrypted at rest | Same injection pattern as staging; rotate on a defined schedule |

### `.env.example`

Commit this file to the repository as a template. It must contain only placeholder values — never real secrets.

```bash
SPRING_PROFILES_ACTIVE=dev
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=enterprise_dashboard
POSTGRES_USER=dashuser
POSTGRES_PASSWORD=changeme
JWT_SECRET=REPLACE_WITH_BASE64_SECRET
JWT_EXPIRY_MS=86400000
JWT_REFRESH_EXPIRY_MS=604800000
SERVER_PORT=8080
CORS_ALLOWED_ORIGINS=http://localhost:3000
```

### `.gitignore` Entry

Ensure `.env` is gitignored:

```
.env
*.env.local
```

### Generating a Secure `JWT_SECRET`

The JWT signing secret must be at least 32 bytes (256 bits) when decoded. Use a base64-encoded random value:

```bash
# Linux / macOS
openssl rand -base64 32

# Windows PowerShell
[Convert]::ToBase64String((1..32 | ForEach-Object { [byte](Get-Random -Maximum 256) }))
```

The output is a 44-character base64 string. Set this as the value of `JWT_SECRET`.

### Secret Rotation

When rotating `JWT_SECRET` in production:

1. Generate a new secret value.
2. Update the secret in the cloud secret manager.
3. Perform a rolling restart of the application containers (existing tokens signed with the old secret will be invalidated — users must re-authenticate).
4. If zero-downtime rotation is required, implement dual-key validation: accept tokens signed by either the current or previous secret for a transition window (e.g., 15 minutes), then remove the old key.
