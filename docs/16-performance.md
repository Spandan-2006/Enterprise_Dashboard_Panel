# 16 — Performance Guidelines

## 1. Performance Targets

| Metric | Target | Measurement Method |
|---|---|---|
| API p95 response time | < 200ms | JMeter/k6 load test at 100 concurrent users |
| API p99 response time | < 500ms | JMeter/k6 load test at 100 concurrent users |
| Database query time | < 50ms for indexed queries | PostgreSQL `EXPLAIN ANALYZE` |
| JWT validation time | < 5ms | Micrometer timer on `JwtUtil.isTokenValid()` |
| Application startup | < 30s | Spring Boot startup timer |
| Dashboard stats API | < 200ms (compliance with NFR-001) | Load test |

These targets apply to the staging and production environments under normal load (up to 100 concurrent users). Targets are verified with load tests before each production release.

---

## 2. Database Performance

### Index Strategy

Full index definitions are in `docs/07-database.md`. Key indexes for query performance:

- `idx_deployments_status` on `deployments(status)` — filters by status in list endpoints
- `idx_deployments_environment` on `deployments(environment)` — filters by environment
- `idx_deployments_env_status` composite on `deployments(environment, status)` — supports combined filters
- `idx_audit_logs_deployment_id` on `audit_logs(deployment_id)` — JOIN in audit history queries
- `idx_audit_logs_timestamp` on `audit_logs(created_at)` — time-range queries

Refer to `docs/07-database.md` for the complete index list, including partial indexes and expression indexes.

### HikariCP Connection Pool Configuration

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000    # 30 seconds — fail fast if pool exhausted
      idle-timeout: 600000         # 10 minutes — reclaim idle connections
      max-lifetime: 1800000        # 30 minutes — rotate before DB-side timeout
```

Tuning guidance:
- `maximum-pool-size: 10` is appropriate for a single application instance under typical load. Increase to 20 if p99 latency degrades under load test and CPU/DB are not saturated.
- Keep `minimum-idle` low (2) in dev/staging to reduce idle DB connections. Raise it in production if connection acquisition latency is observed on burst traffic.
- `max-lifetime` must be shorter than the PostgreSQL `idle_in_transaction_session_timeout` and the network proxy timeout (typically 30–60 minutes).

### N+1 Query Prevention

Never load associations lazily inside a loop. Two safe patterns:

**Option 1 — `@EntityGraph` on repository method:**

```java
@EntityGraph(attributePaths = {"rollbacks"})
Optional<Deployment> findWithRollbacksById(Long id);
```

**Option 2 — JOIN FETCH in JPQL:**

```java
@Query("SELECT d FROM Deployment d LEFT JOIN FETCH d.rollbacks WHERE d.id = :id")
Optional<Deployment> findWithRollbacksById(@Param("id") Long id);
```

Use projection interfaces for read-only list endpoints to avoid loading full entities and their associations (see Section 4).

### Mandatory Pagination

All list endpoints must use `Pageable`. Unbounded collection queries are prohibited.

```java
// Repository
Page<Deployment> findAll(Specification<Deployment> spec, Pageable pageable);

