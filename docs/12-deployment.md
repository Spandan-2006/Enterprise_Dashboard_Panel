# docs/12-deployment.md — Deployment Guide

## 1. Prerequisites Checklist

Complete all items in this checklist before attempting to deploy the application. Missing prerequisites are the most common cause of failed first-time deployments.

- [ ] Docker Desktop 24+ installed and running
- [ ] Docker Compose v2.x included with Docker Desktop
- [ ] Minimum 2 GB free RAM available on the host machine
- [ ] Ports 8080 and 5432 are not already in use by another process
- [ ] Git installed (any recent version)
- [ ] (Optional — required only for frontend development) Node.js 18+

**Verify Docker is running:**

```bash
docker --version      # should print Docker version 24.x.x or higher
docker compose version  # should print Docker Compose version v2.x.x
```

**Check that required ports are free (Linux/macOS):**

```bash
lsof -i :8080
lsof -i :5432
```

If either command returns output, a process is already using that port. Stop it before proceeding.

**Check that required ports are free (Windows PowerShell):**

```powershell
netstat -ano | findstr :8080
netstat -ano | findstr :5432
```

---

## 2. Local Deployment Steps

Follow these steps in order. Each step must succeed before proceeding to the next.

### Step 1 — Clone the repository

```bash
git clone https://github.com/Spandan-2006/Enterprise_Dashboard_Panel.git
```

### Step 2 — Navigate to the project root

```bash
cd Enterprise_Dashboard_Panel
```

All subsequent commands must be run from this directory.

### Step 3 — Create the `.env` file from template

Copy the template from Section 3 of this document into a new file named `.env` at the project root, then fill in the required values.

```bash
# Linux/macOS
cp .env.example .env   # if .env.example exists, otherwise create manually
nano .env              # or use any text editor

# Windows
copy .env.example .env
notepad .env
```

At minimum, set `POSTGRES_PASSWORD` and `JWT_SECRET`. See Section 3 for the full template.

### Step 4 — Start all services

```bash
docker compose up --build
```

This command:
1. Builds the Spring Boot Docker image using the multi-stage `Dockerfile`.
2. Pulls the `postgres:15-alpine` image from Docker Hub (first run only).
3. Starts the `enterprise-db` (PostgreSQL) container.
4. Waits for the PostgreSQL health check to pass.
5. Starts the `enterprise-app` (Spring Boot) container.
6. Runs Flyway database migrations automatically on application startup.

**To run in the background (detached mode):**

```bash
docker compose up --build -d
docker compose logs -f app   # then tail logs separately
```

### Step 5 — Wait for health checks to pass

Watch the logs for the following line, which indicates the Spring Boot application has started successfully:

```
enterprise-app | Started EnterpriseDashboardApplication in X.XXX seconds
```

The initial startup typically takes 30–60 seconds on the first run as Docker downloads base images and Gradle resolves dependencies.

### Step 6 — Verify application health

```bash
curl http://localhost:8080/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

If the status is `DOWN` or `OUT_OF_SERVICE`, check the logs: `docker compose logs app`.

### Step 7 — Open Swagger UI

Navigate to the following URL in a browser:

```
http://localhost:8080/swagger-ui/index.html
```

The Swagger UI lists all available API endpoints with request/response schemas and allows interactive testing directly from the browser.

### Step 8 — Login with seed credentials

Use the Swagger UI or a REST client (curl, Postman, Insomnia) to authenticate:

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

Expected response (abbreviated):

```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "eyJhbGciOi...",
  "tokenType": "Bearer",
  "expiresIn": 86400000
}
```

Copy the `accessToken` value and use it as a Bearer token in the `Authorization` header for subsequent requests.

**Default seed users:**

| Username | Password | Role |
|---|---|---|
| `admin` | `admin123` | ADMIN |
| `manager` | `manager123` | MANAGER |
| `developer` | `developer123` | DEVELOPER |

> **Security note:** Change these seed passwords immediately in any environment that is accessible from outside localhost.

---

## 3. `.env` File Template

Create a file named `.env` at the project root with the following content. Replace placeholder values before starting the application.

```
# ─────────────────────────────────────────────────────
# Database Configuration
# ─────────────────────────────────────────────────────
POSTGRES_DB=enterprise_dashboard
POSTGRES_USER=dashuser
POSTGRES_PASSWORD=change_me_local_only

# ─────────────────────────────────────────────────────
# JWT Configuration
# ─────────────────────────────────────────────────────
# Must be at least 32 characters; use a Base64-encoded random string
# Generate with: openssl rand -base64 32
JWT_SECRET=your-minimum-32-character-secret-key-here-base64encoded
JWT_EXPIRY_MS=86400000
JWT_REFRESH_EXPIRY_MS=604800000

# ─────────────────────────────────────────────────────
# Application Configuration
# ─────────────────────────────────────────────────────
CORS_ALLOWED_ORIGINS=http://localhost:3000
SERVER_PORT=8080
```

**Generating a secure `JWT_SECRET`:**

```bash
# Linux/macOS
openssl rand -base64 32

