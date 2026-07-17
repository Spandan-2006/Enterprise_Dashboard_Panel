# UI/UX Brief

**Project:** Enterprise Deployment Dashboard  
**Version:** 1.0  
**Status:** Approved

This document is the design reference for the optional React 18 frontend. The backend REST API operates fully independently — this frontend is additive. Teams working API-only can skip this document.

---

## 1. Frontend Tech Stack

| Layer | Library | Version | Notes |
|-------|---------|---------|-------|
| UI Framework | React | 18.x | Hooks-based; no class components |
| Build Tool | Vite | 5.x | HMR dev server; native ES modules; replaces Create React App (no longer maintained) |
| Styling | Tailwind CSS | 3.x | Utility-first; no hand-written CSS files unless strictly unavoidable |
| HTTP Client | Axios | 1.x | Central `axiosInstance` with request/response interceptors |
| Charts | Chart.js + react-chartjs-2 | 4.x / 5.x | Doughnut (status breakdown) + Line (7-day success rate trend) |
| Routing | React Router | v6 | `<Outlet>` for nested layouts; `useNavigate` for programmatic navigation |
| State | React Context + `useReducer` | built-in | Auth state + global toast state; no Redux in v1 |
| Forms | Controlled components | built-in | No form library; `useState` per field; errors mapped from API 400 response |

---

## 2. Page Inventory

| Page | Route | Roles | Primary API Calls |
|------|-------|-------|-------------------|
| Login | `/login` | unauthenticated | `POST /api/auth/login` |
| Dashboard | `/dashboard` | ALL | `GET /api/dashboard/stats`, `GET /api/dashboard/success-rate`, `GET /api/dashboard/recent` |
| Deployments List | `/deployments` | ALL | `GET /api/deployments` |
| Deployment Detail | `/deployments/:id` | ALL | `GET /api/deployments/:id`, `GET /api/deployments/:id/rollbacks` |
| New Deployment | `/deployments/new` | ADMIN, DEVOPS_ENGINEER | `POST /api/deployments` |
| Rollback | `/deployments/:id/rollback` | ADMIN, DEVOPS_ENGINEER | `POST /api/deployments/:id/rollback` |
| Audit Logs | `/audit` | ADMIN | `GET /api/audit-logs` |
| User Management | `/users` | ADMIN | `GET /api/users`, `PUT /api/users/:id/role`, `DELETE /api/users/:id` |

---

## 3. Component Hierarchy

```
App (src/App.jsx)
├── AuthProvider (src/context/AuthContext.jsx)
│   └── BrowserRouter
│       ├── /login → LoginPage
│       │     ├── LoginForm
│       │     └── ErrorAlert
│       │
│       └── PrivateRoute
│           └── Layout (Sidebar + TopBar + <Outlet />)
│               ├── /dashboard → DashboardPage
│               │     ├── StatsCard [×4]
│               │     ├── DeploymentStatusDonut
│               │     ├── SuccessRateLine
│               │     └── RecentDeploymentsTable
│               │
│               ├── /deployments → DeploymentsPage
│               │     ├── FilterBar
│               │     ├── DeploymentTable
│               │     │     └── StatusBadge [per row]
│               │     ├── Pagination
│               │     └── CreateDeploymentButton (ADMIN/DEVOPS only)
│               │
│               ├── /deployments/:id → DeploymentDetailPage
│               │     ├── DeploymentCard
│               │     ├── StatusTimeline
│               │     ├── RollbackHistory
│               │     └── RollbackButton (ADMIN/DEVOPS only; visible when status=FAILED)
│               │
│               ├── RoleGuard allowedRoles=[ADMIN, DEVOPS_ENGINEER]
│               │     ├── /deployments/new → NewDeploymentPage
│               │     │     └── DeploymentForm (EnvSelector, VersionInput)
│               │     └── /deployments/:id/rollback → RollbackPage
│               │           ├── DeploymentSummary
│               │           ├── RollbackForm
│               │           └── ConfirmDialog
│               │
│               └── RoleGuard allowedRoles=[ADMIN]
│                     ├── /audit → AuditLogsPage
│                     │     ├── AuditLogTable
│                     │     ├── DateRangePicker
│                     │     └── ActionFilter
│                     └── /users → UserManagementPage
│                           ├── UserTable
│                           ├── RoleSelector
│                           └── DeleteConfirmDialog
│
├── /forbidden → ForbiddenPage
└── /* → NotFoundPage
```

### File Layout

