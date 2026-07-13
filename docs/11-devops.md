# docs/11-devops.md — DevOps

## 1. CI/CD Pipeline Overview

Every push to a feature or develop branch triggers the GitHub Actions CI pipeline, which compiles the code, runs all tests, enforces the coverage gate, and builds a Docker image. The resulting image is pushed to the GitHub Container Registry only when the triggering ref is `main` or `develop`.

```
Developer pushes to feature/* or develop branch
        ↓
GitHub Actions: ci.yml triggers
        ↓
┌─────────────────────────────────┐
│  Step 1: Checkout code          │  actions/checkout@v4
│  Step 2: Setup Java 21          │  actions/setup-java@v4 (temurin)
│  Step 3: Cache Gradle           │  actions/cache@v4 (key: gradle-wrapper)
│  Step 4: ./gradlew build        │  compile + package
│  Step 5: ./gradlew test         │  unit + integration tests
│  Step 6: JaCoCo coverage gate   │  fails if < 80% line coverage
│  Step 7: Build Docker image     │  docker/build-push-action@v5
│  Step 8: Push to registry       │  ghcr.io/Spandan-2006/enterprise-dashboard
└─────────────────────────────────┘
        ↓ (only on push to main or develop)
Artifacts available for deployment
```

The pipeline is intentionally kept linear. No parallel jobs are defined in v1 — the build and test steps run sequentially to keep the workflow simple and easy to debug. Parallelism can be introduced in v2 when build times become a concern.

---

## 2. GitHub Actions Workflow Files

### `.github/workflows/ci.yml`

Runs on every push to `main`, `develop`, or any `feature/**` branch, and on pull requests targeting `main` or `develop`.

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: gradle

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build
        run: ./gradlew build -x test

      - name: Test
        run: ./gradlew test

      - name: Coverage verification
        run: ./gradlew jacocoTestCoverageVerification

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/reports/tests/

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: ghcr.io/spandan-2006/enterprise-dashboard:${{ github.sha }}
```

**Notes:**
- `cache: gradle` on `setup-java@v4` caches both the Gradle wrapper and the dependency cache, removing the need for a separate `actions/cache` step.
- `build -x test` is a fast pre-check that fails immediately on compilation errors before running the slower test suite.
- `push: false` on the CI job intentionally does not push the image — image push only happens in `release.yml`.
- Test results are uploaded even when the build fails (`if: always()`) so that engineers can inspect JUnit XML reports from failing runs.

---

### `.github/workflows/release.yml`

Triggers when a semantic version tag (`v*.*.*`) is pushed. Builds the fat JAR, logs in to GitHub Container Registry, and publishes the image with both the version tag and `latest`.

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: gradle

      - name: Build JAR
        run: |
          chmod +x gradlew
          ./gradlew bootJar

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/spandan-2006/enterprise-dashboard:${{ github.ref_name }}
            ghcr.io/spandan-2006/enterprise-dashboard:latest
```

**Notes:**
- `permissions: packages: write` is required to push to GitHub Container Registry (GHCR) when using the automatic `GITHUB_TOKEN`. Without this permission declaration the push will be rejected.
- `github.ref_name` evaluates to the tag name (e.g., `v1.2.0`) on a tag push event, making the image tag match the Git tag exactly.
- The `latest` tag is updated on every release. Production deployments should pin to the versioned tag (e.g., `v1.2.0`), not `latest`.

---

## 3. Dockerfile (Multi-stage)

The multi-stage build separates the JDK-heavy build environment from the lean JRE runtime image. The final image is based on Alpine Linux and runs as a non-root user.

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

**Layer caching strategy:** The `COPY gradle/`, `COPY gradlew`, and `RUN ./gradlew dependencies` lines are placed before `COPY src/` deliberately. Docker caches these layers and only re-runs the dependency download when `build.gradle.kts` changes, not on every source code change. This keeps rebuild times fast during development.

**JVM flags:**
- `-XX:+UseContainerSupport` — tells the JVM to read CPU and memory limits from container cgroups rather than the host machine, preventing the JVM from over-allocating heap.
- `-XX:MaxRAMPercentage=75.0` — caps the JVM heap at 75% of the container's memory limit, leaving headroom for the OS, metaspace, and off-heap memory.

