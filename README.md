# SiriusSchool

Multi-tenant SaaS school management system for small-to-medium schools. Modular, permission-driven platform covering admissions, attendance, results, notices, and reports.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Node.js + TypeScript, Express.js, Prisma ORM |
| Frontend | React + TypeScript, Refine, Ant Design, Vite |
| Database | PostgreSQL 16 |
| Validation | Zod (shared frontend + backend) |
| Auth | JWT (access + refresh tokens with rotation) |
| Monorepo | npm workspaces |

## Project Structure

```
sirius-school/
├── backend/
│   ├── prisma/                    # Schema + migrations
│   ├── src/
│   │   ├── lib/                   # Shared utilities (JWT, SMS, etc.)
│   │   ├── middleware/            # Tenant context, auth, validation, error handler
│   │   └── modules/               # 14 feature modules
│   │       ├── platform/
│   │       ├── auth/
│   │       ├── academic/
│   │       ├── settings/
│   │       ├── user-permission/
│   │       ├── module-management/
│   │       ├── student/
│   │       ├── attendance/
│   │       ├── result/
│   │       ├── notice/
│   │       ├── notification/
│   │       ├── reports/
│   │       ├── dashboard/
│   │       └── audit/
│   └── tests/
├── frontend/
│   └── src/
│       ├── components/
│       ├── pages/                 # Refine pages per module
│       ├── providers/             # Refine data providers
│       └── lib/                   # API client, hooks
├── packages/
│   └── shared/                    # Shared Zod schemas, TS types, constants
│       └── src/
│           ├── schemas/
│           ├── types/
│           └── constants/
└── docs/
    ├── pre-specs/                 # Architecture, conventions, implementation order
    ├── specs/                     # OpenAPI 3.1 specs (80 endpoints, 14 modules)
    ├── PRD/                       # Product Requirements Document
    ├── SRS/                       # Software Requirements Specification
    ├── DB/                        # Data dictionary + Prisma schema (23 models)
    └── traceability/              # FR → OpenAPI → Prisma mapping
```

## Modules

| # | Module | Priority |
|---|--------|----------|
| 1 | Platform & Multi-Tenant | P0 |
| 2 | Authentication | P0 |
| 3 | Academic Structure | P0 |
| 4 | Settings | P0 |
| 5 | User & Permission Management | P0 |
| 6 | Module Management | P0 |
| 7 | Student Management & Admission | P0 |
| 8 | Attendance Management | P0 |
| 9 | Result Management | P1 |
| 10 | Notice Board | P1 |
| 11 | Notification System | P1 |
| 12 | Reports | P1 |
| 13 | Dashboard | P0 |
| 14 | Audit Logging | P0 |

## Getting Started

### Prerequisites

- Node.js 22.x
- PostgreSQL 16
- npm

### Setup

```bash
# Install dependencies
npm install

# Set up environment
cp backend/.env.example backend/.env

# Run database migrations
npx prisma migrate dev --schema backend/prisma/schema.prisma

# Seed initial data
npx prisma db seed

# Start development
npm run dev
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |
| `JWT_SECRET` | Access token signing secret |
| `JWT_REFRESH_SECRET` | Refresh token signing secret |
| `PORT` | Backend server port (default: 4000) |

## Architecture Highlights

- **Multi-tenant:** Shared database with `tenant_id` on every scoped table, enforced by middleware
- **Two-tier permissions:** Platform Admin → School Admin (module-level) → Manager (module-level toggle)
- **API response envelope:** `{ data, meta, error }` on all endpoints
- **API versioning:** `/api/v1/` prefix
- **JWT auth:** Access tokens (15 min) + refresh tokens (7 days) with rotation and reuse detection

## Documentation

| Document | Location |
|----------|----------|
| Product Requirements | `docs/PRD/PRD.md` |
| Software Requirements | `docs/SRS/SRS.md` |
| Architecture Decisions | `docs/pre-specs/ARCHITECTURE.md` |
| Tech Stack & Rationale | `docs/pre-specs/TECH-STACK.md` |
| Coding Conventions | `docs/pre-specs/CODING-CONVENTIONS.md` |
| Implementation Order | `docs/pre-specs/IMPLEMENTATION-ORDER.md` |
| Testing Strategy | `docs/pre-specs/TESTING-STRATEGY.md` |
| OpenAPI Specs | `docs/specs/index.yaml` |
| Database Dictionary | `docs/DB/DB-Dictionary.md` |
| Prisma Schema | `docs/DB/schema.prisma` |

## License

Proprietary — All Rights Reserved.