```
src/
├── api/
│   ├── axiosInstance.js    — Base URL, request interceptor, response interceptor
│   ├── authApi.js          — register, login, refreshToken
│   ├── deploymentApi.js    — CRUD + status + rollback
│   ├── dashboardApi.js     — stats, successRate, recent
│   ├── userApi.js          — getUsers, getUser, updateRole, deleteUser
│   └── auditApi.js         — getAuditLogs
├── context/
│   └── AuthContext.jsx     — AuthProvider, useAuth hook
├── routes/
│   ├── AppRoutes.jsx
│   ├── PrivateRoute.jsx
│   └── RoleGuard.jsx
├── components/
│   ├── layout/             — Layout, Sidebar, TopBar, NavItem, UserBadge, UserMenu
│   ├── common/             — StatusBadge, Pagination, ErrorAlert, ConfirmDialog, LoadingSpinner
│   ├── dashboard/          — StatsCard, DeploymentStatusDonut, SuccessRateLine, RecentDeploymentsTable
│   ├── deployments/        — DeploymentTable, FilterBar, DeploymentCard, DeploymentForm, StatusTimeline, RollbackButton, RollbackHistory, CreateDeploymentButton
│   ├── rollback/           — RollbackForm, DeploymentSummary
│   ├── audit/              — AuditLogTable, DateRangePicker, ActionFilter
│   └── users/              — UserTable, RoleSelector, DeleteConfirmDialog
└── pages/
    ├── LoginPage.jsx
    ├── DashboardPage.jsx
    ├── DeploymentsPage.jsx
    ├── DeploymentDetailPage.jsx
    ├── NewDeploymentPage.jsx
    ├── RollbackPage.jsx
    ├── AuditLogsPage.jsx
    ├── UserManagementPage.jsx
    ├── ForbiddenPage.jsx
    └── NotFoundPage.jsx
```

---

## 4. Layout & Navigation

### Sidebar Navigation Links

| Link | Route | Icon | Visibility |
|------|-------|------|-----------|
| Dashboard | `/dashboard` | Chart icon | ALL |
| Deployments | `/deployments` | Rocket icon | ALL |
| Audit Logs | `/audit` | Shield icon | ADMIN only |
| User Management | `/users` | Users icon | ADMIN only |

### Responsive Behavior

| Breakpoint | Tailwind | Sidebar | Stats Cards | Tables |
|-----------|---------|---------|-------------|--------|
| Mobile | default (< 768px) | Off-canvas drawer; hamburger toggle in TopBar | `grid-cols-2` (2×2) | Horizontal scroll (`overflow-x-auto`) |
| Tablet | `md:` (768px) | Always visible, icons-only, 64px wide; tooltip labels on hover | `grid-cols-2` | Condensed columns (hide `remarks`, `end_time`) |
| Desktop | `lg:` (1024px+) | Always visible, icons + labels, 240px wide | `grid-cols-4` (4×1) | Full columns |

### Mobile Sidebar

```
Mobile (<768px):
  Sidebar: hidden by default (translate-x-full)
  Hamburger button in TopBar → toggles off-canvas drawer
  Overlay darkens content area when drawer is open
  Click overlay or press Esc → close drawer
```

---

## 5. Status Badge Design

`StatusBadge` is a pill component used in tables, detail cards, and the status timeline.

| Status | Label | Tailwind Classes | Animation |
|--------|-------|-----------------|-----------|
| PENDING | Pending | `bg-gray-100 text-gray-700` | None |
| BUILDING | Building | `bg-blue-100 text-blue-700` | `motion-safe:animate-pulse` |
| TESTING | Testing | `bg-indigo-100 text-indigo-700` | None |
| DEPLOYING | Deploying | `bg-amber-100 text-amber-700` | `motion-safe:animate-pulse` |
| SUCCESSFUL | Successful | `bg-green-100 text-green-700` | None |
| FAILED | Failed | `bg-red-100 text-red-700` | None |
| ROLLED_BACK | Rolled Back | `bg-orange-100 text-orange-700` | None |

`motion-safe:animate-pulse` respects `prefers-reduced-motion: reduce` OS setting automatically.

### StatusBadge Implementation

```jsx
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
  PENDING: 'Pending', BUILDING: 'Building', TESTING: 'Testing',
  DEPLOYING: 'Deploying', SUCCESSFUL: 'Successful', FAILED: 'Failed', ROLLED_BACK: 'Rolled Back',
};

export function StatusBadge({ status }) {
  return (
    <span className={`inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium
      ${STATUS_STYLES[status] ?? 'bg-gray-100 text-gray-700'}`}>
      {STATUS_LABELS[status] ?? status}
    </span>
  );
}
```

---

## 6. Dashboard Stats Cards

Four cards at the top of the Dashboard in a responsive grid.

| Card | Icon Color | Value | Subtitle | API Field |
|------|-----------|-------|----------|-----------|
| Total Deployments | `text-blue-500` | All deployments | "All environments" | `totalDeployments` |
| Successful Today | `text-green-500` | SUCCESSFUL in last 24h | "+X% vs yesterday" | `successfulDeployments` |
| Failed Today | `text-red-500` | FAILED in last 24h | Failure rate % | `failedDeployments` |
| Active Rollbacks | `text-orange-500` | ROLLED_BACK in last 24h | "Rollbacks initiated today" | `rolledBackDeployments` |

### StatsCard Props

