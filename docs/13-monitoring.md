# docs/13-monitoring.md — Monitoring and Observability

## 1. Logging Strategy

### Framework

The application uses **SLF4J** as the logging API and **Logback** as the underlying implementation. Both are included transitively via `spring-boot-starter-web` and require no additional Gradle dependencies for basic usage.

All logging calls in application code must target the SLF4J API only. Direct use of `java.util.logging`, `log4j`, or `System.out.println` for diagnostic output is not permitted.

### Log Levels by Environment

| Environment | Root Level | `com.enterprise.dashboard` Level |
|---|---|---|
| Development (local) | INFO | DEBUG |
| Test (CI) | WARN | DEBUG |
| Staging | INFO | INFO |
| Production | WARN | WARN |

Log levels are controlled via `application-{profile}.yml` or environment variables (`LOGGING_LEVEL_ROOT`, `LOGGING_LEVEL_COM_ENTERPRISE_DASHBOARD`). Staging and production levels can be changed at runtime via the `/actuator/loggers` endpoint without a restart (Admin JWT required — see Section 3).

### MDC (Mapped Diagnostic Context)

Every log line automatically includes the following MDC fields, populated by a servlet filter (`RequestContextFilter`) that runs before the Spring Security filter chain:

| MDC Key | Description | Example |
|---|---|---|
| `requestId` | UUID generated per HTTP request; read from the `X-Request-ID` header if present, otherwise auto-generated | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| `username` | Authenticated user's username; empty string for unauthenticated requests | `johndoe` |
| `endpoint` | HTTP method and path template (not the full URL, to avoid logging path parameters that may be sensitive) | `POST /api/deployments` |

Clients should include an `X-Request-ID` header on each request to enable end-to-end correlation across systems. If the header is absent, the server generates a UUID and returns it in the response as `X-Request-ID`.

### Log Format

```
[%d{ISO8601}] [%-5level] [%mdc{requestId}] [%mdc{username}] %logger{36} - %msg%n
```

Example output:

```
[2025-07-13T14:32:01.456] [INFO ] [a1b2c3d4-e5f6-7890-abcd-ef1234567890] [johndoe] c.e.d.service.DeploymentService - Deployment created: id=42, project=payments-api, env=staging
```

### Data That Must Never Be Logged

The following data categories must never appear in log output under any log level:

- **Passwords** — user passwords, database passwords, any authentication credential
- **JWT tokens** — full access tokens or refresh tokens (short token prefixes for debugging, e.g., `eyJhbGc...`, are acceptable if truncated to 20 characters)
- **Full email addresses** — mask as `em***@domain.com` (preserve domain for debugging while obscuring the local part)
- **Credit card numbers, national IDs, or any PII beyond username**

Apply masking at the point of log construction, not at the appender level. Log appender-level filtering is a defense-in-depth measure, not the primary control.

---

## 2. Logback Configuration

### File Location

`src/main/resources/logback-spring.xml`

Using `logback-spring.xml` (rather than `logback.xml`) allows Spring Boot to inject environment-specific configuration via `<springProfile>` blocks before Logback initializes.

### Configuration Structure

The Logback configuration defines three profiles:

**Development and Test profiles — console appender with color output**

```xml
<springProfile name="dev,test">
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>%cyan([%d{ISO8601}]) %highlight([%-5level]) [%yellow(%mdc{requestId})] [%green(%mdc{username})] %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
  </root>

  <logger name="com.enterprise.dashboard" level="DEBUG"/>
</springProfile>
```

**Staging and Production profiles — rolling file appender plus console appender for errors**

```xml
<springProfile name="staging,prod">
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/enterprise-dashboard/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <!-- Daily rotation -->
      <fileNamePattern>/var/log/enterprise-dashboard/app.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
      <!-- 30-day retention -->
      <maxHistory>30</maxHistory>
      <!-- 100 MB per file before forced rotation -->
      <maxFileSize>100MB</maxFileSize>
      <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>[%d{ISO8601}] [%-5level] [%mdc{requestId}] [%mdc{username}] %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <!-- Errors also written to console for Docker log driver capture -->
  <appender name="CONSOLE_ERROR" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
      <level>ERROR</level>
    </filter>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>[%d{ISO8601}] [%-5level] [%mdc{requestId}] [%mdc{username}] %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="WARN">
    <appender-ref ref="FILE"/>
    <appender-ref ref="CONSOLE_ERROR"/>
  </root>

  <logger name="com.enterprise.dashboard" level="INFO"/>
</springProfile>
```

