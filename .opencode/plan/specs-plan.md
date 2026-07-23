# Specs Build Plan — SiriusSchool

## Goal
Build machine-readable OpenAPI 3.1 specs (YAML) in `docs/specs/api/` and pre-specification documents in `docs/pre-specs/`, derived from the existing `docs/SRS/SRS.md`, `docs/PRD/PRD.md`, and `docs/DB/`.

## Directory Structure to Create

```
docs/
├── pre-specs/
│   ├── ARCHITECTURE.md          # Monorepo layout, module isolation — SRS §2
│   ├── TECH-STACK.md            # Pinned versions, rationale — SRS §2.4
│   ├── CODING-CONVENTIONS.md    # Naming, patterns, error handling — SRS §4
│   ├── TESTING-STRATEGY.md      # Unit→Contract→Integration→E2E — SRS §4, PRD §10
│   └── IMPLEMENTATION-ORDER.md  # Build order, dependency graph — SRS §1.2, §3
│
├── specs/
│   ├── index.yaml               # Root OpenAPI: $ref imports all module + common files
│   ├── common/
│   │   ├── schemas.yaml         # ApiResponse<T> envelope, PaginatedResponse, UUID, DateString
│   │   ├── parameters.yaml      # tenant-id header, page, limit, sort, filter params
│   │   ├── responses.yaml       # Standard error bodies: 400, 401, 403, 404, 409, 422, 500
│   │   └── security.yaml        # Bearer JWT scheme definition
│   └── api/
│       ├── 01-platform.yaml     # Tenant CRUD, registration sequence, activate/deactivate
│       ├── 02-auth.yaml         # Login, logout, refresh, password change, profile, rate limiting
│       ├── 03-academic.yaml     # Academic years, classes, shifts, sections, subjects
│       ├── 04-settings.yaml     # School branding, logo upload, notification prefs (cross-ref FR-NOT-09)
│       ├── 05-user-permission.yaml  # Manager CRUD, module permission toggles
│       ├── 06-module-management.yaml # Module enable/disable per tenant
│       ├── 07-student.yaml      # Application workflow, student CRUD, promotion, roll numbers
│       ├── 08-attendance.yaml   # Session-based attendance (DRAFT → SUBMITTED)
│       ├── 09-result.yaml       # Exams, marks entry, grade scales, rank list, result sheet
│       ├── 10-notice.yaml       # Notice CRUD, schedule publish, archive, soft-delete
│       ├── 11-notification.yaml # SMS/Email gateway, templates, quota, retry
│       ├── 12-reports.yaml      # Progress report, TC, attendance report, result report
│       ├── 13-dashboard.yaml    # Role-specific widgets, counts, quick actions
│       └── 14-audit.yaml        # Append-only audit log queries
│
└── traceability/
    └── FR-to-OpenAPI-mapping.md # Every FR → OpenAPI path → Prisma model → Service method
```

## Workflow Per Module

For each module, follow this order:

### Step 1: Extract from SRS
- Read SRS Appendix A for the module's endpoints
- Note: HTTP method, path, path/query params
- Note: Request body JSON shape (required/optional fields, types, constraints)
- Note: Response JSON shapes per status code (200, 201, 400, 401, 403, 404, 409)
- Note: Error codes (from SRS Appendix D)

### Step 2: Extract from DB Dictionary
- Identify which Prisma models the module touches
- Map request fields → model columns
- Note unique constraints, indexes, relations

### Step 3: Write Common Components First
Before any module spec, ensure `common/` has:
- `ApiResponse`: `{ data: T | null, meta: { page?, limit?, total?, timestamp } | null, error: { code, message, details? } | null }`
- `PaginationParams`: `page, limit, sortBy, sortOrder`
- `ErrorResponse`: per HTTP status code (all 23 codes from SRS Appendix D):
  - 400: INCORRECT_CURRENT_PASSWORD, SAME_PASSWORD, CURRENT_YEAR_EXISTS, INSUFFICIENT_BALANCE, MARKS_INCOMPLETE
  - 401: INVALID_CREDENTIALS, INVALID_REFRESH_TOKEN, TOKEN_EXPIRED, TOKEN_REUSE_DETECTED
  - 402: SMS_QUOTA_EXCEEDED
  - 403: ACCOUNT_INACTIVE, TENANT_INACTIVE, MANAGER_CANNOT_CHANGE_PASSWORD
  - 404: USER_NOT_FOUND, MANAGER_NOT_FOUND
  - 409: SUBDOMAIN_TAKEN, ALREADY_REVIEWED, DUPLICATE_NAME, OVERLAPPING_DATES, EMAIL_EXISTS, ALREADY_SUBMITTED
  - 422: VALIDATION_ERROR
  - 429: RATE_LIMIT_EXCEEDED