```typescript
interface StatsCardProps {
  title: string;
  value: number;
  subtitle: string;
  icon: React.ReactNode;
  iconBgClass: string;    // e.g. "bg-blue-50"
  iconColorClass: string; // e.g. "text-blue-500"
  trend?: { value: number; direction: 'up' | 'down' | 'neutral' };
}
```

---

## 7. Chart Color Palette (Doughnut Chart)

`DeploymentStatusDonut` uses these colors to match the badge palette.

| Status | Hex | Tailwind equivalent |
|--------|-----|---------------------|
| PENDING | `#9CA3AF` | gray-400 |
| BUILDING | `#60A5FA` | blue-400 |
| TESTING | `#818CF8` | indigo-400 |
| DEPLOYING | `#FBBF24` | amber-400 |
| SUCCESSFUL | `#34D399` | green-400 |
| FAILED | `#F87171` | red-400 |
| ROLLED_BACK | `#FB923C` | orange-400 |

---

## 8. API Integration Layer

All HTTP calls go through `src/api/axiosInstance.js`. No component imports `axios` directly.

### axiosInstance.js

```javascript
const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL ?? 'http://localhost:8080/api',
  headers: { 'Content-Type': 'application/json' },
  timeout: 15000,
});

// Request interceptor — attach Bearer token
axiosInstance.interceptors.request.use((config) => {
  const token = localStorage.getItem('enterprise_dash_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor — centralized error handling
axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    const status = error.response?.status;
    if (status === 401) {
      localStorage.removeItem('enterprise_dash_token');
      localStorage.removeItem('enterprise_dash_refresh');
      window.location.href = '/login';
    }
    if (status === 403) window.location.href = '/forbidden';
    if (status >= 500) {
      window.dispatchEvent(new CustomEvent('api:server-error', {
        detail: { message: error.response?.data?.message ?? 'An unexpected server error occurred.' }
      }));
    }
    return Promise.reject(error);
  }
);
```

### Token Storage

| Key | Storage | Contents |
|-----|---------|----------|
| `enterprise_dash_token` | `localStorage` | JWT access token (24h) |
| `enterprise_dash_refresh` | `localStorage` | JWT refresh token (7d) |

---

## 9. Authentication Context

```typescript
interface AuthContextValue {
  user: { id: number; username: string; email: string; role: 'ADMIN' | 'DEVOPS_ENGINEER' | 'DEVELOPER' } | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (credentials: { username: string; password: string }) => Promise<void>;
  logout: () => void;
}
```

### Role-Based UI Rendering

```javascript
import { useAuth } from '../context/AuthContext';

export function CreateDeploymentButton() {
  const { user } = useAuth();
  if (user?.role === 'DEVELOPER') return null;
  return <button>New Deployment</button>;
}
```

This is UI convenience only — the backend enforces authorization on every endpoint.

---

## 10. Loading States

| State | Component | Implementation |
|-------|-----------|----------------|
| Page initial load | `LoadingSpinner` full-page overlay | `isLoading` state; set `true` before fetch, `false` in `finally` |
| Table re-fetch (filter change) | 5 skeleton rows with `animate-pulse` | Replaces table body during re-fetch |
| Form submit | Inline spinner + `disabled` button | Button shows spinner after click until API responds |

---

## 11. Empty & Error States

### Empty States

| Page | Trigger | Message | Action |
|------|---------|---------|--------|
| DeploymentTable | 0 results (no filter) | "No deployments found. Create your first deployment." | Show `CreateDeploymentButton` (ADMIN/DEVOPS only) |
| DeploymentTable | 0 results (filtered) | "No deployments match your filters." | "Clear filters" link |
| AuditLogTable | 0 results | "No audit events match your filters." | "Clear filters" link |
| RollbackHistory | 0 rollbacks | "No rollbacks recorded for this deployment." | — |

### Error Pages

**403 Forbidden:**
```
Title:   "Access Denied"
Message: "You don't have permission to view this page."
         "Your current role is: {user.role}"
Action:  [← Back to Dashboard]
```

**404 Not Found:**
```
Title:   "404 — Page Not Found"
Message: "The page you're looking for doesn't exist."
Action:  [← Back to Dashboard]
```

### Form Validation (HTTP 400)

```jsx
// Map API errors array to field-level display
{errors.version && (
  <p className="mt-1 text-sm text-red-600">{errors.version}</p>
)}
// Error border on the input
className={`border ${errors.version ? 'border-red-500' : 'border-gray-300'}`}
```

---

## 12. Table Column Visibility by Breakpoint

| Column | Mobile | Tablet | Desktop |
|--------|--------|--------|---------|
| project_name | Show | Show | Show |
| version | Hidden | Show | Show |
| environment | Hidden | Show | Show |
| status (badge) | Show | Show | Show |
| deployed_by | Hidden | Hidden | Show |
| start_time | Show | Show | Show |
| end_time / duration | Hidden | Hidden | Show |
| remarks | Hidden | Hidden | Show |

```html
<!-- Responsive table wrapper -->
<div class="overflow-x-auto rounded-lg border border-gray-200">
  <table class="min-w-full divide-y divide-gray-200">...</table>
</div>
```
