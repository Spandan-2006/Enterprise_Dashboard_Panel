# App Flow

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Status:** Approved

This document describes the end-to-end user flows through the application — screen by screen, action by action — and the deployment state machine that drives the core lifecycle. It is role-sensitive: flows change based on the authenticated user's role.

---

## 1. Authentication Flow

### 1.1 Login

```
User lands on /login (or is redirected from a protected route)
    │
    ▼
Fill LoginForm (username, password)
    │
    ▼
POST /api/auth/login
    ├── 200 OK
    │       │
    │       ▼
    │   Store tokens in localStorage:
    │     enterprise_dash_token  (access, 24h)
    │     enterprise_dash_refresh (refresh, 7d)
    │   Set AuthContext: { user, token, isAuthenticated: true }
    │       │
    │       ▼
    │   Navigate to /dashboard
    │
    └── 401 Unauthorized
            │
            ▼
        Show inline error: "Invalid username or password."
        Stay on /login
```

### 1.2 Session Restoration (App Mount)

```
App loads → AuthProvider useEffect runs
    │
    ├── No token in localStorage → isAuthenticated = false → redirect to /login
    │
    └── Token found
            │
            ├── Token not expired (exp > now) → decode sub + role → set user, isAuthenticated = true
            │
            └── Token expired
                    │
                    ├── Refresh token present → POST /api/auth/refresh
                    │       ├── Success → write new access token → set user
                    │       └── Failure → clear both tokens → redirect to /login
                    │
                    └── No refresh token → clear access token → redirect to /login
```

### 1.3 Logout

```
User clicks logout in UserMenu
    │
    ▼
AuthContext.logout()
    → clear enterprise_dash_token + enterprise_dash_refresh from localStorage
    → dispatch LOGOUT action
    → navigate to /login
```

### 1.4 Token Expiry During Session

```
Any API call returns 401
    │
    ▼
Axios response interceptor:
    → clear both tokens from localStorage
    → window.location.href = '/login'
    (session-expired; unsaved form data is lost)
```

---

## 2. Route Structure & Access Control

```
/                     → redirect to /dashboard (auth) or /login (no auth)
/login                → public; unauthenticated only
/dashboard            → ALL authenticated roles
/deployments          → ALL authenticated roles
/deployments/:id      → ALL authenticated roles
/deployments/new      → ADMIN, DEVOPS_ENGINEER only
/deployments/:id/rollback → ADMIN, DEVOPS_ENGINEER only
/audit                → ADMIN only
/users                → ADMIN only
/forbidden            → rendered on 403 response
/*                    → 404 Not Found page
```

### Route Guard Logic

```
PrivateRoute:
  isAuthenticated? → render <Outlet />
  no             → <Navigate to="/login" replace />

RoleGuard (allowedRoles prop):
  user.role in allowedRoles? → render <Outlet />
  no                        → <Navigate to="/forbidden" replace />
```

---

## 3. Dashboard Flow

```
User navigates to /dashboard
    │
    ▼
Parallel API calls:
  GET /api/dashboard/stats
  GET /api/dashboard/success-rate
  GET /api/dashboard/recent
    │
    ├── Loading → show skeleton cards + spinner in chart areas
    │
    └── Loaded
            │
            ▼
        Display:
          ┌─────────────────────────────────────────────────────┐
          │  [Total]  [Successful Today]  [Failed Today]  [Rollbacks] │  ← 4 StatsCards
          ├─────────────────────────────────────────────────────┤
          │  [Deployment Status Donut]   [Success Rate Line]          │  ← Charts
          ├─────────────────────────────────────────────────────┤
          │  Recent Deployments Table (last 10)                       │  ← With StatusBadge per row
          └─────────────────────────────────────────────────────┘
```

---

## 4. Deployment List Flow

```
User navigates to /deployments
    │
    ▼
GET /api/deployments?page=0&size=20&sort=startTime,desc
    │
    ├── Loading → 5 skeleton rows with animate-pulse
    │
    └── Loaded → render DeploymentTable
            │
            ├── Each row: [project, version, environment, StatusBadge, deployedBy, startTime, duration]
            │
            ├── FilterBar (environment, status, projectName, dateRange)
            │       │
            │       └── On change → re-fetch with new query params
            │
            ├── Pagination controls
            │       └── On page change → re-fetch
            │
            └── [New Deployment] button (visible to ADMIN, DEVOPS_ENGINEER only; hidden for DEVELOPER)
                    └── → navigate to /deployments/new
```

### Empty States

| Trigger | Message |
|---------|---------|
| No deployments in DB | "No deployments found. Create your first deployment." |
| Filter returns 0 results | "No deployments match your filters." + "Clear filters" link |

---

## 5. Deployment Detail Flow

```
User clicks a row → navigate to /deployments/:id
    │
    ▼
Parallel API calls:
  GET /api/deployments/:id
  GET /api/deployments/:id/rollbacks
    │
    └── Loaded
            │
            ▼
        DeploymentCard:
          project, version, environment, StatusBadge, deployedBy
          startTime, endTime, durationSeconds, remarks

        StatusTimeline:
          Visual progression: PENDING → BUILDING → TESTING → DEPLOYING → SUCCESSFUL/FAILED/ROLLED_BACK
          (Built from audit log entries for this deployment)

        RollbackHistory:
          Table of rollback records (previousVersion, reason, time)
          Empty state: "No rollbacks recorded for this deployment."

        RollbackButton (ADMIN, DEVOPS_ENGINEER only):
          Shown only when deployment.status === 'FAILED'
          → navigate to /deployments/:id/rollback
```

