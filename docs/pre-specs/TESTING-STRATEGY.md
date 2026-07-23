# Testing Strategy — SiriusSchool

> **Source:** SRS §4 (Non-Functional Requirements), PRD §10
> **Status:** Pre-spec — decisions before implementation

## Test Pyramid

```
         ╱╲
        ╱ E2E ╲           ← 5% — critical user journeys
       ╱────────╲
      ╱Integration╲        ← 25% — API contract tests, DB interactions
     ╱──────────────╲
    ╱   Unit Tests     ╲   ← 70% — services, validation, utilities
   ╱────────────────────╲
```

## Per-Module Test Requirements

### Unit Tests (Vitest / Jest)

| Target | What to Test | Coverage Target |
|--------|-------------|-----------------|
| Services | Business logic, Prisma query building, transaction logic | 90%+ |
| Validators (Zod) | Schema validation, custom refinements | 100% |
| Utilities | JWT helpers, format functions, error classes | 100% |
| Middleware | Tenant resolver, auth checker, permission checker | 100% |

**Structure:** Each module has `{name}.service.test.ts` next to the service file.

### Contract Tests (SuperTest + OpenAPI)

| Target | What to Test | Tooling |
|--------|-------------|---------|
| API endpoints | Request/response match OpenAPI spec exactly | SuperTest + OpenAPI validator |
| Error codes | Each endpoint returns expected error for invalid inputs | SuperTest |
| Response envelope | `{ data, meta, error }` shape is consistent | SuperTest |

**Approach:**
- Start every module by writing contract tests against the OpenAPI spec
- Assert: status code, response body shape, error code strings
- Tests run against a real test database (not mocked)

```typescript
// Example contract test for POST /api/v1/admin/tenants
describe('POST /api/v1/admin/tenants', () => {
  it('returns 201 with tenant data for valid request', async () => {
    const res = await request(app)
      .post('/api/v1/admin/tenants')
      .set('Authorization', `Bearer ${adminToken}`)
      .send(validTenantPayload);
    expect(res.status).toBe(201);
    expect(res.body).toMatchObject({
      data: { id: expect.any(String), name: 'Test School' },
      meta: null,
      error: null,
    });
  });

  it('returns 409 SUBDOMAIN_TAKEN for duplicate subdomain', async () => {
    const res = await request(app)
      .post('/api/v1/admin/tenants')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ ...validTenantPayload, subdomain: 'existing-school' });
    expect(res.status).toBe(409);
    expect(res.body.error.code).toBe('SUBDOMAIN_TAKEN');
  });
});
```

### Integration Tests

| Target | What to Test | Setup |
|--------|-------------|-------|
| Database | Prisma queries, transactions, unique constraints | Testcontainers (PostgreSQL) or in-memory |
| Middleware pipeline | Full request flow through all middleware | Supertest + test DB |
| Cross-module flows | e.g., create academic year → create class → enroll student | Supertest + test DB |

### E2E Tests (Playwright)

| Flow | Description |
|------|-------------|
| Tenant creation → School Admin login → Dashboard | Full platform admin flow |
| Student admission → attendance → result | Full school academic flow |
| Manager login with restricted permissions | Permission enforcement |

## Test Data Strategy

| Data Type | Approach |
|-----------|----------|
| Prisma seed data | `prisma/seed.ts` — tenants, admin users, academic structure |
| Per-test setup | `beforeEach` creates fresh data using Prisma factories |
| Test database | Dedicated PostgreSQL database (`sirius_school_test`) |
| Teardown | Delete all data after each test suite |

## Tools

| Tool | Purpose |
|------|---------|
| Vitest | Test runner (unit + integration) |
| SuperTest | HTTP assertions (contract + integration) |
| Playwright | E2E test runner |
| Prisma | Test data setup/teardown |
| Testcontainers | PostgreSQL in CI (optional, can use service) |

## CI Pipeline (GitHub Actions)

```
push / PR
  → lint (ESLint + Prettier)
  → type-check (tsc --noEmit)
  → unit tests (vitest)
  → contract tests (vitest + supertest)
  → integration tests (vitest + supertest + test DB)
  → OpenAPI spec lint (Redocly/Spectral)
  → build check (vite build + tsc backend)
```

## Quality Gates

| Gate | Where | Blocking? |
|------|-------|-----------|
| All unit tests pass | PR | Yes |
| Contract tests pass | PR | Yes |
| OpenAPI spec is valid | PR | Yes |
| TypeScript compiles | PR | Yes |
| No lint errors | PR | Yes |
| E2E tests pass | Before release | Yes |