**Security:** The runtime image creates a dedicated system user (`appuser`) and group (`appgroup`) and switches to that user before launching the JVM. The container therefore does not run as root.

---

## 4. Docker Compose Configuration

The `docker-compose.yml` file defines two services: the PostgreSQL database and the Spring Boot application. The application service is configured to wait for the database health check to pass before starting, preventing connection errors during startup.

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    container_name: enterprise-db
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-enterprise_dashboard}
      POSTGRES_USER: ${POSTGRES_USER:-dashuser}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-dashuser}"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: .
    container_name: enterprise-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/${POSTGRES_DB:-enterprise_dashboard}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER:-dashuser}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXPIRY_MS: ${JWT_EXPIRY_MS:-86400000}
      JWT_REFRESH_EXPIRY_MS: ${JWT_REFRESH_EXPIRY_MS:-604800000}
      CORS_ALLOWED_ORIGINS: ${CORS_ALLOWED_ORIGINS:-http://localhost:3000}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:

networks:
  default:
    name: enterprise-network
```

**Key design decisions:**

| Decision | Rationale |
|---|---|
| `${VAR:-default}` syntax | Provides safe defaults for non-secret variables without requiring them in `.env` |
| `POSTGRES_PASSWORD` with no default | Forces the operator to explicitly set the password; no accidental empty passwords |
| `condition: service_healthy` | Uses the `pg_isready` healthcheck — ensures PostgreSQL is accepting connections before the app starts, not just that the container is running |
| Named volume `postgres_data` | Survives `docker-compose down`; data is only destroyed with `docker-compose down -v` |
| Named network `enterprise-network` | Makes the network easy to identify in `docker network ls` and simplifies adding additional services later |

---

## 5. .dockerignore

The `.dockerignore` file prevents unnecessary files from being sent to the Docker build context. This reduces build time and prevents accidental inclusion of secrets or large binary files.

```
.git/
build/
docs/
*.md
frontend/node_modules/
.env
.env.*
```

**Why each entry matters:**

- `.git/` — The Git metadata directory can be hundreds of megabytes. Excluding it prevents it from bloating the build context.
- `build/` — Compiled Gradle output from a local build. The Docker build stage runs `./gradlew bootJar` itself, so this directory must not be copied in (it would cause the build to use stale local artifacts).
- `docs/` — Documentation files have no runtime use inside the container.
- `*.md` — README and similar markdown files are not needed at runtime.
- `frontend/node_modules/` — If a frontend is ever added to the repo, its `node_modules` directory must never be sent to the Docker context.
- `.env`, `.env.*` — Local secret files. These must never be baked into the Docker image. Secrets are injected at runtime via environment variables.

---

## 6. Environment Variable Injection

All secrets and environment-specific configuration are injected via environment variables. This approach follows the [12-factor app methodology](https://12factor.net/config) and keeps secrets out of source control.

### Local Development

Create a `.env` file at the project root. Docker Compose automatically reads `.env` when it starts services. The `.env` file is listed in `.gitignore` and must never be committed.

```
POSTGRES_PASSWORD=change_me_local_only
JWT_SECRET=your-minimum-32-character-secret-key-here
```

### CI/CD (GitHub Actions)

In the GitHub repository, navigate to **Settings → Secrets and variables → Actions** and create the following repository secrets:

| Secret Name | Description |
|---|---|
| `POSTGRES_PASSWORD` | Database password used during integration tests |
| `JWT_SECRET` | JWT signing key (minimum 32 characters) |

These secrets are injected as environment variables in the workflow steps that require them. They are never printed in logs.

### Production

Use a secrets manager rather than a flat `.env` file. Recommended options:

- **AWS Secrets Manager** — inject secrets as environment variables via ECS task definitions or EC2 instance profiles.
- **HashiCorp Vault** — use the Vault Agent sidecar to write secrets to the container's environment at startup.
- **Kubernetes Secrets** — mount secrets as environment variables in the pod spec; consider using External Secrets Operator to sync from a cloud secrets manager.

Refer to [docs/18-environments.md](./18-environments.md) for the complete list of all environment variables and their descriptions.
