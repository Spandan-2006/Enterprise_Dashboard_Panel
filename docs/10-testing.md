# 10 — Testing Strategy

## 1. Testing Philosophy

The testing approach for the Enterprise Deployment Dashboard is guided by four principles:

1. **Test behavior, not implementation.** Tests assert on observable outcomes (return values, status codes, side effects) rather than on internal implementation details (which private methods were called, how many times a field was accessed). This makes tests resilient to refactoring.

2. **Prefer integration tests for controllers.** Controller tests must load the full Spring Security context. Testing a controller without security is testing a different system — it will not catch authorization bugs. `@SpringBootTest` with `@AutoConfigureMockMvc` is the default choice for controller tests.

3. **Prefer unit tests for services.** Service-layer business logic is isolated, deterministic, and fast to test with mocked dependencies. Unit tests provide the tightest feedback loop during development.

4. **Tests are part of the definition of done.** No feature is considered complete without corresponding tests. A pull request that adds a new endpoint without tests for that endpoint will not be merged.

---

## 2. Testing Pyramid

```
         /\
        /  \
       / E2E\      ~5%  — manual / Cypress (not in CI v1)
      /──────\
     /Integra-\    ~35% — @SpringBootTest, @WebMvcTest, @DataJpaTest
    /  tion    \
   /────────────\
  /  Unit Tests  \  ~60% — @ExtendWith(MockitoExtension.class)
 /────────────────\
```

| Layer | Approx. Share | Tools | Runs In CI |
|---|---|---|---|
| Unit | 60% | JUnit 5, Mockito 5 | Yes |
| Integration | 35% | @SpringBootTest, @WebMvcTest, @DataJpaTest, H2 | Yes |
| E2E | 5% | Manual / Cypress | Not in v1 |

The pyramid shape is intentional: unit tests are cheap to run and maintain; integration tests are slower and more brittle; E2E tests are the most expensive. Keeping the proportions in this shape maximises feedback speed without sacrificing coverage of realistic runtime behavior.

---

## 3. Unit Testing

### Framework and Scope

- **Framework**: JUnit 5 (Jupiter) + Mockito 5
- **Scope**: All `@Service` classes — `AuthService`, `DeploymentService`, `UserService`, `AuditService`, `DashboardService`
- **Test runner annotation**: `@ExtendWith(MockitoExtension.class)` (no Spring context loaded)

### Coverage Targets

| Metric | Target |
|---|---|
| Line coverage | 80% minimum |
| Branch coverage | 70% minimum |

These are enforced by JaCoCo in CI (see Section 8).

### Test Naming Convention

```
methodName_givenCondition_thenExpectedResult()
```

Examples:
- `createDeployment_givenValidRequest_thenReturnsDeploymentResponse()`
- `login_givenInvalidPassword_thenThrowsInvalidCredentialsException()`
- `rollback_givenNonFailedDeployment_thenThrowsIllegalStateException()`

Use `@DisplayName` to add a human-readable description that appears in test reports.

### Example — DeploymentService Unit Test

```java
@ExtendWith(MockitoExtension.class)
class DeploymentServiceTest {

    @Mock
    DeploymentRepository deploymentRepository;

    @Mock
    AuditService auditService;

    @InjectMocks
    DeploymentService deploymentService;

    @Test
    @DisplayName("createDeployment: valid request → returns DeploymentResponse with PENDING status")
    void createDeployment_givenValidRequest_thenStatusIsPending() {
        // given
        CreateDeploymentRequest request = new CreateDeploymentRequest(
            "payment-service", "1.0.0", Environment.STAGING, null);
        Deployment saved = new Deployment();
        saved.setId(1L);
        saved.setStatus(DeploymentStatus.PENDING);
        when(deploymentRepository.save(any())).thenReturn(saved);

        // when
        DeploymentResponse response = deploymentService.createDeployment(request, "devops");

        // then
        assertThat(response.getStatus()).isEqualTo(DeploymentStatus.PENDING);
        verify(auditService).log(eq("devops"), eq(AuditActions.DEPLOYMENT_CREATED), anyString());
    }
}
```

**Structure pattern — given / when / then:**

- `// given` — set up the inputs and mock stubs
- `// when` — invoke the method under test
- `// then` — assert on the result and verify mock interactions

---

## 4. Integration Testing — Controller Tests

### Framework and Scope

- **Framework**: `@SpringBootTest` + `@AutoConfigureMockMvc` (full application context) or `@WebMvcTest` for slice tests
- **Database**: H2 in-memory, activated by Spring profile `test`
- **Security**: Spring Security context is always loaded in controller tests. Never use `@WithMockUser` as a substitute for a real JWT — use `@WithMockUser` only when the test focus is not on the token itself.

