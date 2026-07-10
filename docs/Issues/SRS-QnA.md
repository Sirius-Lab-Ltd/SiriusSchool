# SRS — Questions for Developer Team

> **Purpose:** These questions need team input to finalize the SRS and begin implementation.
> **Status:** Updated for v0.2 — covers all MVP modules
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
13. [Module-Specific Questions](#13-module-specific-questions-for-next-srs-iteration)
    - 13.1 Student Management & Admission
    - 13.2 Attendance Management
    - 13.3 Result Management
    - 13.4 Notice Board
    - 13.5 Notification System
    - 13.6 Reports
    - 13.7 Settings
    - 13.8 Dashboard
    - 13.9 Cross-Cutting: User & Permission Management
    - 13.10 Cross-Cutting: Module Management
    - 13.11 Cross-Cutting: Audit Logging
    - 13.12 Multi-Tenant & Data Isolation

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

## 13. Module-Specific Questions (for Next SRS Iteration)

The following questions are specifically needed to write detailed SRS sections for each remaining MVP module.

### 13.1 Student Management & Admission

| # | Question | Options / Notes |
|---|---|---|
| 95 | Admission form — public or authenticated? | PRD says "Student fills admission form" but no parent portal exists in MVP. Is there a public-facing page with no login required? |
| 96 | CAPTCHA / anti-bot for admission form? | If public, should we add CAPTCHA (Google reCAPTCHA, Cloudflare Turnstile, hCaptcha)? |
| 97 | What fields on the admission form? | Full list: student name, DOB, gender, guardian details, address, documents, previous school, etc. Need final field list |
| 98 | Documents upload in admission? | Student photo, birth certificate, previous report card — allowed formats and size limits |
| 99 | Admission form validation rules? | Required vs optional fields, minimum age validation, phone/email format |
| 100 | Temporary admission table TTL mechanism? | Cron job vs database event vs app-level scheduler? Interval? |
| 101 | Permanent registration number generation — atomicity? | Use DB sequence, `SELECT FOR UPDATE`, or application-level lock? |
| 102 | Roll number assignment strategy? | Sequential per section (reset each year), or gaps allowed? Manual override flow? |
| 103 | Bulk promotion workflow — UI flow? | Select from class → preview students → confirm. What about students not promoted (held back)? |
| 104 | Student import from Excel — template format? | Columns, required fields, validation on import, error reporting (which rows failed?) |
| 105 | Student status lifecycle? | `active` → `transferred` / `dropped_out` / `graduated`. Can a student be re-activated? |
| 106 | Transfer Certificate (TC) — data source? | Generated from student data or manually entered fields? PDF template? |
| 107 | Soft delete — restore flow? | Can School Admin restore a soft-deleted student? UI for viewing deleted students? |

### 13.2 Attendance Management

| # | Question | Options / Notes |
|---|---|---|
| 108 | Attendance marking — granularity? | Present/Absent only, or include: Late, Excused, Holiday? |
| 109 | Attendance edit window? | PRD says "same day only" configurable. What's the MVP default? |
| 110 | Month closure — who triggers it? | Manual (School Admin) or automatic (cron at month end)? |
| 111 | Attendance report — percentage calculation? | (Days present / Total days) × 100? What about Sundays/holidays — exclude from total? |
| 112 | SMS for absentees — immediate or batch? | Sent immediately on save, or queued and sent in batches? |
| 113 | Bulk attendance? | Mark attendance for an entire class at once (all present, then mark absentees) |
| 114 | Attendance for non-teaching days? | How to handle weekends, holidays — skip or mark as holiday? |

### 13.3 Result Management

| # | Question | Options / Notes |
|---|---|---|
| 115 | Grading system? | Numeric marks only, grade-based (A/B/C/D/F), or both configurable per exam? |
| 116 | Exam types? | PRD lists Midterm, Final, Quiz. Fixed types or user-defined? |
| 117 | Subject management — per class? | Subjects defined per class or globally? Shared subjects across sections? |
| 118 | Marks entry UI — single student or grid? | Enter per student one-by-one, or grid view (students × subjects) for bulk entry? |
| 119 | Pass/fail criteria? | Minimum percentage per subject? Overall average? Configurable? |
| 120 | Rank calculation — tie handling? | Same marks = same rank, then skip next? (e.g., 1, 2, 2, 4) |
| 121 | Publish/unpublish workflow? | Does publishing trigger SMS/email to all parents? Unpublish requires confirmation? |
| 122 | Result report formats? | Per-student mark sheet, class-wide rank list, subject-wise performance summary |
| 123 | Can marks be entered by subject teachers? | PRD mentions this, but permission model may need subject-level granularity — MVP or post-MVP? |

### 13.4 Notice Board

| # | Question | Options / Notes |
|---|---|---|
| 124 | Notice attachments — file types? | PDF only, or images, Word docs? |
| 125 | Notice visibility scope? | All users (any role) or can target specific roles (e.g., teachers only)? |
| 126 | Scheduled notices — cron or queue? | How is auto-publish triggered? Background job? |
| 127 | Notice categories/tags? | e.g., "Exam", "Holiday", "Meeting" — for filtering |
| 128 | Archive vs delete behavior? | Archive = hide from active view but retain. Delete = soft delete. Can archived notices be restored? |
| 129 | Notice read tracking? | Track who has viewed a notice? (Post-MVP feature for MVP?) |

### 13.5 Notification System

| # | Question | Options / Notes |
|---|---|---|
| 130 | SMS provider? | Twilio, Vonage (Nexmo), AWS SNS, local provider? |
| 131 | Email provider? | SendGrid, SES (AWS), Mailgun, SMTP? |
| 132 | SMS gateway integration — how? | REST API wrapper, third-party SDK? |
| 133 | Email sending — queue? | Always async via queue? Sync for password reset only? |
| 134 | Notification templates — stored where? | Database table (`notification_templates`) or file-based? |
| 135 | Template variables — full list? | `{student_name}`, `{class}`, `{section}`, `{date}`, `{school_name}`, `{percentage}`, `{subject}`, `{exam_name}`, `{guardian_name}`, `{roll_number}` |
| 136 | SMS quota — hard or soft limit? | Hard block at 0 or allow overage with warning? |
| 137 | Low quota warning — threshold? | PRD says < 10%. Configurable? |
| 138 | Notification logs — retention? | 30 days? 90 days? Forever? |
| 139 | Custom SMS/Email — who can send? | School Admin only, or Managers with permission? |

### 13.6 Reports

| # | Question | Options / Notes |
|---|---|---|
| 140 | PDF generation library? | DomPDF (PHP), jsPDF (frontend), Puppeteer, wkhtmltopdf, ReportLab (Python) |
| 141 | Excel export library? | PhpSpreadsheet (PHP), xlsx (Node.js), OpenPyXL (Python) |
| 142 | ID card — layout & dimensions? | Credit card size (85.6 × 54 mm) or A4 sheet with multiple cards? |
| 143 | ID card — what fields? | Student photo, name, registration number, class, section, roll number, school name, logo, academic year |
| 144 | Transfer Certificate (TC) — format? | Fixed government-mandated format or school-specific? |
| 145 | Report generation — sync or async? | Small reports (single student) sync; large reports (bulk) queued? |
| 146 | School branding — where stored? | Logo image, school name, address in Settings. Applied dynamically to all reports |

### 13.7 Settings

| # | Question | Options / Notes |
|---|---|---|
| 147 | Academic Year — date overlap validation? | System prevents overlapping date ranges. What error message? |
| 148 | Closing an Academic Year — prerequisites? | All attendance finalized? All results published? Checklist before close? |
| 149 | Re-opening a closed year — allowed? | PRD mentions it. Any restrictions? (e.g., only if no data entered in new year) |
| 150 | Class → Section → Subject relationship? | Class has many sections. Subjects are assigned to class or section? |
| 151 | Subject — assigned to class or globally? | Global subject pool, then assigned to specific classes? |
| 152 | Password policy defaults? | Min length 8? Require uppercase + lowercase + digit? |
| 153 | Timezone handling? | Store in UTC, display in tenant timezone. Which timezone libraries? |
| 154 | School logo — storage & constraints? | Cloudinary. Max dimensions? Aspect ratio? File size? |

### 13.8 Dashboard

| # | Question | Options / Notes |
|---|---|---|
| 155 | Dashboard metrics — real-time or cached? | PRD says "no caching for MVP" — but frequent DB queries per dashboard load may be slow. Accept for MVP? |
| 156 | Widget layout — fixed or customizable? | Fixed grid per role, or drag-and-drop reorderable? |
| 157 | Manager dashboard — empty state? | What does a Manager with no assigned modules see? |
| 158 | Dashboard refresh interval? | Auto-refresh (every 30s) or manual page reload? |
| 159 | SMS quota widget — real-time or near-real-time? | Display current remaining quota. Refresh on page load or cached? |

### 13.9 Cross-Cutting: User & Permission Management

| # | Question | Options / Notes |
|---|---|---|
| 160 | Permission data model? | Single `permissions` table: (user_id, module, action, is_granted)? Or per-module pivot tables? |
| 161 | School Admin's own permissions — stored or implied? | If a module is assigned by Platform Admin, School Admin gets all actions. Is this stored as explicit rows or checked by code logic? |
| 162 | Manager activation/deactivation — session invalidation? | Uses `token_version` column on `users` — increment on deactivation |
| 163 | Permission cache — strategy? | Cache per user on login, invalidate on permission change. Redis or in-memory? |
| 164 | UI for permission assignment? | Checkbox grid: rows = modules, columns = actions. Simple toggle per cell |

### 13.10 Cross-Cutting: Module Management

| # | Question | Options / Notes |
|---|---|---|
| 165 | Module enable/disable — effect on sidebar? | Disabled module hidden from all users. API endpoints also blocked? |
| 166 | Platform Admin assigns modules — stored where? | `tenant_modules` table: (tenant_id, module_id, is_enabled) |
| 167 | Fixed module list or dynamic? | Hardcoded list in code or database-driven (auto-discoverable)? |
| 168 | Seed modules in database? | Need a seeder that inserts all MVP modules |

### 13.11 Cross-Cutting: Audit Logging

| # | Question | Options / Notes |
|---|---|---|
| 169 | Audit log — what events? | All CUD operations on: students, attendance, results, notices, users, permissions, settings |
| 170 | What data to store per event? | Actor (user_id, role), action (create/update/delete), target (module, record_id), old values, new values, IP, user agent, timestamp |
| 171 | Audit log UI — who can view? | School Admin only? Platform Admin across all tenants? |
| 172 | Audit log retention? | 90 days? 1 year? Forever? |
| 173 | Audit log storage? | Same DB (`audit_logs` table) or separate log storage (ELK, CloudWatch)? |
| 174 | Is audit logging synchronous or async? | Async via queue for performance, except for critical events (user deactivation)? |

### 13.12 Multi-Tenant & Data Isolation

| # | Question | Options / Notes |
|---|---|---|
| 175 | Tenant identification middleware — how implemented? | Middleware that reads subdomain from Host header, queries `tenants` table, sets `tenant_id` on request context |
| 176 | Tenant context — how passed to queries? | Global scope on all models (e.g., Laravel Global Scope, Prisma middleware) |
| 177 | Cross-tenant data leak prevention — testing strategy? | Dedicated test suite that attempts cross-tenant access |
| 178 | `tenants` table structure? | `id`, `name`, `slug` (subdomain), `is_active`, `registration_sequence_start`, `sms_quota`, `sms_used`, `created_at`, `updated_at` |
| 179 | Tenant creation flow? | Platform Admin form: name, slug, assigned modules, SMS quota, starting registration sequence, School Admin name/email/password |

---

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
| Module Priority | Build order (Auth → Settings → Student → ...) | Sprint-level task breakdown |
| Permission Model | Data model (single table vs pivot), caching strategy | UI layout for permission assignment |
| Admission Flow | Public form + CAPTCHA vs authenticated form | Anti-bot measure details |
| Report Generation | PDF library choice | Template design specifics |

---

*End of Questions*
