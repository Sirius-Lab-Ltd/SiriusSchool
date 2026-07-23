# Tech Stack — SiriusSchool

> **Source:** SRS §2.4 (Tech Stack)
> **Status:** Pre-spec — decisions before implementation

## Backend

| Component | Choice | Version | Rationale |
|-----------|--------|---------|-----------|
| Runtime | Node.js + TypeScript | LTS (22.x) | Team familiarity, ecosystem maturity |
| Framework | Express.js | 4.x | Minimal, well-known, sufficient for REST API |
| ORM | Prisma | 5.x | Type-safe queries, auto-generated types, migrations |
| API Versioning | `/api/v1/` | — | Simple path-prefix versioning |
| Auth | JWT (jsonwebtoken) | 9.x | Stateless, standard; access 15 min + refresh 7 days |
| Password Hash | bcrypt | 5.x | Industry standard for password storage |
| Validation | Zod | 3.x | Shared frontend + backend, TypeScript-first |
| Logging | pino | 8.x | Structured JSON logging, low overhead |
| Rate Limiting | express-rate-limit | 7.x | Simple in-memory rate limiting for MVP |

## Frontend

| Component | Choice | Version | Rationale |
|-----------|--------|---------|-----------|
| Framework | React + TypeScript | 18.x+ | Ecosystem, team skill |
| Admin Framework | Refine | 4.x | Built-in Ant Design, TanStack Query, auth, routing |
| UI Library | Ant Design (antd) | 5.x | Comprehensive component library for admin panels |
| Build Tool | Vite | 5.x | Fast HMR, modern build pipeline |
| HTTP Client | Axios | 1.x | Interceptors, cancellation, request/response transforms |
| Form Handling | React Hook Form | 7.x | Performant, minimal re-renders |
| Validation | Zod (shared) | 3.x | Same schemas as backend — single source of truth |
| Server State | TanStack Query (via Refine) | 5.x | Caching, refetching, optimistic updates |

## Database

| Component | Choice | Version | Rationale |
|-----------|--------|---------|-----------|
| Engine | PostgreSQL | 16.x | Reliable, JSONB support, UUID support |
| PK Type | UUID | — | `@default(uuid()) @db.Uuid` — globally unique, no sequential leaks |
| Naming | snake_case | — | Tables via `@@map`, columns in schema |
| Timestamps | TIMESTAMPTZ | — | `created_at` + `updated_at` on all tables |
| Audit Storage | JSONB | — | `old_values` / `new_values` in audit_logs |

## Infrastructure

| Component | Decision | Notes |
|-----------|----------|-------|
| Monorepo | npm workspaces | `/backend`, `/frontend`, `/packages/shared` |
| Containerization | Docker Compose | PostgreSQL + app (development) |
| PDF Generation | TBD | Evaluate: Puppeteer, jsPDF, pdfmake |
| Excel Generation | TBD | Evaluate: ExcelJS, xlsx |
| SMS Provider | TBD | Evaluate: Twilio, Vonage |
| Email Provider | TBD | Evaluate: SendGrid, AWS SES |
| File Storage | Cloudinary | Student photos, notice files, logos |
| Caching | None (MVP) | Redis post-MVP for permission lookups, class lists |
| Queue | DB-based polling | `notifications` table for MVP; Bull/BullMQ post-MVP |
| Deployment | TBD | Evaluate: Railway, Render, Fly.io |
| CI/CD | GitHub Actions | Lint → Test → Build → Deploy |

## Shared Package (`packages/shared`)

| Artifact | Contents | Used By |
|----------|----------|---------|
| Zod schemas | Request/response validation per module | Backend (middleware), Frontend (forms) |
| TypeScript types | `ApiResponse<T>`, `PaginatedResponse`, enums | Backend + Frontend |
| Constants | Error codes, module enums, config | Backend + Frontend |

## Why These Choices

- **Express over NestJS:** Simpler, faster to build MVP, sufficient for API scope
- **Prisma over TypeORM:** Better type inference, simpler migrations, active maintenance
- **Refine over React Admin:** More flexible data providers, better Ant Design integration, active community
- **Zod over Joi/Yup:** TypeScript-first, composable, shared across stack
- **No Redis in MVP:** Reduces infrastructure complexity; DB queries are fast enough for 100+ tenants