### When to Use @SpringBootTest vs @WebMvcTest

| Use Case | Annotation |
|---|---|
| Testing the full request/response cycle including security filter chain | `@SpringBootTest` + `@AutoConfigureMockMvc` |
| Testing controller logic in isolation with mocked service layer | `@WebMvcTest(XController.class)` |

Prefer `@SpringBootTest` for security tests because `@WebMvcTest` may not load all security components by default and can give false positives on authorization.

### Example — AuthController Integration Test

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class AuthControllerIntegrationTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    void login_withValidCredentials_returns200WithTokens() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"admin\",\"password\":\"admin123\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.accessToken").isNotEmpty())
            .andExpect(jsonPath("$.data.tokenType").value("Bearer"));
    }

    @Test
    void login_withInvalidPassword_returns401() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"admin\",\"password\":\"wrongpassword\"}"))
            .andExpect(status().isUnauthorized());
    }
}
```

---

## 5. Repository Testing

### Framework and Scope

- **Framework**: `@DataJpaTest` — loads only the JPA slice (no web layer, no services)
- **Database**: H2 in-memory (same `test` profile)
- **Scope**: All custom query methods defined in `DeploymentRepository`, `RollbackRepository`, `AuditLogRepository`, `UserRepository`

`@DataJpaTest` automatically configures `TestEntityManager` for setting up test data without going through the full service layer.

### Example — DeploymentRepository Test

```java
@DataJpaTest
@ActiveProfiles("test")
class DeploymentRepositoryTest {

    @Autowired
    TestEntityManager entityManager;

    @Autowired
    DeploymentRepository repository;

    @Test
    void findByStatus_givenExistingStatus_returnsMatchingDeployments() {
        // given
        Deployment deployment = new Deployment();
        deployment.setServiceName("payment-service");
        deployment.setVersion("1.0.0");
        deployment.setStatus(DeploymentStatus.PENDING);
        entityManager.persistAndFlush(deployment);

        // when
        Page<Deployment> results = repository.findByStatus(
            DeploymentStatus.PENDING, PageRequest.of(0, 10));

        // then
        assertThat(results.getContent()).hasSize(1);
        assertThat(results.getContent().get(0).getServiceName()).isEqualTo("payment-service");
    }

    @Test
    void findByStatus_givenNoMatchingStatus_returnsEmptyPage() {
        // when
        Page<Deployment> results = repository.findByStatus(
            DeploymentStatus.FAILED, PageRequest.of(0, 10));

        // then
        assertThat(results.getContent()).isEmpty();
    }
}
```

---

## 6. Security Testing

Security tests verify the authorization matrix from `docs/09-security.md`. Every combination of role and endpoint that has a defined `Y` or `N` outcome must have a corresponding test.

### Unauthenticated Access — Expect 401

```java
@Test
void getDeployments_withNoToken_returns401() throws Exception {
    mockMvc.perform(get("/api/deployments"))
        .andExpect(status().isUnauthorized());
}
```

### Wrong Role — Expect 403

```java
@Test
void createDeployment_asDeveloper_returns403() throws Exception {
    // developerToken is a JWT generated in @BeforeEach with Role.DEVELOPER
    mockMvc.perform(post("/api/deployments")
            .header("Authorization", "Bearer " + developerToken)
            .contentType(MediaType.APPLICATION_JSON)
            .content(validCreateDeploymentJson))
        .andExpect(status().isForbidden());
}
```

### Expired Token — Expect 401

```java
@Test
void getDeployments_withExpiredToken_returns401() throws Exception {
    // Generate a token with past expiry (testJwtUtil.generateTokenWithExpiry(username, role, -1))
    String expiredToken = jwtUtil.generateTokenWithCustomExpiry("admin", "ADMIN",
        new Date(System.currentTimeMillis() - 1000));

    mockMvc.perform(get("/api/deployments")
            .header("Authorization", "Bearer " + expiredToken))
        .andExpect(status().isUnauthorized());
}
```

### Correct Role — Expect 200

```java
@Test
void createDeployment_asDevopsEngineer_returns201() throws Exception {
    mockMvc.perform(post("/api/deployments")
            .header("Authorization", "Bearer " + devopsToken)
            .contentType(MediaType.APPLICATION_JSON)
            .content(validCreateDeploymentJson))
        .andExpect(status().isCreated());
}
```

### Security Test Coverage Checklist

For each endpoint in the authorization matrix, write tests for:

1. No token → 401
2. Valid token, wrong role → 403
3. Valid token, correct role → 2xx
4. Expired token → 401

---

## 7. Test Configuration

### application-test.yml

Location: `src/test/resources/application-test.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  flyway:
    enabled: false  # H2 doesn't need Flyway; JPA creates schema from entities

