# 17 — Error Handling Guide

## 1. Standard Error Response Format

All API error responses use the same `ApiResponse` envelope. On error, `success` is `false`, `data` is `null`, and `message` describes the error. The `errors` field is populated only for validation failures.

**Validation error response (400):**

```json
{
  "success": false,
  "message": "Validation failed",
  "data": null,
  "timestamp": "2026-07-13T10:30:00Z",
  "errors": [
    { "field": "projectName", "message": "must not be blank" },
    { "field": "version", "message": "size must be between 1 and 50" }
  ]
}
```

**Non-validation error response (all other errors):**

```json
{
  "success": false,
  "message": "Deployment ID 999 not found",
  "data": null,
  "timestamp": "2026-07-13T10:30:00Z",
  "errors": null
}
```

**`ApiResponse` structure:**

```java
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private Instant timestamp;
    private List<FieldError> errors;

    public record FieldError(String field, String message) {}
}
```

---

## 2. HTTP Status Code Usage

| Code | Name | When Used | Example |
|---|---|---|---|
| 200 | OK | Successful GET, PUT, PATCH | `GET /api/deployments` response |
| 201 | Created | Successful POST (resource created) | `POST /api/deployments`; `Location` header included |
| 204 | No Content | Successful DELETE | `DELETE /api/deployments/{id}` |
| 400 | Bad Request | Validation failure, missing required fields, malformed JSON | Missing `projectName` in request body |
| 401 | Unauthorized | Missing token, invalid token, expired token, invalid credentials | Expired JWT |
| 403 | Forbidden | Valid token but insufficient role | DEVELOPER calling `POST /api/deployments` |
| 404 | Not Found | Resource not found by ID | Deployment ID 999 does not exist |
| 409 | Conflict | Duplicate resource, FK constraint violation | Username already taken |
| 422 | Unprocessable Entity | Business rule violation | Rollback on SUCCESSFUL deployment; invalid status transition |
| 500 | Internal Server Error | Unexpected server error | Unhandled exception — never expose stack trace to client |

---

## 3. GlobalExceptionHandler Complete Mapping

`GlobalExceptionHandler` is annotated with `@RestControllerAdvice` and handles all exceptions at the web layer. The table below lists every handled exception, the HTTP status returned, the `message` value in the response, and whether `errors` is populated.

| Exception Class | HTTP Status | `message` Value | `errors` |
|---|---|---|---|
| `MethodArgumentNotValidException` | 400 | `"Validation failed"` | Field-level array |
| `ConstraintViolationException` | 400 | `"Validation failed"` | Field-level array |
| `HttpMessageNotReadableException` | 400 | `"Malformed JSON request body"` | null |
| `ValidationException` | 400 | `exception.getMessage()` | null |
| `InvalidStatusTransitionException` | 422 | `"Cannot transition from {from} to {to}"` | null |
| `ResourceNotFoundException` | 404 | `"Resource not found: {type} with id {id}"` | null |
| `UnauthorizedException` | 401 | `"Authentication required or credentials invalid"` | null |
| `ExpiredJwtException` | 401 | `"JWT token has expired"` | null |
| `MalformedJwtException` | 401 | `"Invalid JWT token format"` | null |
| `AccessDeniedException` | 403 | `"Insufficient permissions for this operation"` | null |
| `ForbiddenException` | 403 | `exception.getMessage()` | null |
| `ConflictException` | 409 | `exception.getMessage()` | null |
| `DataIntegrityViolationException` | 409 | `"Resource already exists or constraint violation"` | null |
| `Exception` (catch-all) | 500 | `"An unexpected error occurred. Reference: {UUID}"` | null |

### 500 Catch-All Behaviour

When the catch-all `Exception` handler is triggered:

1. Generate a random UUID as a correlation reference.
2. Log at `ERROR` level with the UUID and the full stack trace.
3. Return only the UUID to the client — no internal details, no stack trace.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
    String referenceId = UUID.randomUUID().toString();
    log.error("Unexpected error [ref={}]: {}", referenceId, ex.getMessage(), ex);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(ApiResponse.error("An unexpected error occurred. Reference: " + referenceId));
}
```

The UUID appears in both the log entry and the API response, allowing support teams to correlate a client-reported reference ID with the exact log line.

---

## 4. Validation Error Handling Details

### Bean Validation (`@Valid`)

When a controller method parameter is annotated with `@Valid` (or `@Validated`) and validation fails, Spring throws `MethodArgumentNotValidException`. The handler extracts all field violations:

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException ex) {
    List<ApiResponse.FieldError> fieldErrors = ex.getBindingResult()
        .getFieldErrors()
        .stream()
        .map(e -> new ApiResponse.FieldError(e.getField(), e.getDefaultMessage()))
        .toList();
    return ResponseEntity.badRequest()
        .body(ApiResponse.validationError("Validation failed", fieldErrors));
}
```

