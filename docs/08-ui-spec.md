# UI/UX Specification

**Project:** Enterprise Deployment Dashboard
**Version:** 1.0
**Date:** 2026-07-13
**Status:** Approved

---

## Table of Contents

1. [Scope and Context](#1-scope-and-context)
2. [Technology Stack](#2-technology-stack)
3. [Page Inventory](#3-page-inventory)
4. [Component Hierarchy](#4-component-hierarchy)
5. [Dashboard Stats Cards Specification](#5-dashboard-stats-cards-specification)
6. [Deployment Status Color Mapping](#6-deployment-status-color-mapping)
7. [API Integration Layer](#7-api-integration-layer)
8. [Authentication Flow](#8-authentication-flow)
9. [Responsive Breakpoints](#9-responsive-breakpoints)
10. [Empty States and Error States](#10-empty-states-and-error-states)

---

## 1. Scope and Context

This document specifies the optional React frontend for the Enterprise Deployment Dashboard. The backend REST API is the primary deliverable; it operates fully independently via the documented endpoints (see `docs/06-api-spec.md`) and the Swagger UI at `http://localhost:8080/swagger-ui/index.html`.

This frontend specification covers:

- What pages exist and which roles can access them
- How React components are structured and nested
- How the Axios API layer communicates with the backend
- How JWT authentication is managed in the browser
- How the UI responds to different viewport sizes, empty datasets, and error conditions

Teams choosing not to build the React frontend can interact with the entire system through the REST API alone.

---

## 2. Technology Stack

| Layer | Library / Tool | Version | Role |
|---|---|---|---|
| UI Framework | React | 18.x | Component model, hooks, rendering |
| Build Tool | Vite | 5.x | Dev server with HMR, production bundler |
| Styling | Tailwind CSS | 3.x | Utility-first CSS; no custom CSS files unless unavoidable |
| HTTP Client | Axios | 1.x | API calls; request and response interceptors for auth and error handling |
| Charts | Chart.js + react-chartjs-2 | 4.x / 5.x | Dashboard doughnut and line charts |
| Routing | React Router | v6 | Client-side routing; `<Outlet>` for nested layouts |
| State Management | React Context + `useReducer` | (built-in) | Auth state and global UI state; no Redux in v1 |
| Form Handling | React controlled components | (built-in) | All forms use controlled inputs; no form library in v1 |

**Why no Redux?** The application's global state surface is small: the authenticated user/token, a global error toast, and per-page filter state. React Context with `useReducer` is sufficient and avoids the boilerplate of Redux Toolkit. This decision is revisited in v2 if the state surface grows.

**Why Vite over Create React App?** CRA is no longer maintained as of 2023. Vite provides significantly faster cold-start dev server times and native ES module HMR without ejecting.

---

## 3. Page Inventory

Each row represents a distinct URL-addressable page with its access control, primary API calls, and key components.

| Page | Route | Roles Permitted | Key Components | Primary API Calls |
|---|---|---|---|---|
| Login | `/login` | All unauthenticated | `LoginForm`, `ErrorAlert` | `POST /api/auth/login` |
| Dashboard | `/dashboard` | All authenticated | `StatsCard` (×4), `DeploymentStatusDonut`, `SuccessRateLine`, `RecentDeploymentsTable` | `GET /api/dashboard/stats`, `GET /api/dashboard/success-rate`, `GET /api/dashboard/recent` |
| Deployments List | `/deployments` | All authenticated | `FilterBar`, `DeploymentTable`, `Pagination`, `CreateDeploymentButton` | `GET /api/deployments` |
| Deployment Detail | `/deployments/:id` | All authenticated | `DeploymentCard`, `StatusBadge`, `StatusTimeline`, `RollbackHistory`, `RollbackButton` | `GET /api/deployments/:id`, `GET /api/deployments/:id/rollbacks` |
| New Deployment | `/deployments/new` | ADMIN, DEVOPS_ENGINEER | `DeploymentForm`, `EnvSelector`, `VersionInput` | `POST /api/deployments` |
| Rollback | `/deployments/:id/rollback` | ADMIN, DEVOPS_ENGINEER | `DeploymentSummary`, `RollbackForm`, `ConfirmDialog` | `GET /api/deployments/:id`, `POST /api/deployments/:id/rollback` |
| Audit Logs | `/audit` | ADMIN only | `AuditLogTable`, `DateRangePicker`, `ActionFilter` | `GET /api/audit-logs` |
| User Management | `/users` | ADMIN only | `UserTable`, `RoleSelector`, `DeleteConfirmDialog` | `GET /api/users`, `PATCH /api/users/:id/role`, `DELETE /api/users/:id` |

**Route redirect rules:**

- `/` → redirects to `/dashboard` if authenticated, `/login` if not
- Unauthenticated access to any `/dashboard`, `/deployments`, `/audit`, or `/users` route → redirect to `/login`
- DEVELOPER accessing `/deployments/new`, `/deployments/:id/rollback`, `/users`, or `/audit` → redirect to `/forbidden`
- DEVOPS_ENGINEER accessing `/users` or `/audit` → redirect to `/forbidden`

---

## 4. Component Hierarchy

The tree below shows containment (not import) relationships. Every component in the tree corresponds to a single `.jsx` file under `src/components/` or `src/pages/`.

```
App (src/App.jsx)
├── AuthProvider (src/context/AuthContext.jsx)
│   └── Router (React Router BrowserRouter)
│       ├── PublicRoute (src/routes/PublicRoute.jsx)
│       │   └── /login
│       │       └── LoginPage (src/pages/LoginPage.jsx)
│       │           ├── LoginForm (src/components/auth/LoginForm.jsx)
│       │           └── ErrorAlert (src/components/common/ErrorAlert.jsx)
│       │
│       └── PrivateRoute (src/routes/PrivateRoute.jsx)
│           └── Layout (src/components/layout/Layout.jsx)
│               ├── Sidebar (src/components/layout/Sidebar.jsx)
│               │   ├── NavItem (src/components/layout/NavItem.jsx)  [×6 links]
│               │   └── UserBadge (src/components/layout/UserBadge.jsx)
│               ├── TopBar (src/components/layout/TopBar.jsx)
│               │   ├── PageTitle (src/components/layout/PageTitle.jsx)
│               │   └── UserMenu (src/components/layout/UserMenu.jsx)
│               └── ContentArea (<Outlet /> from React Router)
│                   │
│                   ├── /dashboard
│                   │   └── DashboardPage (src/pages/DashboardPage.jsx)
│                   │       ├── StatsCard (src/components/dashboard/StatsCard.jsx)  [×4]
│                   │       │     Total Deployments | Successful Today | Failed Today | Active Rollbacks
│                   │       ├── DeploymentStatusDonut (src/components/dashboard/DeploymentStatusDonut.jsx)
│                   │       │     Chart.js doughnut — breakdown by current status
│                   │       ├── SuccessRateLine (src/components/dashboard/SuccessRateLine.jsx)
│                   │       │     Chart.js line — 7-day success rate trend
│                   │       └── RecentDeploymentsTable (src/components/dashboard/RecentDeploymentsTable.jsx)
│                   │             Last 10 deployments; status badge per row
│                   │
│                   ├── /deployments
│                   │   └── DeploymentsPage (src/pages/DeploymentsPage.jsx)
│                   │       ├── FilterBar (src/components/deployments/FilterBar.jsx)
│                   │       │     Filters: environment, status, projectName, dateRange
│                   │       ├── DeploymentTable (src/components/deployments/DeploymentTable.jsx)
│                   │       │   └── StatusBadge (src/components/common/StatusBadge.jsx)  [per row]
│                   │       ├── Pagination (src/components/common/Pagination.jsx)
│                   │       └── CreateDeploymentButton (src/components/deployments/CreateDeploymentButton.jsx)
│                   │             Visible to ADMIN and DEVOPS_ENGINEER only; hidden for DEVELOPER
│                   │
│                   ├── /deployments/:id
│                   │   └── DeploymentDetailPage (src/pages/DeploymentDetailPage.jsx)
│                   │       ├── DeploymentCard (src/components/deployments/DeploymentCard.jsx)
│                   │       ├── StatusTimeline (src/components/deployments/StatusTimeline.jsx)
│                   │       │     Visual timeline of status transitions from audit log
│                   │       ├── RollbackHistory (src/components/deployments/RollbackHistory.jsx)
│                   │       └── RollbackButton (src/components/deployments/RollbackButton.jsx)
│                   │             Visible to ADMIN and DEVOPS_ENGINEER only; hidden for DEVELOPER
│                   │
│                   ├── /deployments/new  (ADMIN, DEVOPS_ENGINEER only)
│                   │   └── NewDeploymentPage (src/pages/NewDeploymentPage.jsx)
│                   │       └── DeploymentForm (src/components/deployments/DeploymentForm.jsx)
│                   │           ├── EnvSelector (src/components/deployments/EnvSelector.jsx)
│                   │           └── VersionInput (src/components/deployments/VersionInput.jsx)
│                   │
│                   ├── /deployments/:id/rollback  (ADMIN, DEVOPS_ENGINEER only)
│                   │   └── RollbackPage (src/pages/RollbackPage.jsx)
│                   │       ├── DeploymentSummary (src/components/deployments/DeploymentSummary.jsx)
│                   │       ├── RollbackForm (src/components/rollback/RollbackForm.jsx)
│                   │       └── ConfirmDialog (src/components/common/ConfirmDialog.jsx)
│                   │
│                   ├── /audit  (ADMIN only)
│                   │   └── AuditLogsPage (src/pages/AuditLogsPage.jsx)
│                   │       ├── AuditLogTable (src/components/audit/AuditLogTable.jsx)
│                   │       ├── DateRangePicker (src/components/audit/DateRangePicker.jsx)
│                   │       └── ActionFilter (src/components/audit/ActionFilter.jsx)
│                   │
│                   └── /users  (ADMIN only)
│                       └── UserManagementPage (src/pages/UserManagementPage.jsx)
│                           ├── UserTable (src/components/users/UserTable.jsx)
│                           ├── RoleSelector (src/components/users/RoleSelector.jsx)
│                           └── DeleteConfirmDialog (src/components/users/DeleteConfirmDialog.jsx)
```

### 4.1 Route Guard Implementation

```
PrivateRoute (src/routes/PrivateRoute.jsx)
  Purpose: Checks isAuthenticated from AuthContext.
  If not authenticated → <Navigate to="/login" replace />
  If authenticated → renders <Outlet />

RoleGuard (src/routes/RoleGuard.jsx)
  Purpose: Checks user.role from AuthContext against allowedRoles prop.
  Usage: <RoleGuard allowedRoles={['ADMIN', 'DEVOPS_ENGINEER']} />
  If role not in allowedRoles → <Navigate to="/forbidden" replace />
  If role permitted → renders <Outlet />
```

Route configuration example:

```jsx
// src/routes/AppRoutes.jsx
<Routes>
  <Route path="/login" element={<LoginPage />} />

  <Route element={<PrivateRoute />}>
    <Route element={<Layout />}>
      <Route path="/dashboard" element={<DashboardPage />} />
      <Route path="/deployments" element={<DeploymentsPage />} />
      <Route path="/deployments/:id" element={<DeploymentDetailPage />} />

      <Route element={<RoleGuard allowedRoles={['ADMIN', 'DEVOPS_ENGINEER']} />}>
        <Route path="/deployments/new" element={<NewDeploymentPage />} />
        <Route path="/deployments/:id/rollback" element={<RollbackPage />} />
      </Route>

      <Route element={<RoleGuard allowedRoles={['ADMIN']} />}>
        <Route path="/audit" element={<AuditLogsPage />} />
        <Route path="/users" element={<UserManagementPage />} />
      </Route>
    </Route>
  </Route>

  <Route path="/forbidden" element={<ForbiddenPage />} />
  <Route path="*" element={<NotFoundPage />} />
</Routes>
```

---

## 5. Dashboard Stats Cards Specification

Four cards are displayed in a responsive grid at the top of the Dashboard page. Each card shows a single KPI with an icon, value, and contextual subtitle.

### 5.1 Card Layout

| Screen Size | Grid Layout |
|---|---|
| Mobile (<768px) | 2 columns × 2 rows (`grid-cols-2`) |
| Desktop (≥768px) | 4 columns × 1 row (`grid-cols-4`) |

### 5.2 Card Definitions

| Card | Icon Color | Value | Subtitle | Data Source |
|---|---|---|---|---|
| **Total Deployments** | Blue (`text-blue-500`) | Count of all deployments in database | "All environments" | `GET /api/dashboard/stats` → `totalDeployments` |
| **Successful Today** | Green (`text-green-500`) | Count of `SUCCESSFUL` deployments in last 24 hours | "+X% vs yesterday" (trend indicator) | `GET /api/dashboard/stats` → `successfulToday`, `successTrend` |
| **Failed Today** | Red (`text-red-500`) | Count of `FAILED` deployments in last 24 hours | Failure rate as percentage (failed / total today × 100) | `GET /api/dashboard/stats` → `failedToday`, `failureRate` |
| **Active Rollbacks** | Orange (`text-orange-500`) | Count of `ROLLED_BACK` deployments in last 24 hours | "Rollbacks initiated today" | `GET /api/dashboard/stats` → `activeRollbacks` |

### 5.3 Trend Indicator Logic (Successful Today card)

```
successTrend = ((successfulToday - successfulYesterday) / max(successfulYesterday, 1)) × 100

Display:
  If successTrend > 0  → green arrow up   "+{successTrend}% vs yesterday"
  If successTrend < 0  → red arrow down   "{successTrend}% vs yesterday"
  If successTrend == 0 → gray dash        "Same as yesterday"
```

### 5.4 StatsCard Component Props

```typescript
interface StatsCardProps {
  title: string;          // Card heading (e.g., "Total Deployments")
  value: number;          // Primary displayed number
  subtitle: string;       // Secondary text below value
  icon: React.ReactNode;  // Icon element (Heroicons or similar)
  iconBgClass: string;    // Tailwind bg class for icon container (e.g., "bg-blue-50")
  iconColorClass: string; // Tailwind text class for icon (e.g., "text-blue-500")
  trend?: {
    value: number;        // Percentage change
    direction: 'up' | 'down' | 'neutral';
  };
}
```

---

## 6. Deployment Status Color Mapping

The `StatusBadge` component (`src/components/common/StatusBadge.jsx`) renders a pill badge for each deployment status. The badge is used in tables, detail cards, and the status timeline.

### 6.1 Status Badge Styles

| Status | Badge Label | Tailwind Classes | Animation |
|---|---|---|---|
| `PENDING` | Pending | `bg-gray-100 text-gray-700 rounded-full px-2.5 py-0.5 text-xs font-medium` | None |
| `BUILDING` | Building | `bg-blue-100 text-blue-700 rounded-full px-2.5 py-0.5 text-xs font-medium animate-pulse` | Pulse |
| `TESTING` | Testing | `bg-indigo-100 text-indigo-700 rounded-full px-2.5 py-0.5 text-xs font-medium` | None |
| `DEPLOYING` | Deploying | `bg-amber-100 text-amber-700 rounded-full px-2.5 py-0.5 text-xs font-medium animate-pulse` | Pulse |
| `SUCCESSFUL` | Successful | `bg-green-100 text-green-700 rounded-full px-2.5 py-0.5 text-xs font-medium` | None |
| `FAILED` | Failed | `bg-red-100 text-red-700 rounded-full px-2.5 py-0.5 text-xs font-medium` | None |
| `ROLLED_BACK` | Rolled Back | `bg-orange-100 text-orange-700 rounded-full px-2.5 py-0.5 text-xs font-medium` | None |

**Note on animated statuses:** `BUILDING` and `DEPLOYING` use `animate-pulse` to communicate that the deployment is actively in progress. This aids at-a-glance scanning in the deployments list. The pulse animation is disabled if the user's OS preference is `prefers-reduced-motion: reduce` — Tailwind's JIT compiler automatically respects this when the `motion-safe:` / `motion-reduce:` variants are used:

```
motion-safe:animate-pulse
```

### 6.2 StatusBadge Implementation Reference

```jsx
// src/components/common/StatusBadge.jsx
const STATUS_STYLES = {
  PENDING:     'bg-gray-100   text-gray-700',
  BUILDING:    'bg-blue-100   text-blue-700   motion-safe:animate-pulse',
  TESTING:     'bg-indigo-100 text-indigo-700',
  DEPLOYING:   'bg-amber-100  text-amber-700  motion-safe:animate-pulse',
  SUCCESSFUL:  'bg-green-100  text-green-700',
  FAILED:      'bg-red-100    text-red-700',
  ROLLED_BACK: 'bg-orange-100 text-orange-700',
};

const STATUS_LABELS = {
  PENDING:     'Pending',
  BUILDING:    'Building',
  TESTING:     'Testing',
  DEPLOYING:   'Deploying',
  SUCCESSFUL:  'Successful',
  FAILED:      'Failed',
  ROLLED_BACK: 'Rolled Back',
};

export function StatusBadge({ status }) {
  const style = STATUS_STYLES[status] ?? 'bg-gray-100 text-gray-700';
  const label = STATUS_LABELS[status] ?? status;
  return (
    <span className={`inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium ${style}`}>
      {label}
    </span>
  );
}
```

### 6.3 Chart.js Color Palette (Doughnut Chart)

The `DeploymentStatusDonut` chart uses these colors to match the badge palette:

| Status | Chart Color (hex) |
|---|---|
| PENDING | `#9CA3AF` (gray-400) |
| BUILDING | `#60A5FA` (blue-400) |
| TESTING | `#818CF8` (indigo-400) |
| DEPLOYING | `#FBBF24` (amber-400) |
| SUCCESSFUL | `#34D399` (green-400) |
| FAILED | `#F87171` (red-400) |
| ROLLED_BACK | `#FB923C` (orange-400) |

---

## 7. API Integration Layer

All API communication is routed through a central Axios instance. Direct `axios.get()` calls in components are forbidden — components import from the API module files below.

### 7.1 File Structure

```
src/
└── api/
    ├── axiosInstance.js     — Base URL, request interceptor, response interceptor
    ├── authApi.js           — Authentication endpoints
    ├── deploymentApi.js     — Deployment CRUD and lifecycle endpoints
    ├── dashboardApi.js      — Dashboard stats and chart endpoints
    ├── userApi.js           — User management endpoints
    └── auditApi.js          — Audit log query endpoint
```

### 7.2 axiosInstance.js

```javascript
// src/api/axiosInstance.js
import axios from 'axios';

const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL ?? 'http://localhost:8080/api',
  headers: { 'Content-Type': 'application/json' },
  timeout: 15000, // 15 s timeout
});

// Request interceptor — attach JWT access token to every outgoing request
axiosInstance.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('enterprise_dash_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor — centralized error handling
axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    const status = error.response?.status;

    if (status === 401) {
      // Token expired or invalid — clear storage and redirect to login
      localStorage.removeItem('enterprise_dash_token');
      localStorage.removeItem('enterprise_dash_refresh');
      window.location.href = '/login';
    }

    if (status === 403) {
      // Authorized but lacking permission — navigate to forbidden page
      window.location.href = '/forbidden';
    }

    if (status >= 500) {
      // Server error — surface a toast with the reference ID from the API response
      const message = error.response?.data?.errors?.[0]?.message ?? 'An unexpected server error occurred.';
      // Dispatch to global toast system (implementation: useToast hook or Context)
      window.dispatchEvent(new CustomEvent('api:server-error', { detail: { message } }));
    }

    return Promise.reject(error);
  }
);

export default axiosInstance;
```

### 7.3 authApi.js

```javascript
// src/api/authApi.js
import axiosInstance from './axiosInstance';

export const login = (credentials) =>
  axiosInstance.post('/auth/login', credentials);
  // credentials: { username: string, password: string }
  // Response: { success, data: { accessToken, refreshToken, tokenType, expiresIn, user } }

export const register = (userData) =>
  axiosInstance.post('/auth/register', userData);
  // userData: { username, email, password, role }
  // Role must be DEVELOPER or DEVOPS_ENGINEER; ADMIN requires existing admin token

export const refreshToken = (token) =>
  axiosInstance.post('/auth/refresh', { refreshToken: token });
  // Response: { success, data: { accessToken, refreshToken, tokenType, expiresIn } }
```

### 7.4 deploymentApi.js

```javascript
// src/api/deploymentApi.js
import axiosInstance from './axiosInstance';

export const getDeployments = (filters = {}, pagination = {}) =>
  axiosInstance.get('/deployments', {
    params: { ...filters, ...pagination },
    // filters: { projectName?, environment?, status?, startDate?, endDate?, deployedBy? }
    // pagination: { page?, size?, sortBy?, sortDir? }
  });

export const createDeployment = (data) =>
  axiosInstance.post('/deployments', data);
  // data: { projectName, version, environment, deployedBy, remarks? }

export const getDeployment = (id) =>
  axiosInstance.get(`/deployments/${id}`);

export const updateDeployment = (id, data) =>
  axiosInstance.put(`/deployments/${id}`, data);

export const deleteDeployment = (id) =>
  axiosInstance.delete(`/deployments/${id}`);

export const updateStatus = (id, status) =>
  axiosInstance.patch(`/deployments/${id}/status`, { status });
  // status: one of PENDING|BUILDING|TESTING|DEPLOYING|SUCCESSFUL|FAILED|ROLLED_BACK

export const triggerRollback = (id, data) =>
  axiosInstance.post(`/deployments/${id}/rollback`, data);
  // data: { previousVersion: string, rollbackReason: string }

export const getRollbacks = (id, pagination = {}) =>
  axiosInstance.get(`/deployments/${id}/rollbacks`, { params: pagination });
```

### 7.5 dashboardApi.js

```javascript
// src/api/dashboardApi.js
import axiosInstance from './axiosInstance';

export const getStats = () =>
  axiosInstance.get('/dashboard/stats');
  // Response: { totalDeployments, successfulToday, failedToday, activeRollbacks,
  //             successTrend, failureRate, successfulYesterday }

export const getSuccessRate = () =>
  axiosInstance.get('/dashboard/success-rate');
  // Response: { data: [{ date, successRate, totalDeployments }] } — last 7 days

export const getRecent = () =>
  axiosInstance.get('/dashboard/recent');
  // Response: paginated list of the 10 most recent deployments
```

### 7.6 userApi.js

```javascript
// src/api/userApi.js
import axiosInstance from './axiosInstance';

export const getUsers = (pagination = {}) =>
  axiosInstance.get('/users', { params: pagination });

export const getUser = (id) =>
  axiosInstance.get(`/users/${id}`);

export const updateRole = (id, role) =>
  axiosInstance.patch(`/users/${id}/role`, { role });
  // role: ADMIN | DEVOPS_ENGINEER | DEVELOPER

export const deleteUser = (id) =>
  axiosInstance.delete(`/users/${id}`);
```

### 7.7 auditApi.js

```javascript
// src/api/auditApi.js
import axiosInstance from './axiosInstance';

export const getAuditLogs = (filters = {}, pagination = {}) =>
  axiosInstance.get('/audit-logs', {
    params: { ...filters, ...pagination },
    // filters: { username?, action?, startDate?, endDate? }
    // pagination: { page?, size?, sortBy?, sortDir? }
  });
```

---

## 8. Authentication Flow

### 8.1 Token Storage

| Key | Storage | Contains |
|---|---|---|
| `enterprise_dash_token` | `localStorage` | JWT access token (24-hour lifetime) |
| `enterprise_dash_refresh` | `localStorage` | JWT refresh token (7-day lifetime) |

`localStorage` was chosen over `sessionStorage` so that the session persists across browser tabs and page refreshes. For enhanced security in production, `httpOnly` cookie-based token storage is the recommended v2 upgrade; `localStorage` is acceptable for the v1 enterprise internal deployment context.

### 8.2 App Mount Token Check

On initial app load (`useEffect` in `AuthProvider`), the following logic executes:

```
1. Read enterprise_dash_token from localStorage
2. If token is absent → set isAuthenticated = false, user = null
3. If token is present:
   a. Decode the JWT payload (base64-decode the middle segment)
   b. Check the `exp` claim (Unix timestamp in seconds)
   c. If exp > Date.now() / 1000 → token is valid
      - Decode `sub` (username) and `role` from the payload
      - Set user = { username, role }, isAuthenticated = true
   d. If exp <= Date.now() / 1000 → token has expired
      - Read enterprise_dash_refresh from localStorage
      - If refresh token is present → attempt POST /api/auth/refresh
        * On success: write new access token and refresh token to localStorage, set user
        * On failure: clear both tokens, set isAuthenticated = false
      - If refresh token is absent → clear access token, set isAuthenticated = false
```

### 8.3 Login Flow

```
User submits LoginForm
  → authApi.login({ username, password })
  → On success:
      localStorage.setItem('enterprise_dash_token', data.accessToken)
      localStorage.setItem('enterprise_dash_refresh', data.refreshToken)
      AuthContext.dispatch({ type: 'LOGIN', payload: { user: data.user, token: data.accessToken } })
      navigate('/dashboard')
  → On 401 error:
      Display ErrorAlert: "Invalid username or password."
  → On network error:
      Display ErrorAlert: "Unable to connect to server. Please try again."
```

### 8.4 Logout Flow

```
User clicks logout in UserMenu
  → AuthContext.logout()
      localStorage.removeItem('enterprise_dash_token')
      localStorage.removeItem('enterprise_dash_refresh')
      AuthContext.dispatch({ type: 'LOGOUT' })
      navigate('/login')
```

Note: The backend `POST /api/auth/logout` endpoint should also be called if the backend supports server-side token invalidation. If a token blacklist is implemented server-side, omitting this call would allow the token to remain valid until its natural expiry.

### 8.5 AuthContext Interface

```typescript
// src/context/AuthContext.jsx — exported context shape
interface AuthContextValue {
  user: {
    id: number;
    username: string;
    email: string;
    role: 'ADMIN' | 'DEVOPS_ENGINEER' | 'DEVELOPER';
  } | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (credentials: { username: string; password: string }) => Promise<void>;
  logout: () => void;
}
```

### 8.6 Role-Based UI Rendering

Components that behave differently by role use the `useAuth` hook to read `user.role`:

```javascript
// Pattern for hiding elements from DEVELOPER role
import { useAuth } from '../context/AuthContext';

export function CreateDeploymentButton() {
  const { user } = useAuth();
  if (user?.role === 'DEVELOPER') return null;
  return <button ...>New Deployment</button>;
}
```

This is a UI convenience only — the backend enforces role authorization on every endpoint. Hiding a button does not prevent a DEVELOPER from calling the API directly.

---

## 9. Responsive Breakpoints

The layout uses Tailwind CSS's built-in breakpoint system. All breakpoints are `min-width` (mobile-first).

| Breakpoint | Tailwind Prefix | Min Width | Layout Behavior |
|---|---|---|---|
| Mobile | (default, no prefix) | < 768px | Sidebar collapses to a hamburger menu (off-canvas drawer); stats cards in 2×2 grid (`grid-cols-2`); tables scroll horizontally with `overflow-x-auto` |
| Tablet | `md:` | 768px | Sidebar shows icons only (no text labels, 64px wide); tables show condensed column set (remove `remarks` and `end_time` columns); stats cards remain 2×2 |
| Desktop | `lg:` | 1024px | Full sidebar with icons and labels (240px wide); stats cards in 4×1 grid (`lg:grid-cols-4`); full table with all columns |

### 9.1 Sidebar Behavior by Breakpoint

```
Mobile (<768px):
  - Sidebar is hidden (translate-x-full)
  - Hamburger button in TopBar toggles an off-canvas drawer
  - Overlay darkens the content area when drawer is open
  - Clicking the overlay or pressing Esc closes the drawer

Tablet (768–1024px):
  - Sidebar is always visible at 64px width (w-16)
  - Only icons are shown; labels are hidden (sr-only for accessibility)
  - Tooltips on hover reveal the link labels

Desktop (>1024px):
  - Sidebar is always visible at 240px width (w-60)
  - Icons and labels are both shown
  - No hamburger button
```

### 9.2 Table Responsive Strategy

Tables use a horizontal scroll container on small screens rather than stacking rows vertically, because deployment data is inherently tabular and collapsing it to cards loses the ability to scan across rows.

```html
<!-- Responsive table wrapper -->
<div class="overflow-x-auto rounded-lg border border-gray-200">
  <table class="min-w-full divide-y divide-gray-200">
    ...
  </table>
</div>
```

Columns hidden on tablet (`md:hidden`): `remarks`, `end_time`
Columns hidden on mobile (default `hidden`, shown `sm:table-cell`): `version`, `environment`

---

## 10. Empty States and Error States

### 10.1 Empty States

Empty states are shown when a page loads successfully but has no data to display. They replace the table or content area, not the entire page.

| Page / Component | Trigger | Message | Action |
|---|---|---|---|
| DeploymentTable | `deployments.length === 0` after a successful API call | "No deployments found. Create your first deployment." | Show `CreateDeploymentButton` (ADMIN/DEVOPS only); no button for DEVELOPER |
| DeploymentTable (filtered) | `deployments.length === 0` after filtering | "No deployments match your filters." | "Clear filters" link that resets all FilterBar values |
| AuditLogTable | `auditLogs.length === 0` after filtering | "No audit events match your filters." | "Clear filters" link |
| RollbackHistory | `rollbacks.length === 0` on deployment detail | "No rollbacks recorded for this deployment." | None (informational only) |

### 10.2 Error States

#### HTTP 401 — Unauthorized

Handled silently by the Axios response interceptor. The user is redirected to `/login` without displaying an error message. Any unsaved form data is lost; this is acceptable for the v1 scope (a session-expired toast can be added in v2).

#### HTTP 403 — Forbidden

The `/forbidden` page is displayed with:

```
Title:    "Access Denied"
Message:  "You don't have permission to view this page."
          "Your current role is: {user.role}"
Action:   [← Back to Dashboard] button → navigate('/dashboard')
```

#### HTTP 404 — Not Found

The `*` catch-all route renders a `NotFoundPage`:

```
Title:    "404 — Page Not Found"
Message:  "The page you're looking for doesn't exist."
Action:   [← Back to Dashboard] button → navigate('/dashboard')
```

Also used when `GET /api/deployments/:id` returns 404 (deployment not found): the `DeploymentDetailPage` renders an inline not-found state within the content area rather than redirecting, preserving the sidebar and breadcrumb context.

#### Network Error (no response)

When Axios receives no response (e.g., `ECONNREFUSED`, timeout), a global toast notification is shown:

```
Toast (red, top-right, auto-dismiss after 8 s):
  "Unable to connect to server. Please try again."
```

Triggered by the Axios response interceptor via `window.dispatchEvent(new CustomEvent('api:server-error', ...))`. A `ToastListener` component at the `App` level subscribes to this event and renders the toast.

#### HTTP 5xx — Server Error

A toast notification surfaces the error message from the API response body:

```
Toast (red, top-right, auto-dismiss after 8 s):
  "{errors[0].message from API response}"
  Fallback: "An unexpected server error occurred."
```

#### Form Validation Errors (HTTP 400)

The API returns a `400 Bad Request` response with an `errors` array:

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    { "field": "version", "message": "Version must not be blank" },
    { "field": "environment", "message": "Environment must be one of: DEV, QA, STAGING, PRODUCTION" }
  ]
}
```

The form component maps this array to field-level error messages displayed directly below the relevant input in red:

```jsx
// Pattern for field-level error display
{errors.version && (
  <p className="mt-1 text-sm text-red-600">{errors.version}</p>
)}
```

Fields with errors receive an error border: `border-red-500 focus:ring-red-500`.

### 10.3 Loading States

| State | Component | Implementation |
|---|---|---|
| Page initial load | Full-page spinner overlay | `LoadingSpinner` component; shown while the primary page data fetch is `pending` |
| Table re-fetch (filter change) | Row skeleton | 5 skeleton rows with `animate-pulse` gray blocks replace the table body during re-fetch |
| Button loading (form submit) | Inline spinner + disabled state | Submit button shows a spinner and becomes `disabled` after click until the API responds |

Loading states use React's `useState` with a `isLoading` flag, set to `true` before the API call and `false` in both the success and error handlers (`finally` block).
