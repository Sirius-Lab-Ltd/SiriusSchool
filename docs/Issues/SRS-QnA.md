# SRS — Questions for Developer Team

> **Purpose:** These questions need team input to finalize the SRS and begin implementation.
> **Status:** Open
> **Date:** July 6, 2026

---

## Table of Contents

1. [Tech Stack](#1-tech-stack)
2. [API Design](#2-api-design)
3. [Database Design](#3-database-design)
4. [Authentication & Security](#4-authentication--security)
5. [Frontend / UI](#5-frontend--ui)
6. [File Storage](#6-file-storage)
7. [Testing](#7-testing)
8. [Deployment & Infrastructure](#8-deployment--infrastructure)
9. [Code Conventions](#9-code-conventions)
10. [Project Structure](#10-project-structure)
11. [Error Handling & Logging](#11-error-handling--logging)
12. [Module Planning](#12-module-planning)

---

## 1. Tech Stack

### 1.1 Backend Framework

| # | Question | Options / Notes |
|---|---|---|
| 1 | Which backend framework should we use? | Laravel (PHP), NestJS/Express (Node.js), Django (Python), ASP.NET Core, other? |
| 2 | What are the team's strongest skills? | Consider team expertise for faster delivery |
| 3 | Do we need real-time capabilities (WebSockets) for MVP? | e.g., live dashboard updates, notification badges |
| 4 | Should we use an ORM or raw queries? | Eloquent (Laravel), Prisma/TypeORM, Django ORM, Dapper, etc. |
| 5 | What version of the language/runtime? | PHP 8.x, Node 20/22, Python 3.12, .NET 8/9 |

### 1.2 Frontend Framework

| # | Question | Options / Notes |
|---|---|---|
| 6 | Which frontend framework should we use? | React, Vue 3, Svelte, Angular, Livewire (if Laravel), or server-side rendering? |
| 7 | SPA or SSR or hybrid? | Single Page App, Server-Side Rendered, or both (Next.js/Nuxt) |
| 8 | Which CSS/UI library? | Tailwind CSS, Bootstrap, Shadcn/ui, MUI (Material UI), Ant Design, DaisyUI |
| 9 | Do we need a component library? | Pre-built components or build from scratch? |
| 10 | TypeScript or JavaScript? | Strongly recommended: TypeScript for both frontend and backend (if Node.js) |

### 1.3 Database

| # | Question | Options / Notes |
|---|---|---|
| 11 | Which database engine? | PostgreSQL (recommended for multi-tenant), MySQL 8, SQL Server |
| 12 | Why this choice? | PostgreSQL: better JSON support, CTEs, array types, robust indexing for multi-tenant queries |
| 13 | Do we need MongoDB for any part? | e.g., audit logs, temp admission data, notification logs |
| 14 | ORM vs raw SQL for complex queries? | e.g., attendance reports, result calculations |

### 1.4 Caching & Queues

| # | Question | Options / Notes |
|---|---|---|
| 15 | Which cache store? | Redis (recommended), Memcached |
| 16 | What should be cached? | Permission lookups, tenant settings, class/section lists, user sessions |
| 17 | Do we need a queue system for MVP? | For SMS/email notifications (async), report generation |
| 18 | Which queue driver? | Redis, RabbitMQ, Amazon SQS, database queue |

---

## 2. API Design

| # | Question | Options / Notes |
|---|---|---|
| 19 | API style? | REST (recommended), GraphQL, or mixed? |
| 20 | API versioning strategy? | URL prefix (`/api/v1/`), header (`Accept: application/vnd.sirius.v1+json`), or both? |
| 21 | Response envelope format? | `{ data: ..., meta: ..., error: ... }` or bare response? |
| 22 | Pagination style? | Cursor-based (recommended for large datasets) or offset/limit? |
| 23 | Pagination response format? | Include `next_cursor`, `prev_cursor`, `total`, `per_page`? |
| 24 | Should we document APIs with OpenAPI/Swagger? | Auto-generated from code or hand-written? |
| 25 | Naming convention for endpoints? | `snake_case` or `camelCase`? Plural or singular? e.g., `/api/v1/students` vs `/api/v1/student` |
| 26 | Should responses include included/relationships (like JSON:API)? | For nested resources (e.g., student with enrollments) |

---

## 3. Database Design

| # | Question | Options / Notes |
|---|---|---|
| 27 | Naming convention for tables? | `snake_case` plural (e.g., `students`, `student_enrollments`) |
| 28 | Naming convention for columns? | `snake_case` (e.g., `tenant_id`, `registration_number`) |
| 29 | Primary key type? | Auto-increment integer or UUID? (UUID recommended for multi-tenant to avoid ID guessing) |
| 30 | Timestamp columns? | `created_at`, `updated_at` on all tables? Soft delete with `deleted_at`? |
| 31 | Migrations strategy? | Who writes migrations? How are they reviewed? |
| 32 | Seed data for development? | Demo tenant with sample classes, sections, students? |
| 33 | How to document the schema? | ER diagram tool? (Mermaid, dbdiagram.io, MySQL Workbench) |
| 34 | Audit logging — table structure? | Single `audit_logs` table or per-module tables? |

### Example DB Schema Format — for team agreement

Below is a proposed format for documenting all tables in the SRS. Do we agree on this level of detail?

```markdown
#### Table: `students`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGINT UNSIGNED | PK, AUTO_INCREMENT | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK → tenants.id, NOT NULL | Tenant scope |
| `registration_number` | VARCHAR(20) | NOT NULL | e.g., `26000001` |
| `first_name` | VARCHAR(100) | NOT NULL | |
| `last_name` | VARCHAR(100) | NOT NULL | |
| `date_of_birth` | DATE | NOT NULL | |
| `gender` | ENUM('male','female','other') | NOT NULL | |
| `photo_url` | VARCHAR(500) | NULLABLE | Cloudinary URL |
| `guardian_name` | VARCHAR(255) | NOT NULL | |
| `guardian_phone` | VARCHAR(20) | NOT NULL | |
| `guardian_email` | VARCHAR(255) | NULLABLE | |
| `address` | TEXT | NOT NULL | |
| `status` | ENUM('active','transferred','dropped_out','graduated') | NOT NULL, DEFAULT 'active' | |
| `deleted_at` | TIMESTAMP | NULLABLE | Soft delete |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

**Indexes:**
- `UNIQUE(tenant_id, registration_number)`
- `INDEX(tenant_id)`
- `INDEX(tenant_id, status)`
```

---

## 4. Authentication & Security

| # | Question | Options / Notes |
|---|---|---|
| 35 | JWT secret management? | Environment variable, AWS Secrets Manager, HashiCorp Vault? |
| 36 | Access token expiry? | Default 15 minutes? Configurable per tenant? |
| 37 | Refresh token expiry? | 7 days? 30 days? Configurable? |
| 38 | Token storage on client? | httpOnly cookie (recommended) or localStorage? |
| 39 | Single session per user? | Enforce or optional? PRD says "optional" — what's the MVP decision? |
| 40 | Password policy? | Minimum length? Complexity requirements? Configurable per tenant (PRD §6.9)? |
| 41 | Session timeout (idle)? | PRD mentions it in Settings — what default? 30 min? 60 min? |
| 42 | Rate limiting — store? | In-memory (Redis) or database? |
| 43 | CORS — which origins allowed? | `*.sirius-skool.com`, `admin.sirius-skool.com`, localhost for dev? |
| 44 | Platform Admin session — separate cookie domain? | Since Platform Admin uses `admin.sirius-skool.com`, cookies won't conflict with tenant subdomains |

---

## 5. Frontend / UI

| # | Question | Options / Notes |
|---|---|---|
| 45 | Separate frontend repo or monorepo? | Separate frontend/backend repos, or monorepo with workspaces? |
| 46 | Build tool? | Vite (recommended for React/Vue/Svelte), Webpack, Turbopack |
| 47 | State management? | Redux Toolkit, Zustand, Pinia (Vue), React Query/TanStack Query for server state |
| 48 | Routing library? | React Router, Vue Router, TanStack Router |
| 49 | Form handling? | React Hook Form + Zod, Formik, Vue Formulate |
| 50 | HTTP client? | Axios, ky, fetch wrapper |
| 51 | Internationalization (i18n) — start now or defer? | PRD mentions future scope, but schema should support it. Start with English only? |
| 52 | Dark mode? | Needed for MVP? |
| 53 | Responsive design? | Desktop-first or mobile-first? |
| 54 | Design system / style guide? | Separate Figma design phase — who owns this? |

---

## 6. File Storage

| # | Question | Options / Notes |
|---|---|---|
| 55 | Cloudinary or alternative? | Cloudinary (PRD recommendation), AWS S3 + CloudFront, ImageKit |
| 56 | Upload flow? | Direct to Cloudinary from frontend (signed upload) or via backend proxy? |
| 57 | Allowed file types? | Images (student photos, ID card templates), PDF (notices, reports), Excel (imports) |
| 58 | File size limits? | Photo: 5MB? Document: 10MB? |
| 59 | Image transformation? | Cloudinary auto-crop/resize for ID card photos, thumbnails |

---

## 7. Testing

| # | Question | Options / Notes |
|---|---|---|
| 60 | Testing framework (backend)? | PHPUnit/Pest (Laravel), Jest/Vitest (Node), pytest (Python) |
| 61 | Testing framework (frontend)? | Vitest + Testing Library, Cypress/Playwright for E2E |
| 62 | Test coverage target for MVP? | 70%? 80%? Critical paths only? |
| 63 | CI/CD pipeline? | GitHub Actions, GitLab CI, Jenkins |
| 64 | API tests? | Integration tests for all endpoints? |
| 65 | Database testing strategy? | In-memory SQLite or test PostgreSQL database? Refresh per test suite? |
| 66 | Who writes tests? | Developers before or after implementation? PR First approach? |

---

## 8. Deployment & Infrastructure

| # | Question | Options / Notes |
|---|---|---|
| 67 | Hosting provider? | AWS, DigitalOcean, Vercel + Railway, Hetzner, self-hosted? |
| 68 | Containerization? | Docker + Docker Compose for development? Kubernetes for production? |
| 69 | Database hosting? | Managed RDS (AWS), DigitalOcean Managed DB, self-hosted? |
| 70 | Redis hosting? | Managed Redis (Upstash, AWS ElastiCache), or sidecar container? |
| 71 | Environment management? | `.env` files, GitHub Secrets, AWS Parameter Store? |
| 72 | Wildcard SSL for `*.sirius-skool.com`? | Let's Encrypt, AWS Certificate Manager, Cloudflare? |
| 73 | Domain DNS setup? | Route53, Cloudflare, Namecheap? |
| 74 | Backup strategy? | Daily automated DB backup. 30-day retention (PRD). Who manages this? |
| 75 | Staging environment? | Separate staging subdomain? Same infrastructure or scaled down? |
| 76 | Monitoring & alerting? | Sentry (error tracking), New Relic/Datadog, Uptime monitoring |

---

## 9. Code Conventions

| # | Question | Options / Notes |
|---|---|---|
| 77 | Linter/Formatter? | ESLint + Prettier (JS/TS), Laravel Pint/PHP CS Fixer (PHP), Ruff/Black (Python) |
| 78 | Git workflow? | Git Flow, GitHub Flow, trunk-based development? |
| 79 | Branch naming? | `feature/auth-module`, `bugfix/login-error`, `chore/update-deps` |
| 80 | Commit convention? | Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:` |
| 81 | Code review process? | PR requires 1 approval? 2 approvals? Who reviews? |
| 82 | Pre-commit hooks? | Husky + lint-staged for linting/tests before commit? |

---

## 10. Project Structure

| # | Question | Options / Notes |
|---|---|---|
| 83 | Monorepo or multi-repo? | Monorepo (Turborepo, Nx) or separate backend/frontend repos? |
| 84 | Backend folder structure? | Modular (by domain) or layered (controllers/services/repos)? |

**Proposed backend structure (example for discussion):**

```
src/
├── Modules/
│   ├── Auth/
│   │   ├── Controllers/
│   │   ├── Requests/    (validation)
│   │   ├── Services/
│   │   └── Models/
│   ├── Student/
│   ├── Attendance/
│   └── ...
├── Shared/
│   ├── Middleware/       (tenant-scoping, auth, permission)
│   ├── Models/           (base models)
│   └── Traits/
├── Config/
└── Database/
    └── Migrations/
```

**Proposed frontend structure (example for discussion):**

```
src/
├── modules/
│   ├── auth/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── api/
│   ├── students/
│   └── ...
├── shared/
│   ├── components/       (reusable UI)
│   ├── hooks/
│   ├── api/              (axios instance, interceptors)
│   └── utils/
├── layouts/
└── router/
```

---

## 11. Error Handling & Logging

| # | Question | Options / Notes |
|---|---|---|
| 85 | Standard error response format? | Proposed in SRS Auth section: `{ error: { code, message, details } }`. Agreed? |
| 86 | HTTP status codes — which ones? | 200, 201, 400, 401, 403, 404, 409, 422, 429, 500 |
| 87 | Logging library? | Monolog (PHP), Winston/Pino (Node.js), structlog (Python) |
| 88 | Log levels? | debug, info, warning, error, critical |
| 89 | Log storage? | Files (rotated), CloudWatch, Papertrail, ELK stack? |

---

## 12. Module Planning

| # | Question | Options / Notes |
|---|---|---|
| 90 | Which module to build first after Auth? | Settings & Academic Year (foundation for everything else) |
| 91 | Should we build the Platform Admin dashboard first? | Needed to create tenants before School Admin can work |
| 92 | Do we need a tenant seeding script for development? | Auto-create a demo tenant with sample data on first run |
| 93 | Should we estimate story points for each module? | For sprint planning |
| 94 | What's the sprint cadence? | 1 week? 2 weeks? |

---

## Summary of Decisions Needed Before SRS v1.0

| Category | Must Decide Before Coding | Can Decide During Development |
|---|---|---|
| Tech Stack | Backend framework, Frontend framework, Database | UI component library details |
| API Design | REST vs GraphQL, versioning, response format | Specific endpoint optimizations |
| DB Design | Primary key type (UUID vs integer), naming conventions | Specific indexes (can add later) |
| Auth | Token expiry times, single session policy, cookie vs localStorage | Advanced security hardening |
| Frontend | Framework, build tool, state management | Specific component implementation |
| Testing | Framework choice | Specific test cases |
| Deployment | Hosting provider, Docker strategy, SSL | Monitoring dashboard details |
| Code Conventions | Git workflow, commit convention, linter | Pre-commit hook details |
| Project Structure | Monorepo vs multi-repo, folder layout | Specific file organization within modules |

---

*End of Questions*