Key properties:
- **All violations are collected** into a single response — no partial responses, no stopping at the first failure.
- Multiple field errors appear together in the `errors` array.
- The `message` field is always `"Validation failed"` for this exception type.

### `ConstraintViolationException`

Thrown when `@Validated` is used on service or repository methods (path variables, query params). The handler maps `ConstraintViolation` objects to `FieldError` records using the property path as the field name.

### Custom Validators

Future custom validators (e.g., `@ValidVersion` for semantic version format) follow the same Bean Validation API and produce `MethodArgumentNotValidException` — no additional exception handler needed.

---

## 5. Frontend Error Handling

### Axios Response Interceptor

Configure in `axiosInstance.js` (or equivalent HTTP client setup file):

```javascript
axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    const status = error.response?.status;
    const data = error.response?.data;

    if (status === 401) {
      localStorage.removeItem('accessToken');
      localStorage.removeItem('refreshToken');
      window.location.href = '/login';
    } else if (status === 403) {
      navigate('/forbidden');
    } else if (status === 422) {
      // Display data.message inline on the form
      setFormError(data.message);
    } else if (status >= 500) {
      const ref = data?.message || 'Unknown error';
      showToast(`Server error. Reference: ${ref}`, 'error');
    }

    return Promise.reject(error);
  }
);
```

### Handling 400 Validation Errors on Forms

When the server returns a 400 with field-level errors, iterate the `errors` array and display inline messages below each field:

```javascript
if (status === 400 && data.errors) {
  data.errors.forEach(({ field, message }) => {
    setFieldError(field, message); // e.g., React Hook Form setError()
  });
}
```

### Error Display Summary

| Status | Frontend Action |
|---|---|
| 400 (validation) | Iterate `errors` array; display each `message` inline below the corresponding form field |
| 400 (other) | Display `data.message` as a form-level error |
| 401 | Clear localStorage tokens; redirect to `/login` |
| 403 | Navigate to `/forbidden` |
| 404 | Display "Not found" inline or navigate to 404 page |
| 422 | Display `data.message` as an inline error on the form |
| 5xx | Display global error toast: `"Server error. Reference: {data.message}"` |

---

## 6. Error Logging Rules

### Log Levels

| Error Type | Log Level | What to Include |
|---|---|---|
| 4xx client errors | `WARN` | Request path, HTTP method, username (from MDC), error message |
| 5xx server errors | `ERROR` | Full stack trace, generated UUID reference |

### 4xx Logging Pattern

```java
log.warn("[{}] {} {} - {}: {}",
    username,       // from MDC or SecurityContext
    request.getMethod(),
    request.getRequestURI(),
    ex.getClass().getSimpleName(),
    ex.getMessage()
);
```

Example output:
```
WARN  [jdoe] PATCH /api/deployments/42/status - InvalidStatusTransitionException: Cannot transition from FAILED to IN_PROGRESS
```

### 5xx Logging Pattern

```java
log.error("Unexpected error [ref={}]: {}", referenceId, e.getMessage(), e);
```

Example output:
```
ERROR Unexpected error [ref=3f8a2c1d-...]: Connection refused to PostgreSQL
      com.example.SomeService.doWork(SomeService.java:88)
      ...
```

### MDC Setup

To include the username in log output automatically, populate the MDC in a `OncePerRequestFilter` after JWT authentication:

```java
MDC.put("username", authentication.getName());
// ... proceed with filter chain ...
MDC.clear(); // always clear in finally block
```

Add `%X{username}` to the Logback pattern in `logback-spring.xml`.

### What Not to Log

- Never log request bodies that contain `password`, `token`, `secret`, or `Authorization` header values.
- Never log full JWT strings — log only the subject claim or username.
- Filter sensitive fields using a `MaskingPatternLayout` or a `Converter` in Logback if request body logging is needed for debugging.