- `UUID`, `DateString`, `EmailString`, `PhoneString` reusable formats

### Step 4: Write Each Module YAML
Template for each `XX-module.yaml`:

```yaml
openapi: 3.1.0
info:
  title: SiriusSchool - {Module Name}
  version: 1.0.0
  x-sirius-module: {module-name}
  x-sirius-build-order: {N}
  x-sirius-depends-on: [{dependencies}]
paths:
  /api/v1/{resource}:
    get:
      tags: [{Module Tag}]
      x-sirius-fr-ref: [FR-XXX-01, FR-XXX-02]
      summary: ...
      parameters:
        - $ref: '../common/parameters.yaml#/TenantIdHeader'
        - $ref: '../common/parameters.yaml#/PageParam'
        - $ref: '../common/parameters.yaml#/LimitParam'
      responses:
        '200':
          $ref: '../common/responses.yaml#/Success'
        '401':
          $ref: '../common/responses.yaml#/Unauthorized'
components:
  schemas:
    {Resource}Request:
      type: object
      properties: ...
    {Resource}Response:
      allOf:
        - $ref: '../common/schemas.yaml#/ApiResponse'
        - type: object
          properties:
            data: { $ref: '#/components/schemas/{Resource}' }
```

### Step 5: Compose via index.yaml
`index.yaml` is the entry point:

```yaml
openapi: 3.1.0
info:
  title: SiriusSchool API
  version: 1.0.0
servers:
  - url: http://localhost:4000
    description: Development
paths:
  /api/v1/tenants:
    $ref: './api/01-platform.yaml#/paths/~1api~1v1~1tenants'
  /api/v1/tenants/{id}:
    $ref: './api/01-platform.yaml#/paths/~1api~1v1~1tenants~1{id}'
components:
  securitySchemes:
    $ref: './common/security.yaml#/components/securitySchemes'
  schemas:
    ApiResponse: { $ref: './common/schemas.yaml#/ApiResponse' }
    PaginatedResponse: { $ref: './common/schemas.yaml#/PaginatedResponse' }
    ErrorResponse: { $ref: './common/schemas.yaml#/ErrorResponse' }
```

## Quality Gates

| Check | Tool | When |
|-------|------|------|
| Valid YAML | `yaml-lint` | After each file |
| Spec compiles | `npx @redocly/cli lint` | After index.yaml |
| No broken $refs | `npx swagger-cli validate` | After index.yaml |
| All FRs mapped | Manual review vs traceability matrix | After all modules |
| Renders correctly | `npx @redocly/cli preview-docs` | After index.yaml |

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| OpenAPI files | `NN-module-name.yaml` | `07-student.yaml` |
| Schema names | PascalCase | `CreateTenantRequest` |
| Paths | kebab-case | `/api/v1/academic-years` |
| Query params | camelCase | `sortBy`, `sortOrder` |
| $ref paths | relative from file | `../common/schemas.yaml` |
| Tags | PascalCase, plural | `Platform`, `AcademicYears` |
| FR references | `x-sirius-fr-ref` array | `[FR-PLT-01, FR-PLT-02]` |

## Execution Order

1. Create `docs/pre-specs/` — all 5 pre-spec documents
2. Create `docs/specs/common/` — schemas, parameters, responses, security
3. Create `docs/specs/api/` — modules in build order (01 through 14)
4. Create `docs/specs/index.yaml` — compose everything
5. Validate with Spectral + swagger-cli
6. Create `docs/traceability/FR-to-OpenAPI-mapping.md`
