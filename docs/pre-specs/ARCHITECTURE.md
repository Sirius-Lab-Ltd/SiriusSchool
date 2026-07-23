# Architecture — SiriusSchool

> **Source:** SRS §2 (Architecture Overview)
> **Status:** Pre-spec — decisions before implementation

## Repository Structure (Monorepo)

```
sirius-school/
├── docs/                    # Documentation (SRS, PRD, API specs, DB)
│   ├── pre-specs/           # Pre-specification documents
│   ├── specs/               # OpenAPI / machine-readable specs
│   ├── PRD/
│   ├── SRS/
│   ├── DB/
│   └── design/
├── packages/
│   └── shared/              # Shared code: Zod schemas, TypeScript types, constants
│       ├── src/schemas/      # Zod validation schemas (1 per module, mirroring OpenAPI)
│       ├── src/types/        # TypeScript types derived from Prisma
│       └── src/constants/    # Enums, error codes, config
├── backend/
│   ├── prisma/              # Prisma schema + migrations
│   ├── src/
│   │   ├── modules/         # Feature modules (one folder per SRS module)
│   │   │   ├── platform/
│   │   │   ├── auth/
│   │   │   ├── academic/
│   │   │   ├── settings/
│   │   │   ├── user-permission/
│   │   │   ├── module-management/
│   │   │   ├── student/
│   │   │   ├── attendance/
│   │   │   ├── result/
│   │   │   ├── notice/
│   │   │   ├── notification/
│   │   │   ├── reports/
│   │   │   ├── dashboard/
│   │   │   └── audit/
│   │   ├── middleware/       # tenant-context, auth, validation, error-handler
│   │   ├── lib/             # Shared utilities (jwt, sms, cloudinary, pdf)
│   │   └── app.ts           # Express app setup (routes, middleware, error handler)
│   └── tests/
├── frontend/
│   ├── src/
│   │   ├── pages/           # Refine pages per module
│   │   ├── components/      # Shared UI (layouts, forms, tables)
│   │   ├── providers/       # Refine data providers (REST, auth, auditLog)
│   │   ├── lib/             # API client, hooks
│   │   └── App.tsx
│   └── tests/
├── docker-compose.yml       # PostgreSQL + app services
└── package.json             # Workspace root (npm workspaces)
```

## Multi-Tenant Model

- **Strategy:** Shared database with `tenant_id` column on every tenant-scoped table
- **Enforcement:** Middleware injects `tenant_id` into every query — never trusts user input
- **Non-tenant tables:** `platform_admins`, `tenants` — no `tenant_id`
- **Cross-tenant access:** Prohibited at every layer (middleware, service, query)

```
Platform Admin (admin.sirius-skool.com)
└── Tenant A (school-a.sirius-skool.com)
│   └── Users (School Admin + Managers)
│   └── Students, Attendance, Results, Notices, etc.
└── Tenant B (school-b.sirius-skool.com)
    └── Users, Students, etc.
```

## User Roles & Hierarchy

| Role | Scope | Auth Table | Login URL |
|------|-------|------------|-----------|
| Platform Admin | All tenants | `platform_admins` | `admin.sirius-skool.com` |
| School Admin | Single tenant | `users` (role: `SCHOOL_ADMIN`) | `{tenant}.sirius-skool.com` |
| Manager | Single tenant | `users` (role: `MANAGER`) | `{tenant}.sirius-skool.com` |

## Permission Model (Two-Tier)

| Tier | Assigner → Assignee | Granularity | Effect |
|------|--------------------|-------------|--------|
| 1 | Platform Admin → School Admin | Module-level | If module assigned, School Admin gets all actions |
| 2 | School Admin → Manager | Module-level (single toggle) | Manager gets predefined actions (V/C/E for most; View-only for Reports; Delete is Admin-only) |

- **School Admin:** Full access to all enabled modules — no explicit permission records
- **Manager:** Needs explicit `manager_permissions` record per granted module

## Module Isolation Rules

- Each module in `backend/src/modules/` is self-contained (routes, controller, service, validation, tests)
- Modules communicate only through the service layer — no cross-module route coupling
- Shared logic (JWT, tenant context, audit logging) lives in `middleware/` or `lib/`
- Each module folder follows the same internal structure:
  ```
  modules/{name}/
  ├── {name}.routes.ts       # Express router
  ├── {name}.controller.ts   # Request/response handling, validation
  ├── {name}.service.ts      # Business logic, Prisma queries
  └── {name}.types.ts        # Module-specific types
  ```

## Middleware Pipeline (order matters)

```
Request
  → CORS (helmet)
  → Rate Limiter (express-rate-limit)
  → Body Parser (JSON)
  → Tenant Context Resolver (extracts tenant from subdomain/header)
  → Auth Middleware (JWT verification)
  → Role Checker (Platform Admin? School Admin? Manager?)
  → Permission Checker (module-level for Manager)
  → Request Validation (Zod schema)
  → Route Handler (controller)
  → Response Envelope (wrap in { data, meta, error })
  → Error Handler (catch-all, format error response)
  → Audit Logger (async, post-response)
  → Response
```

## Response Envelope

```typescript
{
  data: T | null,
  meta: {
    page?: number,
    limit?: number,
    total?: number,
    timestamp: string  // ISO 8601
  } | null,
  error: {
    code: string,      // machine-readable (e.g. "SUBDOMAIN_TAKEN")
    message: string,   // human-readable
    details?: any      // validation errors, etc.
  } | null
}
```

## API Versioning

- Base path: `/api/v1/`
- Platform Admin paths: `/api/v1/admin/*`
- Tenant user paths: `/api/v1/{resource}`

## Key Dependencies Between Modules

```
01-platform   (no deps)
02-auth       (depends on: platform — tenant context)
03-academic   (no deps)
04-settings   (depends on: platform — tenant context)
05-user       (depends on: auth, academic)
06-module     (depends on: platform, auth)
07-student    (depends on: academic, settings)
08-attendance (depends on: academic, student)
09-result     (depends on: academic, student)
10-notice     (depends on: auth)
11-notify     (depends on: auth)
12-reports    (depends on: student, result, attendance)
13-dashboard  (depends on: all above)
14-audit      (cross-cutting, implemented alongside)
```