# Windows PowerShell
[Convert]::ToBase64String((1..32 | ForEach-Object { [byte](Get-Random -Max 256) }))
```

> **WARNING:** Never commit `.env` to Git. The file is listed in `.gitignore`. If you accidentally commit it, rotate all secrets immediately using your secrets manager or GitHub repository settings.

---

## 4. Health Check Endpoints

The following endpoints are provided by Spring Boot Actuator and are available without authentication.

| Endpoint | Purpose | Expected Response |
|---|---|---|
| `GET /actuator/health` | Overall application liveness and database connectivity | `{"status":"UP"}` |
| `GET /actuator/info` | Application name, version, and build info | `{"app":{"name":"enterprise-dashboard","version":"1.0.0"}}` |
| `GET /actuator/metrics` | JVM heap, HTTP latency, HikariCP pool metrics | Prometheus-compatible metrics output |

**Health check response fields:**

The `/actuator/health` endpoint returns the aggregate status of all health indicators. The `db` indicator checks that a JDBC connection to PostgreSQL can be obtained and a simple query executed:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP"
    }
  }
}
```

If the database is unreachable, the `db` component will show `DOWN` and the aggregate `status` will also be `DOWN`. In Docker Compose, the application container health check uses this endpoint to determine whether the container is healthy.

---

## 5. Production Deployment Considerations

The Docker Compose setup described in this document is suitable for local development and functional testing. The following changes are required before deploying to a production environment.

### Secrets Management

Do not use a `.env` file in production. Use a dedicated secrets manager:

- **AWS Secrets Manager** — store secrets as key-value pairs and inject them as environment variables via ECS task definitions, Lambda environment variables, or EC2 instance IAM role policies.
- **HashiCorp Vault** — use the Vault Agent sidecar or the Spring Vault integration (`spring-cloud-vault`) to inject secrets at startup.
- **GCP Secret Manager** — access secrets from GCP Workload Identity without static credentials.

### TLS / HTTPS

Do not terminate TLS inside the Spring Boot application. Place a TLS-terminating load balancer or reverse proxy in front of the container:

- AWS Application Load Balancer (ALB) with an ACM certificate
- GCP Cloud Load Balancing
- nginx reverse proxy with a Let's Encrypt certificate (self-managed)

The Spring Boot application continues to listen on HTTP port 8080 behind the load balancer.

### Database

Replace the containerized PostgreSQL service with a managed database:

- **AWS RDS for PostgreSQL 15** — automated backups, Multi-AZ failover, managed patches.
- **GCP Cloud SQL for PostgreSQL** — similar managed features on GCP.
- **Azure Database for PostgreSQL** — on Azure.

Update `SPRING_DATASOURCE_URL` to point to the managed database endpoint.

### Docker Image Tags

Never use the `latest` tag in production. Pin to a specific semantic version tag:

```yaml
image: ghcr.io/spandan-2006/enterprise-dashboard:v1.2.0
```

This ensures that a `docker compose up -d` after an inadvertent `latest` push does not silently upgrade the running application.

### Log Aggregation

Configure a log aggregation solution so that logs persist beyond container restarts:

- **AWS CloudWatch Logs** — use the `awslogs` Docker log driver.
- **ELK Stack** (Elasticsearch + Logstash + Kibana) — use Logstash or Filebeat to ship logs.
- **Grafana Loki** — lightweight log aggregation integrated with Grafana dashboards.

### Horizontal Scaling

The application is stateless by design (JWT tokens are self-contained, no server-side session state). Multiple application instances can be run behind a load balancer:

```bash
docker compose up --scale app=3
```

Ensure the load balancer has a sticky-session policy disabled — all instances are equivalent and any instance can handle any request.

---

## 6. Rolling Back an Incorrect Deployment

### Application Rollback (Docker)

To roll back to a previous application version, pull and redeploy the tagged image:

```bash
docker pull ghcr.io/spandan-2006/enterprise-dashboard:v1.0.0
docker compose up -d
```

Update `docker-compose.yml` to reference the specific tag before restarting, or use the `image` override:

```bash
IMAGE_TAG=v1.0.0 docker compose up -d
```

### Database Rollback (Flyway)

Flyway migration files follow the naming convention `V{version}__{description}.sql` (e.g., `V2__add_audit_logs_table.sql`).

**If the schema change has not yet modified data (safe to undo):**

> **Important:** Flyway undo migrations (`V{n}__undo.sql`) require **Flyway Teams or Enterprise edition**. The Community edition (which this project uses) does NOT support undo scripts. Instead, use a forward-fix migration: create a new script `V{n+1}__fix_description.sql` that corrects the schema. For data-only rollbacks, restore from a database backup taken before the deployment.

Run the corresponding undo migration script `V{n}__undo_{description}.sql` to reverse the schema change, then redeploy the previous application version.

**If the schema change has already modified data (data present in new schema):**

Do not attempt to undo the Flyway migration directly. Instead:

1. Restore the database from the last known-good backup (automated backup from the managed database service, or a manual `pg_dump` snapshot taken before the deployment).
2. Redeploy the previous application version.

> **Critical rule:** Never roll back a Flyway migration that has already modified production data. Data written in the new schema format may be truncated or corrupted by an undo script. Always prefer a forward-fix migration (e.g., `V3__fix_column_type.sql`) or a database restore from backup.

### Verifying a Successful Rollback

After rolling back, run the health check to confirm the application is up and using the expected schema:

```bash
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/info
```

The `/actuator/info` endpoint should report the rolled-back application version number.