jwt:
  secret: dGVzdC1zZWNyZXQta2V5LWZvci11bml0LXRlc3RpbmctMzItYnl0ZXM=  # test-only 32-byte key
  expiry-ms: 86400000        # 24 hours
  refresh-expiry-ms: 604800000  # 7 days
```

**Important:** The `jwt.secret` value above is a test-only key hardcoded for reproducibility in CI. It must never be used in production. The production value is always supplied via the `JWT_SECRET` environment variable.

### Test Data Builders

Use the builder pattern to construct test entities. Builders provide sensible defaults and allow tests to only specify the fields relevant to the scenario under test.

```java
public class UserTestBuilder {
    private String username = "testuser";
    private String email = "test@test.com";
    private String password = "$2a$12$hashedPasswordHere";
    private Role role = Role.DEVELOPER;

    public UserTestBuilder withUsername(String username) {
        this.username = username;
        return this;
    }

    public UserTestBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserTestBuilder withRole(Role role) {
        this.role = role;
        return this;
    }

    public User build() {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setPassword(password);
        user.setRole(role);
        return user;
    }
}
```

Usage in tests:

```java
User admin = new UserTestBuilder().withRole(Role.ADMIN).withUsername("admin").build();
User developer = new UserTestBuilder().withRole(Role.DEVELOPER).build();
```

Similar builders should exist for `DeploymentTestBuilder` and `AuditLogTestBuilder`.

### Test JWT Utility

For security tests that require an expired token, add a test-only helper method to `JwtUtil` or create a `TestJwtUtil` class:

```java
// In JwtUtil.java (or a test utility class)
String generateTokenWithExpiry(UserDetails userDetails, Date expiry) {
    return Jwts.builder()
        .subject(userDetails.getUsername())
        .expiration(expiry)  // can be in the past for expired-token tests
        .signWith(getSignKey())
        .compact();
}
```

Use `new Date(System.currentTimeMillis() - 1000)` as expiry to produce an already-expired token in tests.

---

## 8. CI Test Execution

### Running Tests Locally

```bash
./gradlew test
```

Generates:
- HTML test report: `build/reports/tests/test/index.html`
- JaCoCo XML + HTML coverage report: `build/reports/jacoco/test/html/index.html`

### JaCoCo Configuration in build.gradle.kts

```kotlin
jacoco {
    toolVersion = "0.8.11"
}

tasks.jacocoTestReport {
    reports {
        xml.required.set(true)   // required by some CI coverage plugins
        html.required.set(true)
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = "LINE"
                minimum = "0.80".toBigDecimal()
              }
              limit {
                counter = "BRANCH"
                minimum = "0.70".toBigDecimal()
            }
        }
    }
}

// jacocoTestCoverageVerification runs as part of `check`
tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}

// Always generate the HTML report before verifying
tasks.jacocoTestCoverageVerification {
    dependsOn(tasks.jacocoTestReport)
}
```

### GitHub Actions Test Step

```yaml
- name: Run tests with coverage verification
  run: ./gradlew test jacocoTestCoverageVerification

- name: Upload test report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-reports
    path: build/reports/tests/test/
```

The `./gradlew test jacocoTestCoverageVerification` command will **fail the build** if line coverage falls below 80%. The `if: always()` on the upload step ensures test reports are available for inspection even when the build fails.

---

## 9. Coverage Targets

The following table is the reference for tracking coverage per module. The `Current` column is updated from the JaCoCo report after each CI run.

| Module | Target | Current |
|---|---|---|
| AuthService | 90% | — |
| DeploymentService | 85% | — |
| AuditService | 80% | — |
| UserService | 85% | — |
| DashboardService | 80% | — |
| AuthController | 85% | — |
| DeploymentController | 85% | — |
| UserController | 85% | — |
| **Overall** | **80%** | **—** |

**Why the targets differ by module:**

- `AuthService` has a higher target (90%) because authentication failures are the most critical class of bugs in this application.
- Dashboard and Audit services have a lower floor (80%) because they contain more read-only aggregation logic with fewer edge cases.
- Controller coverage counts integration tests, which cover the happy path and common error paths but may not exercise every conditional branch in response serialization.
