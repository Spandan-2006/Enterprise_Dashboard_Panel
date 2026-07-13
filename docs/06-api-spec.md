# API Specification

**Project:** Enterprise Deployment Dashboard
**Version:** 1.0
**Date:** 2026-07-13
**Status:** Approved

---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication](#2-authentication)
3. [Standard Response Envelope](#3-standard-response-envelope)
4. [Pagination](#4-pagination)
5. [Full Endpoint Catalog](#5-full-endpoint-catalog)
   - [5.1 Authentication Endpoints (3)](#51-authentication-endpoints)
   - [5.2 Deployment Endpoints (8)](#52-deployment-endpoints)
   - [5.3 Dashboard Endpoints (3)](#53-dashboard-endpoints)
   - [5.4 User Management Endpoints (4)](#54-user-management-endpoints)
   - [5.5 Audit Log Endpoints (1)](#55-audit-log-endpoints)
6. [Error Code Reference](#6-error-code-reference)
7. [Query Parameter Reference](#7-query-parameter-reference)

---

## 1. Overview

| Item | Value |
|---|---|
| Base URL (development) | `http://localhost:8080/api` |
| Base URL (production) | `https://<your-domain>/api` |
| API version | v1 (implicit in current URL structure; v2 path prefix planned) |
| Content type | `application/json` (all requests and responses) |
| Swagger UI | `http://localhost:8080/swagger-ui/index.html` |
| OpenAPI JSON | `http://localhost:8080/v3/api-docs` |
| Total endpoints | 19 |

All endpoint paths are relative to the base URL. For example, `POST /api/auth/register` resolves to `http://localhost:8080/api/auth/register` in development.

**Roles:** `ADMIN`, `DEVOPS_ENGINEER`, `DEVELOPER`  
**Environments:** `DEV`, `QA`, `STAGING`, `PRODUCTION`  
**Deployment statuses:** `PENDING`, `BUILDING`, `TESTING`, `DEPLOYING`, `SUCCESSFUL`, `FAILED`, `ROLLED_BACK`

---

## 2. Authentication

### Token-Based (JWT Bearer)

Protected endpoints require a valid JWT access token in the `Authorization` header:

```
Authorization: Bearer <accessToken>
```

| Token | Lifetime | Notes |
|---|---|---|
| Access token | 24 hours (86 400 s) | Short-lived; included in `Authorization` header |
| Refresh token | 7 days (604 800 s) | Used only at `POST /api/auth/refresh`; not sent with normal requests |

**Token refresh flow:**

1. Make a request to any protected endpoint.
2. If the response is `401 Unauthorized`, the access token has expired.
3. Call `POST /api/auth/refresh` with the refresh token to obtain a new access token.
4. If the refresh token is also expired, the user must authenticate again via `POST /api/auth/login`.

**Public endpoints (no token required):**
- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`

---

## 3. Standard Response Envelope

Every response from the API is wrapped in `ApiResponse<T>`:

```json
{
  "success": true,
  "message": "Operation successful",
  "data": { },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

| Field | Type | Description |
|---|---|---|
| `success` | boolean | `true` when the operation completed without error |
| `message` | string | Human-readable summary of the result |
| `data` | object \| array \| null | Response payload; `null` on error or for `204 No Content` responses |
| `timestamp` | ISO-8601 datetime | Server UTC timestamp at time of response |
| `errors` | array \| null | `null` on success; populated with `{field, message}` objects on validation failure |

### Error Envelope

```json
{
  "success": false,
  "message": "Validation failed",
  "data": null,
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": [
    { "field": "projectName", "message": "must not be blank" },
    { "field": "version",     "message": "must not be blank" }
  ]
}
```

For non-field errors (e.g., 404, 409, 500), `errors` is `null` and the `message` field carries the error description:

```json
{
  "success": false,
  "message": "Resource not found: Deployment with id 42",
  "data": null,
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

---

## 4. Pagination

Endpoints that return lists support Spring-style pagination via query parameters.

### Request Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | int | `0` | Zero-based page index |
| `size` | int | `20` | Number of results per page (max 100) |
| `sort` | string | varies by endpoint | Sort field and direction, e.g. `startTime,desc` or `projectName,asc` |

Example: `GET /api/deployments?page=0&size=20&sort=startTime,desc`

### Response Shape

Paginated data is returned inside the `data` field of the standard envelope:

```json
{
  "success": true,
  "message": "Deployments retrieved successfully",
  "data": {
    "content": [ ],
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8,
    "last": false
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

| Field | Type | Description |
|---|---|---|
| `content` | array | The records for the current page |
| `page` | int | Zero-based current page index |
| `size` | int | Requested page size |
| `totalElements` | long | Total number of matching records across all pages |
| `totalPages` | int | Total number of pages at the current page size |
| `last` | boolean | `true` if this is the final page |

---

## 5. Full Endpoint Catalog

---

### 5.1 Authentication Endpoints

---

#### POST /api/auth/register

**Description:** Register a new user account. Issues JWT tokens immediately on successful registration so the client can proceed without a separate login step.

**Authentication required:** None

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "username": "devops1",
  "email": "devops@company.com",
  "password": "SecurePass1!",
  "role": "DEVOPS_ENGINEER"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `username` | string | Yes | 3–50 characters, must be unique |
| `email` | string | Yes | Valid email format, must be unique |
| `password` | string | Yes | 8–100 characters |
| `role` | string | Yes | One of: `ADMIN`, `DEVOPS_ENGINEER`, `DEVELOPER` |

**Success response — 201 Created:**

```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkZXZvcHMxIiwicm9sZSI6IkRFVk9QU19FTkdJTkVFUiIsImlhdCI6MTcyMDg2MjIwMCwiZXhwIjoxNzIwOTQ4NjAwfQ.signature",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkZXZvcHMxIiwiaWF0IjoxNzIwODYyMjAwLCJleHAiOjE3MjE0NjcwMDB9.signature",
    "tokenType": "Bearer",
    "expiresIn": 86400,
    "username": "devops1",
    "role": "DEVOPS_ENGINEER"
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 400 Bad Request | One or more fields fail validation (missing, too short, invalid email format) |
| 409 Conflict | Username or email is already registered |

---

#### POST /api/auth/login

**Description:** Authenticate with username and password. Returns JWT access and refresh tokens on success.

**Authentication required:** None

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "username": "admin",
  "password": "admin123"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `username` | string | Yes | Must not be blank |
| `password` | string | Yes | Must not be blank |

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJBRE1JTiIsImlhdCI6MTcyMDg2MjIwMCwiZXhwIjoxNzIwOTQ4NjAwfQ.signature",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsImlhdCI6MTcyMDg2MjIwMCwiZXhwIjoxNzIxNDY3MDAwfQ.signature",
    "tokenType": "Bearer",
    "expiresIn": 86400,
    "username": "admin",
    "role": "ADMIN"
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 400 Bad Request | `username` or `password` field is missing or blank |
| 401 Unauthorized | Username not found or password does not match |

---

#### POST /api/auth/refresh

**Description:** Exchange a valid refresh token for a new access token. Use this when the access token has expired (401 response from a protected endpoint).

**Authentication required:** None

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsImlhdCI6MTcyMDg2MjIwMCwiZXhwIjoxNzIxNDY3MDAwfQ.signature"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `refreshToken` | string | Yes | Must not be blank; must be a valid JWT |

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Token refreshed successfully",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJBRE1JTiIsImlhdCI6MTcyMDk0ODYwMCwiZXhwIjoxNzIxMDM1MDAwfQ.signature",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsImlhdCI6MTcyMDg2MjIwMCwiZXhwIjoxNzIxNDY3MDAwfQ.signature",
    "tokenType": "Bearer",
    "expiresIn": 86400,
    "username": "admin",
    "role": "ADMIN"
  },
  "timestamp": "2025-07-13T10:48:00Z",
  "errors": null
}
```

**Note:** The refresh token itself is not rotated in v1. The same refresh token is echoed back until it expires.

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Refresh token is expired, malformed, or has an invalid signature |

---

### 5.2 Deployment Endpoints

---

#### GET /api/deployments

**Description:** Returns a paginated list of all deployments. Supports optional filtering by environment, status, project name, and date range. All roles can access this endpoint.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN`, `DEVOPS_ENGINEER`, or `DEVELOPER`

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | int | No (default: 0) | Zero-based page index |
| `size` | int | No (default: 20) | Results per page |
| `sort` | string | No (default: `startTime,desc`) | Sort field and direction |
| `environment` | string | No | Filter by `DEV`, `QA`, `STAGING`, or `PRODUCTION` |
| `status` | string | No | Filter by `PENDING`, `BUILDING`, `TESTING`, `DEPLOYING`, `SUCCESSFUL`, `FAILED`, or `ROLLED_BACK` |
| `projectName` | string | No | Partial match on project name (case-insensitive) |
| `startDate` | date | No | ISO-8601 date (`YYYY-MM-DD`), inclusive lower bound on `startTime` |
| `endDate` | date | No | ISO-8601 date (`YYYY-MM-DD`), inclusive upper bound on `startTime` |

**Example request:**

```
GET /api/deployments?environment=STAGING&status=SUCCESSFUL&page=0&size=10&sort=startTime,desc
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Deployments retrieved successfully",
  "data": {
    "content": [
      {
        "id": 101,
        "projectName": "payment-service",
        "version": "1.4.2",
        "environment": "STAGING",
        "status": "SUCCESSFUL",
        "deployedBy": "devops1",
        "startTime": "2025-07-13T08:00:00Z",
        "endTime": "2025-07-13T08:14:37Z",
        "remarks": "Release candidate",
        "durationSeconds": 877
      },
      {
        "id": 98,
        "projectName": "auth-service",
        "version": "2.1.0",
        "environment": "STAGING",
        "status": "SUCCESSFUL",
        "deployedBy": "devops2",
        "startTime": "2025-07-12T14:22:00Z",
        "endTime": "2025-07-12T14:35:10Z",
        "remarks": null,
        "durationSeconds": 790
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 42,
    "totalPages": 5,
    "last": false
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |

---

#### POST /api/deployments

**Description:** Creates a new deployment record with `PENDING` status. The `deployedBy` field is automatically set to the authenticated user's username.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` or `DEVOPS_ENGINEER`

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "projectName": "payment-service",
  "version": "1.4.2",
  "environment": "STAGING",
  "remarks": "Release candidate — all integration tests passing"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `projectName` | string | Yes | 1–100 characters |
| `version` | string | Yes | 1–50 characters |
| `environment` | string | Yes | One of: `DEV`, `QA`, `STAGING`, `PRODUCTION` |
| `remarks` | string | No | Free text; may be null or omitted |

**Success response — 201 Created:**

```json
{
  "success": true,
  "message": "Deployment created successfully",
  "data": {
    "id": 105,
    "projectName": "payment-service",
    "version": "1.4.2",
    "environment": "STAGING",
    "status": "PENDING",
    "deployedBy": "devops1",
    "startTime": "2025-07-13T10:30:00Z",
    "endTime": null,
    "remarks": "Release candidate — all integration tests passing",
    "durationSeconds": null
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 400 Bad Request | Required fields missing, blank, or exceed max length |
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user has the `DEVELOPER` role |

---

#### GET /api/deployments/{id}

**Description:** Retrieves a single deployment record by its primary key. Available to all authenticated roles.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN`, `DEVOPS_ENGINEER`, or `DEVELOPER`

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | Deployment primary key |

**Example request:**

```
GET /api/deployments/105
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Deployment retrieved successfully",
  "data": {
    "id": 105,
    "projectName": "payment-service",
    "version": "1.4.2",
    "environment": "STAGING",
    "status": "BUILDING",
    "deployedBy": "devops1",
    "startTime": "2025-07-13T10:30:00Z",
    "endTime": null,
    "remarks": "Release candidate — all integration tests passing",
    "durationSeconds": null
  },
  "timestamp": "2025-07-13T10:35:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 404 Not Found | No deployment exists with the given `id` |

---

#### PUT /api/deployments/{id}

**Description:** Updates the `remarks` field of an existing deployment. Status, version, environment, and `deployedBy` are immutable after creation and cannot be changed through this endpoint.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` or `DEVOPS_ENGINEER`

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | Deployment primary key |

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "remarks": "Updated: hotfix for payment timeout included in this build"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `remarks` | string | No | Pass `null` to clear existing remarks; pass a string to set/update |

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Deployment updated successfully",
  "data": {
    "id": 105,
    "projectName": "payment-service",
    "version": "1.4.2",
    "environment": "STAGING",
    "status": "BUILDING",
    "deployedBy": "devops1",
    "startTime": "2025-07-13T10:30:00Z",
    "endTime": null,
    "remarks": "Updated: hotfix for payment timeout included in this build",
    "durationSeconds": null
  },
  "timestamp": "2025-07-13T10:40:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user has the `DEVELOPER` role |
| 404 Not Found | No deployment exists with the given `id` |

---

#### DELETE /api/deployments/{id}

**Description:** Permanently deletes a deployment record. Restricted to ADMIN role. Deletion is blocked if any rollback records reference this deployment (ON DELETE RESTRICT constraint).

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` only

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | Deployment primary key |

**Example request:**

```
DELETE /api/deployments/105
Authorization: Bearer eyJhbGci...
```

**Success response — 204 No Content:**

No response body.

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user is not an ADMIN |
| 404 Not Found | No deployment exists with the given `id` |
| 409 Conflict | Deployment has associated rollback records and cannot be deleted |

---

#### PATCH /api/deployments/{id}/status

**Description:** Transitions a deployment to a new status. Enforces the state machine — only permitted transitions are accepted. Setting a terminal status (`SUCCESSFUL`, `FAILED`, `ROLLED_BACK`) also sets `endTime`.

**Valid transitions:**

| From | To |
|---|---|
| `PENDING` | `BUILDING`, `FAILED` |
| `BUILDING` | `TESTING`, `FAILED` |
| `TESTING` | `DEPLOYING`, `FAILED` |
| `DEPLOYING` | `SUCCESSFUL`, `FAILED` |
| `FAILED` | `ROLLED_BACK` (use `POST /rollback` endpoint instead) |

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` or `DEVOPS_ENGINEER`

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | Deployment primary key |

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "status": "BUILDING"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `status` | string | Yes | Must be a valid `DeploymentStatus` enum value |

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Deployment status updated successfully",
  "data": {
    "id": 105,
    "projectName": "payment-service",
    "version": "1.4.2",
    "environment": "STAGING",
    "status": "BUILDING",
    "deployedBy": "devops1",
    "startTime": "2025-07-13T10:30:00Z",
    "endTime": null,
    "remarks": "Release candidate — all integration tests passing",
    "durationSeconds": null
  },
  "timestamp": "2025-07-13T10:32:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 400 Bad Request | `status` field is missing or not a valid enum value |
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user has the `DEVELOPER` role |
| 404 Not Found | No deployment exists with the given `id` |
| 422 Unprocessable Entity | The requested status transition is not permitted (e.g., `SUCCESSFUL` → `PENDING`) |

---

#### POST /api/deployments/{id}/rollback

**Description:** Triggers a rollback for a deployment currently in `FAILED` status. Creates a `Rollback` record capturing the previous version and reason, then transitions the deployment status to `ROLLED_BACK`.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` or `DEVOPS_ENGINEER`

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | Deployment primary key (must be in `FAILED` status) |

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "rollbackReason": "Critical bug in payment processing — charge duplication under high load"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `rollbackReason` | string | Yes | Must not be blank; free text |

**Success response — 201 Created:**

```json
{
  "success": true,
  "message": "Rollback triggered successfully",
  "data": {
    "id": 23,
    "deploymentId": 105,
    "previousVersion": "1.4.2",
    "rollbackReason": "Critical bug in payment processing — charge duplication under high load",
    "rollbackTime": "2025-07-13T11:05:00Z"
  },
  "timestamp": "2025-07-13T11:05:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 400 Bad Request | `rollbackReason` is missing or blank |
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user has the `DEVELOPER` role |
| 404 Not Found | No deployment exists with the given `id` |
| 422 Unprocessable Entity | Deployment is not in `FAILED` status |

---

#### GET /api/deployments/{id}/rollbacks

**Description:** Returns the paginated rollback history for a specific deployment, ordered by rollback time descending. Available to all authenticated roles.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN`, `DEVOPS_ENGINEER`, or `DEVELOPER`

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | Deployment primary key |

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | int | No (default: 0) | Zero-based page index |
| `size` | int | No (default: 20) | Results per page |
| `sort` | string | No (default: `rollbackTime,desc`) | Sort field and direction |

**Example request:**

```
GET /api/deployments/105/rollbacks?page=0&size=10
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Rollback history retrieved successfully",
  "data": {
    "content": [
      {
        "id": 23,
        "deploymentId": 105,
        "previousVersion": "1.4.2",
        "rollbackReason": "Critical bug in payment processing — charge duplication under high load",
        "rollbackTime": "2025-07-13T11:05:00Z"
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 1,
    "totalPages": 1,
    "last": true
  },
  "timestamp": "2025-07-13T11:10:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 404 Not Found | No deployment exists with the given `id` |

---

### 5.3 Dashboard Endpoints

---

#### GET /api/dashboard/stats

**Description:** Returns an aggregated statistics snapshot including deployment counts broken down by status, overall success rate, and the 5 most recent deployments. Recommended response caching of 30 seconds on the client side.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN`, `DEVOPS_ENGINEER`, or `DEVELOPER`

**Response headers:** `Cache-Control: max-age=30`

**Example request:**

```
GET /api/dashboard/stats
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Dashboard statistics retrieved successfully",
  "data": {
    "totalDeployments": 150,
    "successfulDeployments": 112,
    "failedDeployments": 18,
    "pendingDeployments": 5,
    "rolledBackDeployments": 15,
    "successRate": 82.96,
    "recentDeployments": [
      {
        "id": 150,
        "projectName": "notification-service",
        "version": "3.0.1",
        "environment": "PRODUCTION",
        "status": "SUCCESSFUL",
        "deployedBy": "admin",
        "startTime": "2025-07-13T10:00:00Z",
        "endTime": "2025-07-13T10:11:22Z",
        "remarks": null,
        "durationSeconds": 682
      },
      {
        "id": 149,
        "projectName": "payment-service",
        "version": "1.4.2",
        "environment": "STAGING",
        "status": "ROLLED_BACK",
        "deployedBy": "devops1",
        "startTime": "2025-07-13T09:30:00Z",
        "endTime": "2025-07-13T11:05:00Z",
        "remarks": null,
        "durationSeconds": 5700
      },
      {
        "id": 148,
        "projectName": "auth-service",
        "version": "2.2.0",
        "environment": "QA",
        "status": "TESTING",
        "deployedBy": "devops2",
        "startTime": "2025-07-13T09:00:00Z",
        "endTime": null,
        "remarks": "Full regression suite",
        "durationSeconds": null
      },
      {
        "id": 147,
        "projectName": "inventory-service",
        "version": "0.9.4",
        "environment": "DEV",
        "status": "FAILED",
        "deployedBy": "devops3",
        "startTime": "2025-07-13T08:45:00Z",
        "endTime": "2025-07-13T08:50:12Z",
        "remarks": null,
        "durationSeconds": 312
      },
      {
        "id": 146,
        "projectName": "reporting-service",
        "version": "1.1.0",
        "environment": "PRODUCTION",
        "status": "SUCCESSFUL",
        "deployedBy": "admin",
        "startTime": "2025-07-12T22:00:00Z",
        "endTime": "2025-07-12T22:09:55Z",
        "remarks": "Monthly reporting update",
        "durationSeconds": 595
      }
    ]
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |

---

#### GET /api/dashboard/success-rate

**Description:** Returns the deployment success rate as a percentage. Excludes `PENDING` deployments from the denominator since they have not yet completed. Returns `0.0` if there are no evaluated deployments.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN`, `DEVOPS_ENGINEER`, or `DEVELOPER`

**Example request:**

```
GET /api/dashboard/success-rate
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Success rate retrieved successfully",
  "data": {
    "successRate": 87.5,
    "totalEvaluated": 40
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

| Field | Type | Description |
|---|---|---|
| `successRate` | double | Percentage (0.0–100.0) of evaluated deployments that succeeded |
| `totalEvaluated` | long | Count of all deployments excluding those in `PENDING` status |

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |

---

#### GET /api/dashboard/recent

**Description:** Returns the 10 most recent deployments ordered by start time descending. Useful for a live activity feed on the dashboard.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN`, `DEVOPS_ENGINEER`, or `DEVELOPER`

**Example request:**

```
GET /api/dashboard/recent
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Recent deployments retrieved successfully",
  "data": [
    {
      "id": 150,
      "projectName": "notification-service",
      "version": "3.0.1",
      "environment": "PRODUCTION",
      "status": "SUCCESSFUL",
      "deployedBy": "admin",
      "startTime": "2025-07-13T10:00:00Z",
      "endTime": "2025-07-13T10:11:22Z",
      "remarks": null,
      "durationSeconds": 682
    },
    {
      "id": 149,
      "projectName": "payment-service",
      "version": "1.4.2",
      "environment": "STAGING",
      "status": "ROLLED_BACK",
      "deployedBy": "devops1",
      "startTime": "2025-07-13T09:30:00Z",
      "endTime": "2025-07-13T11:05:00Z",
      "remarks": null,
      "durationSeconds": 5700
    }
  ],
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

The `data` array contains at most 10 items. This endpoint does not support pagination; use `GET /api/deployments` with pagination for full listing.

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |

---

### 5.4 User Management Endpoints

---

#### GET /api/users

**Description:** Returns a paginated list of all registered users. Passwords are never included in any response. Restricted to ADMIN role.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` only

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | int | No (default: 0) | Zero-based page index |
| `size` | int | No (default: 20) | Results per page |
| `sort` | string | No (default: `createdAt,desc`) | Sort field and direction |

**Example request:**

```
GET /api/users?page=0&size=20&sort=createdAt,desc
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Users retrieved successfully",
  "data": {
    "content": [
      {
        "id": 1,
        "username": "admin",
        "email": "admin@company.com",
        "role": "ADMIN",
        "createdAt": "2025-01-01T09:00:00Z"
      },
      {
        "id": 2,
        "username": "devops1",
        "email": "devops@company.com",
        "role": "DEVOPS_ENGINEER",
        "createdAt": "2025-07-13T10:30:00Z"
      },
      {
        "id": 3,
        "username": "dev1",
        "email": "dev1@company.com",
        "role": "DEVELOPER",
        "createdAt": "2025-07-10T14:00:00Z"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 3,
    "totalPages": 1,
    "last": true
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user is not an ADMIN |

---

#### GET /api/users/{id}

**Description:** Retrieves a single user account by primary key. Restricted to ADMIN role.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` only

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | User primary key |

**Example request:**

```
GET /api/users/2
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "User retrieved successfully",
  "data": {
    "id": 2,
    "username": "devops1",
    "email": "devops@company.com",
    "role": "DEVOPS_ENGINEER",
    "createdAt": "2025-07-13T10:30:00Z"
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user is not an ADMIN |
| 404 Not Found | No user exists with the given `id` |

---

#### PUT /api/users/{id}/role

**Description:** Updates the role of an existing user. Triggers a `USER_ROLE_CHANGED` audit log entry capturing both the old and new role values.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` only

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | User primary key |

**Request headers:** `Content-Type: application/json`

**Request body:**

```json
{
  "role": "DEVOPS_ENGINEER"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `role` | string | Yes | One of: `ADMIN`, `DEVOPS_ENGINEER`, `DEVELOPER` |

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "User role updated successfully",
  "data": {
    "id": 3,
    "username": "dev1",
    "email": "dev1@company.com",
    "role": "DEVOPS_ENGINEER",
    "createdAt": "2025-07-10T14:00:00Z"
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 400 Bad Request | `role` field is missing or not a valid enum value |
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user is not an ADMIN |
| 404 Not Found | No user exists with the given `id` |

---

#### DELETE /api/users/{id}

**Description:** Permanently deletes a user account. Triggers a `USER_DELETED` audit log entry. Deployments previously created by this user are unaffected (the `deployedBy` field is denormalized).

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` only

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | long | User primary key |

**Example request:**

```
DELETE /api/users/3
Authorization: Bearer eyJhbGci...
```

**Success response — 204 No Content:**

No response body.

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user is not an ADMIN |
| 404 Not Found | No user exists with the given `id` |

---

### 5.5 Audit Log Endpoints

---

#### GET /api/audit-logs

**Description:** Queries the audit trail with optional filters. All filter parameters are nullable; omitting them returns all audit logs ordered by timestamp descending. Restricted to ADMIN role.

**Authentication required:** Yes — `Authorization: Bearer <accessToken>`

**Required role:** `ADMIN` only

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | int | No (default: 0) | Zero-based page index |
| `size` | int | No (default: 20) | Results per page |
| `sort` | string | No (default: `timestamp,desc`) | Sort field and direction |
| `username` | string | No | Exact match on `username` field |
| `action` | string | No | Exact match on `action` field (e.g. `DEPLOYMENT_CREATED`) |
| `startDate` | ISO-8601 datetime | No | Inclusive lower bound on `timestamp` (e.g. `2025-07-01T00:00:00`) |
| `endDate` | ISO-8601 datetime | No | Inclusive upper bound on `timestamp` (e.g. `2025-07-13T23:59:59`) |

**Example request:**

```
GET /api/audit-logs?username=devops1&action=DEPLOYMENT_CREATED&page=0&size=10
Authorization: Bearer eyJhbGci...
```

**Success response — 200 OK:**

```json
{
  "success": true,
  "message": "Audit logs retrieved successfully",
  "data": {
    "content": [
      {
        "id": 501,
        "username": "devops1",
        "action": "DEPLOYMENT_CREATED",
        "timestamp": "2025-07-13T10:30:00Z",
        "description": "Deployment created: payment-service v1.4.2 to STAGING"
      },
      {
        "id": 499,
        "username": "devops1",
        "action": "DEPLOYMENT_CREATED",
        "timestamp": "2025-07-12T09:15:00Z",
        "description": "Deployment created: auth-service v2.1.0 to QA"
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 2,
    "totalPages": 1,
    "last": true
  },
  "timestamp": "2025-07-13T10:30:00Z",
  "errors": null
}
```

**Error responses:**

| Code | Condition |
|---|---|
| 401 Unauthorized | Missing or invalid access token |
| 403 Forbidden | Authenticated user is not an ADMIN |

---

## 6. Error Code Reference

| Code | Meaning | When Returned |
|---|---|---|
| 400 | Bad Request | Validation failure on request body fields, missing required fields, malformed JSON, or invalid enum value |
| 401 | Unauthorized | Missing `Authorization` header, invalid JWT signature, expired access token, or invalid login credentials |
| 403 | Forbidden | Valid and unexpired JWT, but the authenticated user's role does not have permission for the requested operation |
| 404 | Not Found | The requested resource (deployment, user, rollback) does not exist by the given ID or identifier |
| 409 | Conflict | Duplicate `username` or `email` on registration; attempting to delete a deployment that has associated rollback records |
| 422 | Unprocessable Entity | Business rule violation: invalid deployment status transition, or triggering rollback on a deployment that is not in `FAILED` status |
| 500 | Internal Server Error | Unexpected server-side error; response body includes a `Reference` UUID for log correlation with server-side logs |

---

## 7. Query Parameter Reference

| Parameter | Type | Default | Applicable To | Description |
|---|---|---|---|---|
| `page` | int | `0` | All paginated endpoints | Zero-based page number |
| `size` | int | `20` | All paginated endpoints | Results per page; maximum 100 |
| `sort` | string | endpoint-specific | All paginated endpoints | Sort field and direction (e.g. `startTime,desc` or `projectName,asc`) |
| `environment` | enum | — | `GET /api/deployments` | Filter by environment: `DEV`, `QA`, `STAGING`, or `PRODUCTION` |
| `status` | enum | — | `GET /api/deployments` | Filter by deployment status: `PENDING`, `BUILDING`, `TESTING`, `DEPLOYING`, `SUCCESSFUL`, `FAILED`, `ROLLED_BACK` |
| `projectName` | string | — | `GET /api/deployments` | Case-insensitive partial match (`LIKE '%value%'`) on `project_name` column |
| `startDate` | date (`YYYY-MM-DD`) | — | `GET /api/deployments` | Inclusive lower bound on `startTime`; matches from start of given date |
| `endDate` | date (`YYYY-MM-DD`) | — | `GET /api/deployments` | Inclusive upper bound on `startTime`; matches through end of given date |
| `startDate` | ISO-8601 datetime | — | `GET /api/audit-logs` | Inclusive lower bound on `timestamp` (e.g. `2025-07-01T00:00:00`) |
| `endDate` | ISO-8601 datetime | — | `GET /api/audit-logs` | Inclusive upper bound on `timestamp` (e.g. `2025-07-13T23:59:59`) |
| `username` | string | — | `GET /api/audit-logs` | Exact match (case-sensitive) on `username` field |
| `action` | string | — | `GET /api/audit-logs` | Exact match on `action` field constant (e.g. `LOGIN_SUCCESS`) |
