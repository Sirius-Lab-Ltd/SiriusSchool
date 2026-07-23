# Coding Conventions — SiriusSchool

> **Source:** SRS §4 (Non-Functional Requirements)
> **Status:** Pre-spec — decisions before implementation

## Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Files (modules) | kebab-case | `student-management.routes.ts` |
| Classes/Interfaces | PascalCase | `CreateTenantRequest`, `TenantService` |
| Variables/Functions | camelCase | `getTenantById`, `smsBalance` |
| Database tables | snake_case | `academic_years`, `student_enrollments` |
| Database columns | snake_case | `tenant_id`, `sms_balance`, `is_active` |
| API paths | kebab-case | `/api/v1/academic-years` |
| API query params | camelCase | `sortBy`, `sortOrder`, `page`, `limit` |
| Enums | PascalCase (TS), UPPER_SNAKE (values) | `StudentStatus.ACTIVE` |
| Error codes | UPPER_SNAKE | `SUBDOMAIN_TAKEN`, `INVALID_CREDENTIALS` |
| JSON fields in API | camelCase | `{ "smsBalance": 100 }` |

## File Organization

### Backend — each module folder

```
modules/{name}/
├── {name}.routes.ts       # Express Router — defines endpoints, applies middleware
├── {name}.controller.ts   # Request/response handling — parse params, call service, format response
├── {name}.service.ts      # Business logic — Prisma queries, validations, transactions
└── {name}.types.ts        # Module-specific request/response TypeScript types
```

### Frontend — pages per module

```
pages/
├── {module}/
│   ├── list.tsx            # Refine List page (table)
│   ├── create.tsx          # Refine Create page (form)
│   ├── edit.tsx            # Refine Edit page (form)
│   └── show.tsx            # Refine Show page (detail)
```

## Imports Order

1. External libraries (`express`, `prisma`, `zod`)
2. Internal shared (`@shared/schemas`, `@shared/types`)
3. Internal module (relative imports)
4. Types (`import type { ... }`)

## Error Handling Pattern

```typescript
// service.ts — throw typed errors
class AppError extends Error {
  constructor(
    public statusCode: number,
    public code: string,    // machine-readable, e.g. "SUBDOMAIN_TAKEN"
    message: string,
    public details?: any
  ) {
    super(message);
  }
}

// controller.ts — catch and pass to error handler
try {
  const result = await tenantService.create(data);
  res.status(201).json({ data: result, meta: null, error: null });
} catch (err) {
  next(err);  // central error handler formats response
}

// error-handler.ts middleware — formats { data, meta, error }
```

## Validation (Zod)

- Zod schemas are defined in `packages/shared/src/schemas/` — one file per module
- Same schema used on backend (request validation middleware) and frontend (React Hook Form resolver)
- Schema file mirrors OpenAPI spec for that module
- Example pattern:
  ```typescript
  export const CreateTenantSchema = z.object({
    name: z.string().min(1).max(200),
    subdomain: z.string().min(3).max(50).regex(/^[a-z0-9-]+$/),
    modules: z.array(z.enum([...])),
    smsQuota: z.number().int().min(0),
    startingSequence: z.number().int().min(1),
    schoolAdmin: z.object({
      fullName: z.string().min(1),
      email: z.string().email(),
      password: z.string().min(8),
    }),
  });
  ```

## API Response Format

All responses follow the envelope:

```typescript
// Success (single)
{ data: { ... }, meta: null, error: null }

// Success (list/paginated)
{ data: [...], meta: { page: 1, limit: 20, total: 100, timestamp: "2026-07-24T..." }, error: null }

// Error
{ data: null, meta: null, error: { code: "SUBDOMAIN_TAKEN", message: "...", details: null } }
```

## Error Codes

All error codes are defined in `packages/shared/src/constants/error-codes.ts` and used by both backend and frontend. See SRS Appendix D for the full list (23 codes).

## Security Conventions

| Rule | Implementation |
|------|---------------|
| Tenant scoping | Middleware reads `x-tenant-id` header or subdomain, injects into `req.tenantId` |
| Auth required | Every route (except login/admission) must have `authenticate` middleware |
| JWT payload | `{ userId, tenantId, role, tokenVersion }` — never include secrets |
| Password policy | Min 8 chars, uppercase + lowercase + digit (Zod validation) |
| Rate limiting | Login: 5/min/IP; general API: 100/min/IP |
| CORS | Whitelist tenant subdomains + admin domain |
| Input sanitization | Zod strips unknown fields, validates all input |

## Git Conventions

| Aspect | Convention |
|--------|-----------|
| Branch naming | `{type}/{module}-{description}` — e.g. `feat/platform-create-tenant` |
| Commit style | Conventional Commits — `feat:`, `fix:`, `chore:`, `docs:` |
| PR scope | One module per PR, or one cross-cutting concern |