**Rolling policy summary:**

| Property | Value |
|---|---|
| Rotation trigger | Daily (midnight UTC) |
| Retention | 30 days |
| Max file size | 100 MB (forces mid-day rotation if a single day's log exceeds this) |
| Total size cap | 3 GB (oldest files deleted if total exceeds this) |
| Compression | `.gz` (Logback compresses rotated files automatically) |
| File path | `/var/log/enterprise-dashboard/app.log` |

Ensure the Docker container has write access to `/var/log/enterprise-dashboard/` by mounting a volume or creating the directory with correct permissions in the Dockerfile.

---

## 3. Spring Boot Actuator Endpoints

Actuator endpoints are configured in `application.yml` using the `management.endpoints` properties. The table below shows the intended exposure per environment.

| Endpoint | Purpose | Exposed In | Auth Required |
|---|---|---|---|
| `GET /actuator/health` | Liveness probe and database connectivity check | All environments | No |
| `GET /actuator/info` | Application name, version, and build info | All environments | No |
| `GET /actuator/metrics` | JVM heap, HTTP latency, HikariCP pool | Staging + Production | Yes (Admin JWT) |
| `GET /actuator/loggers` | View and change runtime log levels per logger | Staging only | Yes (Admin JWT) |
| `GET /actuator/env` | Resolved environment properties | Disabled in production | N/A |

**Recommended `application.yml` configuration:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers
  endpoint:
    health:
      show-details: when-authorized
    loggers:
      enabled: true
  info:
    env:
      enabled: true

# Production override (application-prod.yml):
# management.endpoints.web.exposure.include: health,info,metrics
# management.endpoint.loggers.enabled: false
# management.endpoint.env.enabled: false
```

**Security:** The `/actuator/metrics` and `/actuator/loggers` endpoints are protected by the same JWT security filter as the API endpoints. Requests must include a valid `Authorization: Bearer {token}` header from an `ADMIN`-role user. The `/actuator/health` and `/actuator/info` endpoints are explicitly permitted without authentication in the Spring Security configuration.

---

## 4. Key Metrics to Track

### HTTP Metrics (Micrometer + Actuator)

Micrometer's `WebMvcMetricsFilter` automatically instruments all Spring MVC handler methods and produces the following metrics:

| Metric | Tags | Purpose |
|---|---|---|
| `http.server.requests` | `method`, `uri`, `status`, `outcome` | Per-endpoint request count, total time, and max time |
| Derived: error rate | Filter `outcome=SERVER_ERROR` or `outcome=CLIENT_ERROR` | Rate of 4xx and 5xx responses |

To query p50/p95/p99 latency per endpoint, enable histogram publication in `application.yml`:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
```

### JVM Metrics

Automatically published by the `JvmMetricsAutoConfiguration`:

| Metric | Description |
|---|---|
| `jvm.memory.used{area="heap"}` | Current heap memory usage in bytes |
| `jvm.gc.pause` | Garbage collection pause duration histogram |
| `jvm.threads.live` | Current count of live (non-daemon + daemon) threads |
| `jvm.threads.daemon` | Current count of daemon threads |

### Business Metrics (Custom Micrometer Counters — v2)

The following custom metrics are planned for v2. They will be implemented as `Counter` beans registered in a `@Configuration` class and incremented from the service layer.

| Metric | Type | Description |
|---|---|---|
| `deployment.created.count` | Counter | Number of deployments created; tagged by `environment` and `project` |
| `deployment.failed.count` | Counter | Number of deployments that transitioned to `FAILED` status |
| `rollback.triggered.count` | Counter | Number of rollback actions triggered; tagged by `triggered_by` role |

**Example registration:**

```java
@Bean
public Counter deploymentCreatedCounter(MeterRegistry registry) {
    return Counter.builder("deployment.created.count")
        .description("Number of deployments created")
        .register(registry);
}
```

### Database Metrics (HikariCP)

HikariCP publishes its own metrics to Micrometer automatically when Micrometer is on the classpath:

| Metric | Description |
|---|---|
| `hikaricp.connections.active` | Number of connections currently in use by the application |
| `hikaricp.connections.idle` | Number of connections in the pool that are idle |
| `hikaricp.connections.pending` | Number of threads waiting to acquire a connection |
| `hikaricp.connections.max` | Configured maximum pool size |
| `hikaricp.connections.acquire` | Histogram of time spent waiting to acquire a connection |

A high `hikaricp.connections.pending` value is a leading indicator of database connection exhaustion. Alert on this metric before the pool is completely saturated (see Section 5).

---

## 5. Alert Conditions (Recommended)

The thresholds below are starting points. Calibrate them based on observed baseline values in your specific deployment before enabling production paging alerts.

| Metric | Threshold | Severity | Recommended Action |
|---|---|---|---|
| HTTP error rate (`5xx`) | > 5% of requests over 5 minutes | WARNING | Investigate application logs; check `POST /api/auth/login` for auth issues |
| HTTP error rate (`5xx`) | > 10% of requests over 5 minutes | CRITICAL | Page on-call engineer; check for dependency failures |
| API p99 latency | > 500 ms sustained for 5 minutes | WARNING | Check PostgreSQL slow query log; check HikariCP pool saturation |
| JVM heap usage | > 80% of configured max heap | WARNING | Check for memory leaks; review recent code changes; consider increasing `-XX:MaxRAMPercentage` |
| HikariCP connections pending | > 5 threads waiting for 2 minutes | CRITICAL | Scale database connections or increase `spring.datasource.hikari.maximum-pool-size` |
| Deployment failure rate | > 20% of deployments in 1 hour | CRITICAL | Halt further deployments; investigate pipeline; check external system connectivity |
| Application health | `status != UP` for 1 minute | CRITICAL | Restart container; check database connectivity; page on-call immediately |

**Alert routing guidance:**

- WARNING alerts should create a ticket and send a Slack notification to `#alerts-warning`.
- CRITICAL alerts should page the on-call engineer via PagerDuty or OpsGenie.
- All alerts should include the `requestId` of the first failing request where applicable, to enable rapid log lookup.

---

## 6. Future Observability Stack

The following components are planned for v2 to provide a complete observability solution covering the three pillars: metrics, logs, and traces.

### Metrics — Prometheus + Grafana

1. Add the Micrometer Prometheus registry dependency to `build.gradle.kts`:
   ```kotlin
   implementation("io.micrometer:micrometer-registry-prometheus")
   ```
2. Expose the `/actuator/prometheus` endpoint (Prometheus scrapes this endpoint every 15 seconds by default).
3. Configure a Prometheus scrape job targeting the application instances.
4. Build Grafana dashboards:
   - **HTTP Overview** — request rate, error rate, p95/p99 latency by endpoint
   - **JVM Dashboard** — heap usage, GC activity, thread count
   - **Business Dashboard** — deployment creation rate, failure rate, rollback frequency
   - **HikariCP Dashboard** — connection pool utilization

### Logs — Grafana Loki or ELK Stack

**Option A: Grafana Loki** (recommended for teams already using Grafana)
- Lightweight log aggregation designed to index only log metadata (labels), not full-text content.
- Use Promtail as the log shipper. Configure it to read from `/var/log/enterprise-dashboard/app.log`.
- Correlate logs with Grafana metrics using the `requestId` label.

**Option B: ELK Stack** (Elasticsearch + Logstash + Kibana)
- Use Logstash with a Logback `LogstashEncoder` appender to ship structured JSON logs directly to Elasticsearch.
- Kibana dashboards enable full-text search and filtering across log fields.
- Higher resource requirements than Loki; suitable for high-volume log environments.

### Traces — OpenTelemetry

OpenTelemetry distributed tracing will be added when the application grows to multiple services. The implementation plan:

1. Add the OpenTelemetry Spring Boot starter (`io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter`).
2. Configure the OTLP exporter to send traces to a collector (Jaeger, Tempo, or AWS X-Ray).
3. Propagate the `traceparent` W3C trace context header across service boundaries.
4. Correlate trace IDs with log entries by including the `traceId` and `spanId` in the MDC.

For the current single-service v1 architecture, the `requestId` MDC field provides sufficient request-level correlation. OpenTelemetry will become essential when async processing, event-driven messaging, or additional microservices are introduced.