// Controller
@GetMapping
public ResponseEntity<ApiResponse<Page<DeploymentSummary>>> list(
    @PageableDefault(size = 20, max = 100) Pageable pageable) {
    ...
}
```

Maximum page size: **100 records**. The `@PageableDefault` annotation enforces this. Clients requesting `size > 100` receive the capped value, not an error.

---

## 3. JVM Tuning (Recommended Production)

Set these flags in the container `JAVA_OPTS` environment variable or in the Dockerfile `CMD`:

```
-Xms512m              # initial heap — avoids GC pauses on first load
-Xmx1g                # max heap — leave room for OS and non-heap
-XX:+UseContainerSupport     # respect container memory limits
-XX:MaxRAMPercentage=75.0    # use 75% of container memory as heap
-Djava.security.egd=file:/dev/./urandom  # faster secure random for JWT
```

**Notes:**
- G1GC is the default garbage collector in JDK 21 — no explicit override is needed.
- `UseContainerSupport` is enabled by default in JDK 11+ but is explicitly included here for clarity.
- If both `-Xmx` and `-XX:MaxRAMPercentage` are set, the explicit `-Xmx` takes precedence. Use one or the other — `MaxRAMPercentage` is preferred in containerized deployments where container memory limits may vary across environments.
- The `/dev/./urandom` path notation (with the extra `.`) bypasses the JVM's blocking `/dev/random` fallback check on Linux, preventing JWT signing from stalling under load.

**Dockerfile example:**

```dockerfile
ENV JAVA_OPTS="-Xms512m -Xmx1g -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/./urandom"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## 4. API Response Optimization

### Projection Interfaces

For GET list endpoints, return only the fields the client needs rather than full entity graphs:

```java
public interface DeploymentSummary {
    Long getId();
    String getProjectName();
    String getVersion();
    DeploymentStatus getStatus();
    LocalDateTime getStartTime();
}
```

Spring Data JPA resolves these at query time using column-level selection, avoiding full entity materialization. Use the projection as the return type directly:

```java
Page<DeploymentSummary> findAllProjectedBy(Pageable pageable);
```

### GZIP Compression

Enable in `application.yml` (also included in the base configuration template in `docs/18-environments.md`):

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json
    min-response-size: 1024
```

Compression reduces response size for JSON payloads over 1 KB. For the typical deployment list response (20 items), this reduces payload size by approximately 60–70%.

### Cache-Control Header

For stable endpoints that change infrequently, set a `Cache-Control` header. The dashboard stats endpoint is a good candidate because aggregated statistics change at most every 30 seconds:

```java
// In DashboardController.getStats()
return ResponseEntity.ok()
    .cacheControl(CacheControl.maxAge(30, TimeUnit.SECONDS))
    .body(ApiResponse.success(stats));
```

This allows CDN edge nodes and browser clients to cache the response for 30 seconds, reducing redundant database aggregation queries during traffic spikes.

---

## 5. Load Testing Guidance

### Recommended Tools

- **Apache JMeter** — GUI-based, good for teams new to load testing
- **k6** — scriptable, CI-friendly, lightweight

### Test Scenarios

| Scenario | Endpoint | Notes |
|---|---|---|
| 1 | POST `/api/auth/login` | High frequency — test token issuance throughput |
| 2 | GET `/api/deployments` | Most common read — paginated list |
| 3 | POST `/api/deployments` | Write load — concurrent deployment creation |
| 4 | GET `/api/dashboard/stats` | Aggregation query — most expensive DB operation |
| 5 | PATCH `/api/deployments/{id}/status` | Concurrent status updates — test for race conditions |

### Test Profiles

| Profile | Configuration | Purpose |
|---|---|---|
| Baseline | 10 concurrent users, 5 minutes, all key endpoints | Establish healthy latency baseline |
| Stress | Ramp from 10 to 200 concurrent users over 10 minutes | Find breaking point |
| Soak | 50 concurrent users, 30 minutes | Detect memory leaks and connection pool exhaustion |

### Acceptance Criteria

- p95 response time < 200ms
- p99 response time < 500ms
- Error rate < 1%

### k6 Minimal Script

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 10,
  duration: '5m',
};

export default function () {
  const res = http.get('http://localhost:8080/api/dashboard/stats', {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

**Running the k6 baseline test:**

```bash
# Set a valid bearer token first
export TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  | jq -r '.data.accessToken')

k6 run -e TOKEN=$TOKEN load-test.js
```

**k6 output interpretation:**

- `http_req_duration` p(95) and p(99) — compare against targets above
- `http_req_failed` rate — must stay below 1%
- `http_reqs` total — sanity check that the test ran the expected number of iterations
