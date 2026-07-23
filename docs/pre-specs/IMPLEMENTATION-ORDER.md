# Implementation Order — SiriusSchool

> **Source:** SRS §1.2 (Scope — Build Order), SRS §3 (Module Dependencies)
> **Status:** Pre-spec — execution sequence for MVP

## Build Order

| Order | Module | Priority | Depends On | Endpoints | Key Tables |
|-------|--------|----------|------------|-----------|------------|
| 1 | Platform & Multi-Tenant | P0 | None | 8 | tenants, tenant_settings, tenant_modules |
| 2 | Authentication | P0 | Platform | 6 | platform_admins, users, refresh_tokens |
| 3 | Academic Structure | P0 | None | 9 | academic_years, classes, shifts, sections, subjects |
| 4 | Settings | P0 | Platform | 3 | tenant_settings |
| 5 | User & Permission Management | P0 | Auth, Academic | 7 | users, manager_permissions |
| 6 | Module Management | P0 | Platform, Auth | 2 | tenant_modules |
| 7 | Student Management & Admission | P0 | Academic, Settings | 14 | applications, students, student_enrollments |
| 8 | Attendance Management | P0 | Academic, Student | 7 | attendance_sessions, attendance_records |
| 9 | Result Management | P1 | Academic, Student | 9 | exams, exam_subjects, marks, grade_scales |
| 10 | Notice Board | P1 | Auth | 4 | notices |
| 11 | Notification System | P1 | Auth | 2 | notifications |
| 12 | Reports | P1 | Student, Result, Attendance | 5 | (read-only queries) |
| 13 | Dashboard | P0 | All above | 3 | (aggregated queries) |
| 14 | Audit Logging | P0 | (cross-cutting) | 2 | audit_logs |

## Dependency Graph

```
01-platform ───────────────────────────────────────────────────────────┐
   │                                                                    │
02-auth ───────────────────────────────────────────────────────────────┤
   │                                                                    │
03-academic ───────────────────────────────────────────────────────────┤
   │                           ┌────────────────────────────────────────┘
04-settings ──────────────────┤
   │                           │
05-user-permission ───────────┤
   │                           │
06-module-management ─────────┤
   │                           │
07-student ───────────────────┤
   │                           │
08-attendance ────────────────┤
   │                           │
09-result ────────────────────┤
   │                           │
10-notice ────────────────────┤
   │                           │
11-notification ──────────────┤
   │                           │
12-reports ───────────────────┤
   │                           │
13-dashboard ◄────────────────┘
   │
14-audit (cross-cutting, implemented alongside each module)
```

## Execution Strategy

### Phase 1: Foundation (Orders 1-4)
Build the core infrastructure: tenant creation, login/logout, academic hierarchy, settings.

- Prisma schema for all 23 models (from `docs/DB/schema.prisma`)
- Tenant context middleware
- JWT auth middleware
- Error handling middleware
- Response envelope utility

### Phase 2: Administration (Orders 5-6)
User management, role permissions, module toggles.

- Manager CRUD
- Permission assignment
- Module enable/disable

### Phase 3: Core Academics (Orders 7-8)
Student lifecycle and attendance — the primary daily-use features.

- Application workflow (submit → review → approve/reject)
- Student CRUD, registration numbers, roll numbers
- Session-based attendance

### Phase 4: Academic Results (Orders 9-12)
Exam management, marks entry, notices, notifications, reports.

- Exam creation, marks entry, grade calculation
- Notice CRUD, publish/schedule
- SMS/Email notifications
- Report generation

### Phase 5: Cross-Cutting (Orders 13-14)
Dashboard and audit logging — integrate with all modules.

- Role-specific dashboards
- Audit logging for critical actions

## Per-Module Workflow

For each module:

```
1. OpenAPI spec  →  Write/verify OpenAPI YAML for module endpoints
2. Zod schemas   →  Create validation schemas in packages/shared
3. Prisma        →  Verify model exists (from existing schema.prisma)
4. Backend       →  Implement routes, controller, service
5. Contract test →  Write SuperTest assertions against OpenAPI spec
6. Integration   →  Test DB interactions with full middleware pipeline
7. Frontend      →  Implement Refine pages (list, create, edit, show)
8. Validate      →  Run lint, type-check, tests, e2e
```

## Backend Implementation Template

Each module follows the same pattern:

### 1. Routes (`{name}.routes.ts`)

```typescript
import { Router } from 'express';
import { authenticate } from '../middleware/auth';
import { tenantScoped } from '../middleware/tenant';
import { validate } from '../middleware/validate';
import { controller } from './{name}.controller';

const router = Router();

router.use(authenticate);
router.use(tenantScoped);

router.get('/', controller.list);
router.get('/:id', controller.getById);
router.post('/', validate(createSchema), controller.create);
router.patch('/:id', validate(updateSchema), controller.update);
router.delete('/:id', controller.delete);

export default router;
```

### 2. Controller (`{name}.controller.ts`)

```typescript
import { Request, Response, NextFunction } from 'express';
import * as service from './{name}.service';

export const controller = {
  async list(req: Request, res: Response, next: NextFunction) {
    try {
      const result = await service.list(req.tenantId, req.query);
      res.json({ data: result, meta: { total: result.length }, error: null });
    } catch (err) { next(err); }
  },
  // ... create, getById, update, delete
};
```

### 3. Service (`{name}.service.ts`)

```typescript
import { prisma } from '../../lib/prisma';
import { AppError } from '../../lib/errors';

export async function create(tenantId: string, data: CreateInput) {
  // Business logic + Prisma queries
  // Throw AppError for known error cases
  // Wrap multiple operations in transactions
}
```