### Status Update Flow (ADMIN / DEVOPS_ENGINEER)

```
On Deployment Detail page → [Update Status] action
    │
    ▼
Select new status from dropdown (state machine valid options only)
    │
    ▼
PATCH /api/deployments/:id/status  { status: "BUILDING" }
    ├── 200 OK  → refresh deployment card, update StatusBadge
    └── 422     → show error: "Cannot transition from {from} to {to}"
```

---

## 6. Create Deployment Flow (ADMIN / DEVOPS_ENGINEER)

```
Navigate to /deployments/new
    │
    ▼
DeploymentForm:
  projectName (required, max 100)
  version     (required, max 50)
  environment (required — dropdown: DEV, QA, STAGING, PRODUCTION)
  remarks     (optional)
    │
    ▼
[Create Deployment] → POST /api/deployments
    ├── 201 Created
    │       └── navigate to /deployments/:newId (detail page)
    │
    ├── 400 Bad Request
    │       └── show field-level validation errors below each input
    │
    └── 403 Forbidden
            └── redirect to /forbidden
```

---

## 7. Rollback Flow (ADMIN / DEVOPS_ENGINEER)

```
From /deployments/:id (status must be FAILED)
    │
    ▼
Click [Trigger Rollback] → navigate to /deployments/:id/rollback
    │
    ▼
DeploymentSummary:
  shows project, version being reverted, environment

RollbackForm:
  rollbackReason (required, free text)
    │
    ▼
[Confirm Rollback] → ConfirmDialog: "Are you sure? This will mark the deployment as ROLLED_BACK."
    │
    ├── Cancelled → stay on rollback page
    │
    └── Confirmed
            │
            ▼
        POST /api/deployments/:id/rollback  { rollbackReason: "..." }
            ├── 201 Created
            │       └── navigate to /deployments/:id (detail shows ROLLED_BACK status)
            │
            ├── 422 Unprocessable
            │       └── "Deployment is not in FAILED status" error
            │
            └── 400 Bad Request
                    └── "rollbackReason must not be blank" field error
```

---

## 8. User Management Flow (ADMIN only)

```
Navigate to /users
    │
    ▼
GET /api/users?page=0&size=20&sort=createdAt,desc
    │
    └── UserTable: [id, username, email, role, createdAt, actions]

    Per-row actions:
      [Change Role] → inline RoleSelector dropdown
              │
              ▼
          PUT /api/users/:id/role  { role: "DEVOPS_ENGINEER" }
              └── 200 OK → update row in table

      [Delete] → DeleteConfirmDialog
              │
              ▼
          DELETE /api/users/:id
              ├── 204 No Content → remove row from table
              └── 404 Not Found  → show error toast
```

---

## 9. Audit Log Flow (ADMIN only)

```
Navigate to /audit
    │
    ▼
GET /api/audit-logs?page=0&size=20&sort=timestamp,desc
    │
    └── AuditLogTable: [id, username, action, timestamp, description]

    Filters:
      DateRangePicker (startDate, endDate)
      ActionFilter (dropdown of known action constants)
      Username input (exact match)
        │
        └── On filter change → re-fetch with new query params

    Pagination controls → re-fetch on page change
```

---

## 10. Deployment State Machine

The valid status transitions are enforced in `DeploymentService.updateStatus()`. Illegal transitions return HTTP 422.

```
                    ┌─────────┐
        Created ──► │ PENDING │
                    └────┬────┘
                         │ ADMIN / DEVOPS_ENGINEER
                         ▼
                    ┌──────────┐
                    │ BUILDING │ ◄── animate-pulse badge
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ TESTING  │
                    └────┬─────┘
                         │
                    ┌────▼──────┐
                    │ DEPLOYING │ ◄── animate-pulse badge
                    └────┬──────┘
                         │
                    ┌────▼───────┐
                    │ SUCCESSFUL │ ← terminal (endTime set)
                    └────────────┘

    At any active stage (PENDING / BUILDING / TESTING / DEPLOYING):
                         │
                         ▼ error
                    ┌────────┐
                    │ FAILED │ ← terminal (endTime set)
                    └────┬───┘
                         │ Admin / DevOps triggers rollback
                         ▼
                  ┌─────────────┐
                  │ ROLLED_BACK │ ← terminal (endTime set; Rollback record created)
                  └─────────────┘
```

**Terminal state rules:**
- `endTime` is set when status reaches `SUCCESSFUL`, `FAILED`, or `ROLLED_BACK`
- `ROLLED_BACK` can only be reached via `POST /api/deployments/:id/rollback` (not via PATCH status)
- No transitions out of terminal states

---

## 11. Error State Handling

| HTTP Status | User-facing behavior |
|-------------|---------------------|
| 400 | Field-level error messages displayed below each form input in red |
| 401 | Silent redirect to `/login` via Axios interceptor |
| 403 | Redirect to `/forbidden` page with role context and "Back to Dashboard" button |
| 404 | Inline not-found state within content area (preserves sidebar) |
| 422 | Toast or inline error: business rule violation message from API |
| 5xx | Red toast (top-right, 8s auto-dismiss) with server error message or reference ID |
| Network error | Red toast: "Unable to connect to server. Please try again." |
