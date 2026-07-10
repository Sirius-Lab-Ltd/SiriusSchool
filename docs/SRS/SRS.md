# Sirius-Skool — Software Requirements Specification

> **Version:** 0.2  
> **Status:** Draft  
> **Date:** July 10, 2026  
> **Source Documents:** PRD v1.0 (`docs/PRD.md`), DB Dictionary (`docs/DB/DB-Dictionary.md`)

---

## Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | Jul 6 | — | Initial Auth module draft |
| 0.2 | Jul 10 | — | Full rewrite: all MVP modules, API specs, data models, business rules |

---

## Table of Contents

1. [Introduction](#1-introduction)
   - 1.1 Purpose
   - 1.2 Scope
   - 1.3 Definitions & Acronyms
   - 1.4 References
2. [Architecture Overview](#2-architecture-overview)
   - 2.1 Multi-Tenant Model
   - 2.2 User Roles & Hierarchy
   - 2.3 Permission Model (Two-Tier)
   - 2.4 Tech Stack
   - 2.5 Core Entity Relationships
3. [Module Requirements](#3-module-requirements)
   - 3.1 Platform & Multi-Tenant Foundation
   - 3.2 Authentication
   - 3.3 Academic Structure
   - 3.4 Settings
   - 3.5 User & Permission Management
   - 3.6 Module Management
   - 3.7 Student Management & Admission
   - 3.8 Attendance Management
   - 3.9 Result Management
   - 3.10 Notice Board
   - 3.11 Notification System
   - 3.12 Reports
   - 3.13 Dashboard
   - 3.14 Audit Logging
4. [Non-Functional Requirements](#4-non-functional-requirements)
5. [Appendices](#5-appendices)
   - A. Requirements Traceability Matrix
   - B. Error Code Reference
   - C. Open Questions Log

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification (SRS) defines the detailed functional and non-functional requirements for the Sirius-Skool School Management System. It is the single source of truth for developers implementing the MVP.

Each requirement is uniquely ID'd, testable, and traced to a source (PRD § or DB Dictionary table). This document supplements the PRD — where this SRS is silent, the PRD takes precedence.

### 1.2 Scope

**In-scope (MVP):**

| # | Module | Priority | Build Order |
|---|--------|----------|-------------|
| 1 | Platform & Multi-Tenant Foundation | P0 | 1 |
| 2 | Authentication | P0 | 2 |
| 3 | Academic Structure | P0 | 3 |
| 4 | Settings | P0 | 4 |
| 5 | User & Permission Management | P0 | 5 |
| 6 | Module Management | P0 | 6 |
| 7 | Student Management & Admission | P0 | 7 |
| 8 | Attendance Management | P0 | 8 |
| 9 | Result Management | P1 | 9 |
| 10 | Notice Board | P1 | 10 |
| 11 | Notification System | P1 | 11 |
| 12 | Reports | P1 | 12 |
| 13 | Dashboard | P0 | 13 |
| 14 | Audit Logging | P0 | 14 |

**Out-of-scope (post-MVP):** Fee management, parent/student portal, library, transport, hostel, mobile apps, online exams, payroll, LMS integration.

### 1.3 Definitions & Acronyms

| Term | Definition |
|------|------------|
| **Tenant** | A single school/organization with fully isolated data |
| **Platform Admin** | Super Admin managing all tenants (stored in `platform_admins` table) |
| **School Admin** | Primary admin for a single tenant (stored in `users` table, role `SCHOOL_ADMIN`) |
| **Manager** | Staff member with action-level permissions (stored in `users` table, role `MANAGER`) |
| **JWT** | JSON Web Token for API authentication |
| **Access Token** | Short-lived JWT (15 min default) for endpoint authorization |
| **SMS** | Short Message Service for guardian notifications |
| **Registration No** | Permanent student identifier (`YY + sequence`, e.g. `26000001`) |
| **Roll Number** | Per-section, per-year numeric identifier for daily use |
| **PRD** | Product Requirements Document |
| **SRS** | Software Requirements Specification |

### 1.4 References

| Document | Location |
|----------|----------|
| Product Requirements Document | `docs/PRD.md` |
| Database Dictionary | `docs/DB/DB-Dictionary.md` |

---

## 2. Architecture Overview

### 2.1 Multi-Tenant Model

**Strategy:** Shared database with `tenant_id` column on every tenant-scoped table, enforced by application middleware.

```
Platform Admin (admin.sirius-skool.com)
└── Tenant A (school-a.sirius-skool.com)
│   └── Users (School Admin + Managers)
│   └── Students, Attendance, Results, Notices, etc.
└── Tenant B (school-b.sirius-skool.com)
    └── Users, Students, etc.
```

**Key rules:**
- Every tenant-scoped row carries a `tenant_id` FK → `tenants(id)`.
- The `tenant_id` is injected into every query via middleware (never trusts user input).
- `platform_admins` and `tenants` tables are the only ones without a `tenant_id`.
- Cross-tenant data access is prohibited at every layer.

### 2.2 User Roles & Hierarchy

```
Platform (SaaS Owner)
└── Platform Admin (Super Admin)
      └── Tenant (School)
            └── School Admin (1 per tenant)
                  └── Manager 1..n
```

| Role | Scope | Auth Table | Login URL |
|------|-------|------------|-----------|
| Platform Admin | All tenants | `platform_admins` | `admin.sirius-skool.com` |
| School Admin | Single tenant | `users` (role: `SCHOOL_ADMIN`) | `{tenant}.sirius-skool.com` |
| Manager | Single tenant | `users` (role: `MANAGER`) | `{tenant}.sirius-skool.com` |

### 2.3 Permission Model (Two-Tier)

| Tier | Assigner → Assignee | Granularity | Effect |
|------|--------------------|-------------|--------|
| **Tier 1** | Platform Admin → School Admin | Module-level | If a module is assigned, School Admin gets all actions on it |
| **Tier 2** | School Admin → Manager | Action-level | School Admin picks View/Create/Edit/Delete/Print/Export per module |

**School Admin has full access** to all enabled modules — no explicit permission records are stored. **Managers** need explicit records in `manager_permissions` for each module + action.

### 2.4 Tech Stack

#### 2.4.1 Backend

| Component | Choice |
|-----------|--------|
| Runtime | Node.js + TypeScript |
| Framework | Express.js |
| ORM | Prisma (`docs/DB/schema.prisma`) |
| API Versioning | `/api/v1/` |
| Response Envelope | `{ data, meta, error }` |
| Authentication | JWT (access 15 min + refresh 7 days, rotation) |
| Password Hashing | bcrypt |

#### 2.4.2 Frontend

| Component | Choice |
|-----------|--------|
| Framework | React + TypeScript |
| Admin Framework | Refine |
| UI Library | Ant Design (antd) |
| Build Tool | Vite |
| HTTP Client | Axios |
| Form Handling | React Hook Form |
| Validation | Zod (shared frontend + backend) |
| Server State | TanStack Query (via Refine) |

#### 2.4.3 Database & Storage

| Component | Choice |
|-----------|--------|
| Engine | PostgreSQL |
| ORM | Prisma (`docs/DB/schema.prisma`) |
| PK Type | UUID (`@default(uuid()) @db.Uuid`) |
| Naming | `snake_case` (tables via `@@map`, columns in schema) |
| Timestamps | `created_at` + `updated_at` on all tables, `@db.Timestamptz` |
| Audit Storage | `Json` `@db.JsonB` for `old_values` / `new_values` |
| File Storage | Cloudinary |
| Caching | None for MVP |
| Queue | DB-based polling (`notifications` table) |

#### 2.4.4 Infrastructure

| Component | Decision |
|-----------|----------|
| Repository | Monorepo (`/backend`, `/frontend`, `/docs`) |
| PDF Generation | TBD |
| Excel Generation | TBD |
| SMS Provider | TBD |
| Email Provider | TBD |
| Testing | TBD |
| Deployment | TBD |

> **TBD** = To Be Decided — these decisions will be made during development.

### 2.5 Core Entity Relationships

```
Tenant
 ├── Academic Years
 │    ├── Classes → Sections
 │    └── Student Enrollments
 │         ├── Attendance Sessions → Attendance Records
 │         └── Marks (via Exam Subjects)
 ├── Students
 ├── Subjects (per class)
 ├── Exams → Exam Subjects
 ├── Grade Scales
 ├── Applications (admission)
 ├── Notices
 ├── Users (School Admin + Managers)
 │    └── Manager Permissions (→ Tenant Modules)
 ├── Notifications
 ├── Audit Logs
 └── Settings
```

---

## 3. Module Requirements

### 3.1 Platform & Multi-Tenant Foundation

**Quick Summary:** This module provides the core multi-tenant infrastructure. Platform Admin can create, activate, deactivate, and manage tenants (schools). Each tenant gets a unique subdomain, an SMS balance, and a student registration sequence counter. Tenant-level settings and module assignments are managed through this foundation. This is the first module to build — everything else depends on it.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-PLT-01 | Platform Admin shall create a new tenant with name, subdomain, assigned modules, SMS quota, starting registration sequence, and initial School Admin credentials | P0 | Platform Admin is authenticated | POST /api/admin/tenants |
| FR-PLT-02 | System shall validate that subdomain is globally unique | P0 | Tenant creation request | Before create |
| FR-PLT-03 | System shall create School Admin user in `users` table during tenant creation (same transaction) | P0 | Tenant record created | After tenant insert |
| FR-PLT-04 | System shall create a `tenant_settings` record for the new tenant | P0 | Tenant record created | After tenant insert |
| FR-PLT-05 | System shall seed `tenant_modules` records for all assigned modules | P0 | Tenant record created | After tenant insert |
| FR-PLT-06 | Platform Admin shall activate/deactivate a tenant | P0 | Tenant exists | PATCH /api/admin/tenants/{id} |
| FR-PLT-07 | System shall block all user access for a deactivated tenant | P0 | `tenants.is_active = false` | On any auth request |
| FR-PLT-08 | Platform Admin shall view a list of all tenants with status | P0 | Platform Admin is authenticated | GET /api/admin/tenants |
| FR-PLT-09 | Platform Admin shall adjust SMS balance for a tenant | P0 | Tenant exists | PATCH /api/admin/tenants/{id}/sms-balance |
| FR-PLT-10 | Platform Admin shall view notification logs across all tenants | P1 | Platform Admin is authenticated | GET /api/admin/notifications |

#### API Endpoints

##### POST /api/admin/tenants

Create a new tenant with its School Admin.

**Request:**
```json
{
  "name": "Springfield School",
  "subdomain": "springfield",
  "modules": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT"],
  "sms_quota": 500,
  "starting_sequence": 1,
  "school_admin": {
    "full_name": "John Doe",
    "email": "admin@springfield.com",
    "password": "SecurePass123"
  }
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "name": "Springfield School",
  "subdomain": "springfield",
  "is_active": true,
  "sms_balance": 500,
  "school_admin": {
    "id": "uuid",
    "email": "admin@springfield.com",
    "role": "SCHOOL_ADMIN"
  }
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 409 | `SUBDOMAIN_TAKEN` | Subdomain already exists |
| 422 | `VALIDATION_ERROR` | Invalid fields |

##### GET /api/admin/tenants

List all tenants.

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Springfield School",
      "subdomain": "springfield",
      "is_active": true,
      "sms_balance": 420,
      "created_at": "2026-07-01T00:00:00Z"
    }
  ]
}
```

##### PATCH /api/admin/tenants/{id}

Update tenant (activate/deactivate, change name, etc.).

**Request (partial):**
```json
{
  "is_active": false
}
```

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "is_active": false
}
```

##### PATCH /api/admin/tenants/{id}/sms-balance

Adjust SMS balance.

**Request:**
```json
{
  "adjustment": 200
}
```

**Response `200 OK`:**
```json
{
  "sms_balance": 620
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `INSUFFICIENT_BALANCE` | Adjustment would make balance negative |

#### Data Model

**Table: `tenants`** — Full schema in DB Dictionary §Table 2.

| Column | Type | Example |
|--------|------|---------|
| id | UUID | `a1b2c3d4-...` |
| name | VARCHAR(255) | `Springfield School` |
| subdomain | VARCHAR(100) | `springfield` |
| is_active | BOOLEAN | `true` |
| sms_balance | INTEGER | `500` |
| current_student_sequence | INTEGER | `156` |

**Table: `tenant_settings`** — DB Dictionary §Table 3. One-to-one with tenant. Logo, address, phone, email.

**Table: `tenant_modules`** — DB Dictionary §Table 4. Links tenant to enabled modules.

#### Business Rules

| ID | Rule |
|----|------|
| BR-PLT-01 | IF tenant is created THEN School Admin user, settings record, and module assignments are created in the same transaction |
| BR-PLT-02 | IF tenant.is_active = false THEN all login attempts return 403 TENANT_INACTIVE |
| BR-PLT-03 | IF subdomain already exists THEN return 409 SUBDOMAIN_TAKEN |
| BR-PLT-04 | IF SMS balance adjustment would go below 0 THEN return 400 |
| BR-PLT-05 | Platform Admin cannot directly modify tenant operational data (students, attendance, results) |

#### Edge Cases & Error States

| Scenario | Behavior |
|----------|----------|
| Create tenant with existing subdomain | 409 CONFLICT |
| Deactivate active tenant | All active sessions invalidated immediately |
| Reactivate tenant | All users can log in again (sessions not restored) |
| Set SMS balance to 0 | SMS sending blocked, email still works |
| Delete tenant | Not implemented in MVP (soft-deactivate only) |

#### Open Questions

| # | Question |
|---|----------|
| Q-PLT-01 | Should tenant deletion be implemented in MVP, or just deactivation? |
| Q-PLT-02 | What is the default module set for a new tenant? |

---

### 3.2 Authentication

**Quick Summary:** All users authenticate through this module. Platform Admin uses a separate login page (`admin.sirius-skool.com`) and authenticates against the `platform_admins` table. School Admin and Managers log in via their tenant subdomain (`{tenant}.sirius-skool.com`) and authenticate against the `users` table. The system issues JWT access tokens (15 min) and refresh tokens (7 days) with rotation and reuse detection.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-AUTH-01 | System shall authenticate Platform Admin using email + password against `platform_admins` table | P0 | Admin page (`admin.sirius-skool.com`) | POST /api/admin/auth/login |
| FR-AUTH-02 | System shall authenticate School Admin/Manager using email + password against `users` table, scoped by tenant subdomain | P0 | Tenant is active, user is active | POST /api/auth/login |
| FR-AUTH-03 | System shall extract tenant from subdomain in Host header (with X-Tenant-Slug fallback for dev) | P0 | Login page loaded | Before authentication |
| FR-AUTH-04 | System shall issue JWT access token (15 min) + refresh token (UUID, 7 days) on successful login | P0 | Credentials valid | After authentication |
| FR-AUTH-05 | System shall redirect to role-specific dashboard after login | P0 | Login succeeds | After token issuance |
| FR-AUTH-06 | System shall revoke refresh token on logout | P0 | User is authenticated | POST /api/auth/logout |
| FR-AUTH-07 | System shall issue new access token using valid refresh token (rotation + reuse detection) | P0 | Refresh token is valid | POST /api/auth/refresh |
| FR-AUTH-08 | System shall send password reset email with time-limited token (1 hour) | P0 | Email belongs to active account | POST /api/auth/forgot-password |
| FR-AUTH-09 | System shall reset password using valid, non-expired, non-used reset token | P0 | Token valid | POST /api/auth/reset-password |
| FR-AUTH-10 | System shall allow authenticated users to change password (requires current password) | P0 | User is authenticated | POST /api/auth/change-password |
| FR-AUTH-11 | System shall return current user profile + permissions | P0 | Valid access token | GET /api/auth/me |
| FR-AUTH-12 | System shall block login for inactive users | P0 | `users.is_active = false` | Login attempt |
| FR-AUTH-13 | System shall block login for inactive tenants | P0 | `tenants.is_active = false` | Login attempt |
| FR-AUTH-14 | System shall rate-limit login attempts (5 failed per minute per IP) | P1 | Rate exceeded | Login attempt |

#### API Endpoints

##### POST /api/admin/auth/login

Platform Admin login.

**Request:**
```json
{
  "email": "admin@sirius.com",
  "password": "password123"
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "uuid",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": {
    "id": "uuid",
    "email": "admin@sirius.com",
    "role": "platform_admin"
  }
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many attempts |

##### POST /api/auth/login

School Admin / Manager login (tenant-scoped via subdomain).

**Request:**
```json
{
  "email": "admin@school.com",
  "password": "password123"
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "uuid",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": {
    "id": "uuid",
    "email": "admin@school.com",
    "full_name": "John Doe",
    "role": "SCHOOL_ADMIN",
    "tenant_id": "uuid"
  }
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 403 | `ACCOUNT_INACTIVE` | User is deactivated |
| 403 | `TENANT_INACTIVE` | Tenant is deactivated |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many attempts |

##### POST /api/auth/logout

Revoke refresh token.

**Request:**
```json
{
  "refresh_token": "uuid"
}
```

**Response `200 OK`:**
```json
{
  "message": "Logged out successfully"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_REFRESH_TOKEN` | Token not found or already revoked |

##### POST /api/auth/refresh

Rotate refresh token. Issues new access + refresh token.

**Request:**
```json
{
  "refresh_token": "uuid"
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "new-uuid",
  "token_type": "Bearer",
  "expires_in": 900
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_REFRESH_TOKEN` | Not found or revoked |
| 401 | `TOKEN_EXPIRED` | Expired |
| 401 | `TOKEN_REUSE_DETECTED` | Revoked token reused — all tokens for user revoked |

##### POST /api/auth/forgot-password

Send reset link (always returns 200 to prevent email enumeration).

**Request:**
```json
{
  "email": "admin@school.com"
}
```

**Response `200 OK`:**
```json
{
  "message": "If the email exists, a reset link has been sent"
}
```

##### POST /api/auth/reset-password

Reset password with token.

**Request:**
```json
{
  "token": "reset-token",
  "password": "NewPass123"
}
```

**Response `200 OK`:**
```json
{
  "message": "Password reset successfully"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `INVALID_TOKEN` | Token is invalid |
| 400 | `TOKEN_EXPIRED` | Token expired |
| 400 | `TOKEN_ALREADY_USED` | Token already used |

##### POST /api/auth/change-password

Change password (authenticated).

**Request:**
```json
{
  "current_password": "OldPass123",
  "new_password": "NewPass456"
}
```

**Response `200 OK`:**
```json
{
  "message": "Password changed successfully"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `INCORRECT_CURRENT_PASSWORD` | Current password is wrong |
| 400 | `SAME_PASSWORD` | New password same as current |

##### GET /api/auth/me

Current user profile + permissions.

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "email": "admin@school.com",
  "full_name": "John Doe",
  "role": "SCHOOL_ADMIN",
  "tenant": {
    "id": "uuid",
    "name": "Springfield School",
    "slug": "springfield",
    "is_active": true
  },
  "permissions": {
    "STUDENT_MANAGEMENT": ["view", "create", "edit", "delete", "export"],
    "ATTENDANCE_MANAGEMENT": ["view", "take", "edit"]
  }
}
```

#### Data Model

**Table: `platform_admins`** — DB Dictionary §Table 1.

| Column | Type | Example |
|--------|------|---------|
| id | UUID | `uuid` |
| email | VARCHAR(255) | `admin@sirius.com` |
| password_hash | TEXT | `$2y$...` |

**Table: `users`** — DB Dictionary §Table 5.

| Column | Type | Example |
|--------|------|---------|
| id | UUID | `uuid` |
| tenant_id | UUID | `uuid` |
| full_name | VARCHAR(255) | `John Doe` |
| email | VARCHAR(255) | `admin@school.com` |
| password_hash | TEXT | `$2y$...` |
| role | USER_ROLE | `SCHOOL_ADMIN` |
| is_active | BOOLEAN | `true` |
| last_login_at | TIMESTAMPTZ | `2026-07-10T10:00:00Z` |

> **Note:** The Prisma schema (`docs/DB/schema.prisma`) does not currently define `refresh_tokens` or `password_resets` models. The developer must either add them to `schema.prisma` or implement token validation at the application level (e.g., signed JWTs for refresh tokens with no DB lookup).

#### Business Rules

| ID | Rule |
|----|------|
| BR-AUTH-01 | IF user logs in via `admin.sirius-skool.com` THEN authenticate against `platform_admins` |
| BR-AUTH-02 | IF user logs in via `{tenant}.sirius-skool.com` THEN authenticate against `users` scoped to that tenant |
| BR-AUTH-03 | IF user.is_active = false THEN return 403 ACCOUNT_INACTIVE |
| BR-AUTH-04 | IF tenant.is_active = false THEN return 403 TENANT_INACTIVE |
| BR-AUTH-05 | IF reset token is reused THEN it is invalidated after first use |
| BR-AUTH-06 | IF revoked refresh token is presented THEN revoke ALL refresh tokens for that user (reuse detection) |
| BR-AUTH-07 | IF login fails >5 times in 1 minute from same IP THEN rate-limit (429) |

#### Error Response Format (Standard)

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable description",
    "details": {}
  }
}
```

#### Open Questions

| # | Question |
|---|----------|
| Q-AUTH-01 | Access token expiry — 15 minutes or configurable? |
| Q-AUTH-02 | Refresh token storage — add `refresh_tokens` model to Prisma, or use signed JWTs without DB lookup? |
| Q-AUTH-03 | Single session per user — enforced in MVP or optional? |

---

### 3.3 Academic Structure

**Quick Summary:** This module defines the academic hierarchy that every other module depends on. School Admin creates academic years (e.g., `2026-2027`), classes (e.g., `Class 6`), sections (e.g., `Section A`), and subjects (e.g., `Mathematics` per class). These entities are permanent and reused across academic years. Student enrollments link students to this structure per year.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-ACA-01 | School Admin shall create academic years with name, start date, end date | P0 | Authenticated as School Admin | POST /api/academic-years |
| FR-ACA-02 | System shall enforce exactly one active academic year per tenant | P0 | Academic years exist | Setting `is_current = true` |
| FR-ACA-03 | System shall prevent overlapping academic year date ranges | P0 | Creating or updating | Validation |
| FR-ACA-04 | School Admin shall create classes with code, name, display_order | P0 | Authenticated | POST /api/classes |
| FR-ACA-05 | School Admin shall create sections within a class | P0 | Class exists | POST /api/classes/{id}/sections |
| FR-ACA-06 | School Admin shall create subjects per class | P0 | Class exists | POST /api/classes/{id}/subjects |
| FR-ACA-07 | School Admin shall set one academic year as current | P0 | Academic year exists | PATCH /api/academic-years/{id} |
| FR-ACA-08 | System shall provide list endpoints for all academic entities (dropdowns) | P0 | Entity exists | GET endpoints |

#### API Endpoints

##### POST /api/academic-years

**Request:**
```json
{
  "name": "2026-2027",
  "start_date": "2026-01-01",
  "end_date": "2026-12-31",
  "is_current": true
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "name": "2026-2027",
  "start_date": "2026-01-01",
  "end_date": "2026-12-31",
  "is_current": true
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 409 | `DUPLICATE_NAME` | Name already exists for this tenant |
| 409 | `OVERLAPPING_DATES` | Date range overlaps existing academic year |
| 400 | `CURRENT_YEAR_EXISTS` | Another year is already current |

##### GET /api/academic-years

List academic years for the tenant.

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "2025-2026",
      "is_current": false
    },
    {
      "id": "uuid",
      "name": "2026-2027",
      "is_current": true
    }
  ]
}
```

##### POST /api/classes

**Request:**
```json
{
  "code": "6",
  "name": "Class 6",
  "display_order": 7
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "code": "6",
  "name": "Class 6",
  "display_order": 7,
  "is_active": true
}
```

##### POST /api/classes/{classId}/sections

**Request:**
```json
{
  "code": "A",
  "name": "Section A",
  "display_order": 1
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "class_id": "uuid",
  "code": "A",
  "name": "Section A"
}
```

##### POST /api/classes/{classId}/subjects

**Request:**
```json
{
  "code": "MATH",
  "name": "Mathematics",
  "subject_type": "MANDATORY",
  "display_order": 1
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "class_id": "uuid",
  "code": "MATH",
  "name": "Mathematics",
  "subject_type": "MANDATORY"
}
```

#### Data Model

**Table: `academic_years`** — DB Dictionary §Table 7.

| Column | Example |
|--------|---------|
| name | `2026-2027` |
| start_date | `2026-01-01` |
| end_date | `2026-12-31` |
| is_current | `true` |

**Table: `classes`** — DB Dictionary §Table 8.

| Column | Example |
|--------|---------|
| code | `6` |
| name | `Class 6` |
| display_order | `7` |

**Table: `sections`** — DB Dictionary §Table 9.

| Column | Example |
|--------|---------|
| class_id | `uuid` |
| code | `A` |
| name | `Section A` |

**Table: `subjects`** — DB Dictionary §Table 10.

| Column | Example |
|--------|---------|
| class_id | `uuid` |
| code | `MATH` |
| name | `Mathematics` |
| subject_type | `MANDATORY` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-ACA-01 | A tenant can have only one current academic year at a time |
| BR-ACA-02 | Academic year names must be unique per tenant |
| BR-ACA-03 | Academic year date ranges must not overlap |
| BR-ACA-04 | Classes, sections, and subjects use `is_active` to disable (never delete) |
| BR-ACA-05 | A section belongs to exactly one class |
| BR-ACA-06 | A subject belongs to exactly one class |
| BR-ACA-07 | Subject codes must be unique within a class |

#### Open Questions

| # | Question |
|---|----------|
| Q-ACA-01 | Do we need a `close` endpoint for academic years, or just `is_current` toggle? |
| Q-ACA-02 | Should closing an academic year verify no pending attendance/results? |

---

### 3.4 Settings

**Quick Summary:** School Admin configures their school's branding (logo, address, phone), academic preferences, and general settings. This is a CRUD module — a single `tenant_settings` record per tenant holds all configurable values. No APIs beyond basic GET/PATCH are needed.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-SET-01 | School Admin shall view their tenant settings | P0 | Authenticated as School Admin | GET /api/settings |
| FR-SET-02 | School Admin shall update school branding (logo, address, phone, email) | P0 | Authenticated | PATCH /api/settings |
| FR-SET-03 | System shall upload logo to Cloudinary and store URL | P0 | File uploaded | During settings update |

#### API Endpoints

##### GET /api/settings

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "logo_url": "https://res.cloudinary.com/.../logo.png",
  "address": "123 School Street",
  "phone": "+8801712345678",
  "email": "info@springfield.edu"
}
```

##### PATCH /api/settings

**Request:**
```json
{
  "address": "456 New Street",
  "phone": "+8801711111111"
}
```

**Response `200 OK`:**
```json
{
  "address": "456 New Street",
  "phone": "+8801711111111"
}
```

#### Data Model

**Table: `tenant_settings`** — DB Dictionary §Table 3. Single row per tenant.

#### Business Rules

| ID | Rule |
|----|------|
| BR-SET-01 | Only School Admin can modify settings |
| BR-SET-02 | Every tenant has exactly one settings record |

#### Open Questions

| # | Question |
|---|----------|
| Q-SET-01 | Where are notification templates stored? DB Dictionary doesn't define a `notification_templates` table yet |
| Q-SET-02 | Should password policy settings (min length, complexity) be stored in settings or stay hardcoded? |

---

### 3.5 User & Permission Management

**Quick Summary:** School Admin creates Manager accounts and assigns action-level permissions per module. Managers can only be assigned modules that the School Admin has (from Platform Admin allocation). School Admin has implicit full access to all assigned modules. This module implements the Tier 2 permission model.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-UP-01 | School Admin shall create a Manager with name, email, password | P0 | Authenticated as School Admin | POST /api/managers |
| FR-UP-02 | School Admin shall activate/deactivate a Manager | P0 | Manager exists | PATCH /api/managers/{id} |
| FR-UP-03 | School Admin shall view list of all Managers with permission summary | P0 | Authenticated | GET /api/managers |
| FR-UP-04 | School Admin shall assign action-level permissions to a Manager per module | P0 | Manager exists, module is enabled | PUT /api/managers/{id}/permissions |
| FR-UP-05 | School Admin shall only see modules assigned by Platform Admin in the permission UI | P0 | Authenticated | GET /api/permissions/available-modules |
| FR-UP-06 | System shall deactivate a Manager immediately — all sessions invalidated | P0 | Manager deactivated | Token version incremented |

#### API Endpoints

##### POST /api/managers

**Request:**
```json
{
  "full_name": "Jane Smith",
  "email": "jane@school.com",
  "password": "SecurePass123",
  "phone": "+8801712345678"
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "full_name": "Jane Smith",
  "email": "jane@school.com",
  "role": "MANAGER",
  "is_active": true
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 409 | `EMAIL_EXISTS` | Email already in use |

##### GET /api/managers

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "full_name": "Jane Smith",
      "email": "jane@school.com",
      "is_active": true,
      "permissions": {
        "STUDENT_MANAGEMENT": ["view", "create"]
      },
      "last_login_at": "2026-07-09T08:00:00Z"
    }
  ]
}
```

##### PATCH /api/managers/{id}/activate

**Request:**
```json
{
  "is_active": false
}
```

##### PUT /api/managers/{id}/permissions

**Request:**
```json
{
  "permissions": [
    {
      "module": "STUDENT_MANAGEMENT",
      "can_view": true,
      "can_create": true,
      "can_edit": false,
      "can_delete": false,
      "can_print": false,
      "can_export": false
    },
    {
      "module": "ATTENDANCE_MANAGEMENT",
      "can_view": true,
      "can_create": true,
      "can_edit": true,
      "can_delete": false,
      "can_print": false,
      "can_export": false
    }
  ]
}
```

**Response `200 OK`:**
```json
{
  "user_id": "uuid",
  "permissions": [
    { "module": "STUDENT_MANAGEMENT", "can_view": true, "can_create": true }
  ]
}
```

#### Data Model

**Table: `manager_permissions`** — DB Dictionary §Table 6. Stores per-module boolean permission flags.

| Column | Example |
|--------|---------|
| user_id | `uuid` |
| tenant_module_id | `uuid` |
| can_view | `true` |
| can_create | `true` |
| can_edit | `false` |
| can_delete | `false` |
| can_print | `false` |
| can_export | `false` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-UP-01 | Only users with role `MANAGER` can have permission records |
| BR-UP-02 | School Admin automatically has full access to all enabled modules |
| BR-UP-03 | School Admin can only delegate modules that are in `tenant_modules` for their tenant |
| BR-UP-04 | IF a module is disabled in `tenant_modules` THEN all related permissions become inactive |
| BR-UP-05 | IF a permission flag is not explicitly set to true THEN it is denied |
| BR-UP-06 | Deactivating a Manager increments `token_version` to invalidate all sessions |

#### Open Questions

| # | Question |
|---|----------|
| Q-UP-01 | Should School Admin be able to edit a Manager's email and password? |
| Q-UP-02 | Is `can_print` separate from `can_export` in the UI, or are they combined? |

---

### 3.6 Module Management

**Quick Summary:** School Admin can enable or disable modules from the pool assigned by Platform Admin. Disabled modules are hidden from the sidebar and inaccessible to all users in the tenant. Data is preserved when a module is disabled.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-MM-01 | School Admin shall view all assigned modules with enable/disable status | P0 | Authenticated | GET /api/tenant-modules |
| FR-MM-02 | School Admin shall toggle a module on/off | P0 | Module is assigned by Platform Admin | PATCH /api/tenant-modules/{module} |
| FR-MM-03 | System shall hide disabled modules from sidebar and block API access | P0 | Module toggled off | On any request to that module |

#### API Endpoints

##### GET /api/tenant-modules

**Response `200 OK`:**
```json
{
  "data": [
    { "module": "STUDENT_MANAGEMENT", "is_enabled": true },
    { "module": "ATTENDANCE_MANAGEMENT", "is_enabled": true },
    { "module": "RESULT_MANAGEMENT", "is_enabled": false }
  ]
}
```

##### PATCH /api/tenant-modules/{module}

**Request:**
```json
{
  "is_enabled": false
}
```

#### Data Model

**Table: `tenant_modules`** — DB Dictionary §Table 4.

| Column | Example |
|--------|---------|
| tenant_id | `uuid` |
| module | `STUDENT_MANAGEMENT` |
| is_enabled | `true` |

**Module ENUM:** `STUDENT_MANAGEMENT`, `ATTENDANCE_MANAGEMENT`, `RESULT_MANAGEMENT`, `NOTICE_BOARD`, `REPORTS`, `NOTIFICATIONS`

#### Business Rules

| ID | Rule |
|----|------|
| BR-MM-01 | Authentication and Settings modules cannot be disabled |
| BR-MM-02 | School Admin cannot enable a module not assigned by Platform Admin |
| BR-MM-03 | Disabling a module hides it for the entire tenant (all roles) |
| BR-MM-04 | Re-enabling restores access to existing data |

---

### 3.7 Student Management & Admission

**Quick Summary:** Covers the complete student lifecycle: admission (public form → review → accept/reject), student CRUD, enrollment management, promotion, and bulk import/export. Registration numbers follow `YY + sequence` format. Roll numbers are per-section, per-year. Students have a permanent identity (`students` table) separate from their yearly enrollment (`student_enrollments`). Admission applications (`applications`) go through a PENDING → APPROVED/REJECTED workflow.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-STU-01 | System shall accept admission applications with applicant details | P0 | Tenant is active | POST /api/applications |
| FR-STU-02 | System shall generate a unique application number (`APP-{year}-{seq}`) | P0 | Application submitted | After create |
| FR-STU-03 | Manager with admission permission shall view pending applications | P0 | Authenticated | GET /api/applications?status=PENDING |
| FR-STU-04 | Manager shall approve an application → creates student + enrollment | P0 | Application is PENDING | POST /api/applications/{id}/approve |
| FR-STU-05 | Manager shall reject an application with reason | P0 | Application is PENDING | POST /api/applications/{id}/reject |
| FR-STU-06 | System shall auto-generate registration number (`YY + sequence`) on approval in an atomic transaction | P0 | Application approved | During approval |
| FR-STU-07 | System shall auto-assign roll number within section (sequential) | P0 | Enrollment created | During approval |
| FR-STU-08 | School Admin/Manager shall CRUD student personal info | P0 | Student exists | /api/students endpoints |
| FR-STU-09 | System shall search students by name, roll number, registration number, class | P0 | Students exist | GET /api/students?search=... |
| FR-STU-10 | School Admin shall promote students to next class (bulk or individual) | P0 | Academic year exists, previous enrollment exists | POST /api/students/promote |
| FR-STU-11 | System shall handle end-of-year outcomes: Promote, Repeat, Graduate, Transfer, Dropout | P0 | Academic year ending | POST /api/students/{id}/outcome |
| FR-STU-12 | School Admin shall import students from Excel | P1 | Authenticated | POST /api/students/import |
| FR-STU-13 | School Admin shall export student list to Excel/PDF | P1 | Authenticated | GET /api/students/export |

#### API Endpoints

##### POST /api/applications

Submit admission application (public — no auth required, or optionally protected).

**Request:**
```json
{
  "academic_year_id": "uuid",
  "applying_class_id": "uuid",
  "full_name": "Alice Johnson",
  "gender": "female",
  "date_of_birth": "2014-05-15",
  "guardian_name": "Bob Johnson",
  "guardian_phone": "+8801712345678",
  "address": "123 Main Street"
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "application_no": "APP-2026-000001",
  "status": "PENDING"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 422 | `VALIDATION_ERROR` | Invalid fields |

##### GET /api/applications

List applications (filterable by status).

**Query params:** `?status=PENDING&page=1&per_page=20`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "application_no": "APP-2026-000001",
      "full_name": "Alice Johnson",
      "status": "PENDING",
      "created_at": "2026-07-01T00:00:00Z"
    }
  ],
  "meta": { "total": 1, "page": 1 }
}
```

##### POST /api/applications/{id}/approve

**Request:**
```json
{
  "section_id": "uuid"
}
```

**Response `200 OK`:**
```json
{
  "student_id": "uuid",
  "registration_no": "26000001",
  "enrollment_id": "uuid",
  "roll_number": 1
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 409 | `ALREADY_REVIEWED` | Application already approved/rejected |

##### POST /api/applications/{id}/reject

**Request:**
```json
{
  "rejection_reason": "Incomplete documents"
}
```

**Response `200 OK`:**
```json
{
  "status": "REJECTED"
}
```

##### CRUD /api/students

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/students | List/search students |
| GET | /api/students/{id} | Get student with enrollments |
| PATCH | /api/students/{id} | Update personal info |
| POST | /api/students | Create student directly (skip application) |

**GET /api/students?search=alice&class_id=uuid&academic_year_id=uuid**

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "registration_no": "26000001",
      "full_name": "Alice Johnson",
      "gender": "female",
      "status": "ACTIVE",
      "current_enrollment": {
        "academic_year": "2026-2027",
        "class": "Class 6",
        "section": "A",
        "roll_number": 1
      }
    }
  ]
}
```

##### POST /api/students/promote

Bulk promote students to next class.

**Request:**
```json
{
  "academic_year_id": "uuid",
  "from_class_id": "uuid",
  "to_class_id": "uuid",
  "section_id": "uuid",
  "student_ids": ["uuid1", "uuid2"],
  "outcome": "PROMOTED"
}
```

**Response `200 OK`:**
```json
{
  "promoted": 2,
  "enrollments": [
    { "student_id": "uuid", "roll_number": 1 },
    { "student_id": "uuid", "roll_number": 2 }
  ]
}
```

#### Data Model

**Table: `applications`** — DB Dictionary §Table 12.

| Column | Example |
|--------|---------|
| application_no | `APP-2026-000001` |
| full_name | `Alice Johnson` |
| status | `PENDING` |

**Table: `students`** — DB Dictionary §Table 13.

| Column | Example |
|--------|---------|
| registration_no | `26000001` |
| full_name | `Alice Johnson` |
| status | `ACTIVE` |

**Table: `student_enrollments`** — DB Dictionary §Table 14.

| Column | Example |
|--------|---------|
| student_id | `uuid` |
| academic_year_id | `uuid` |
| class_id | `uuid` |
| section_id | `uuid` |
| roll_number | `1` |
| status | `ACTIVE` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-STU-01 | Registration number format: `YY` (admission year) + continuous sequence per tenant |
| BR-STU-02 | Registration number sequence is generated atomically using `SELECT ... FOR UPDATE` |
| BR-STU-03 | Registration number is permanent and never changes |
| BR-STU-04 | Roll numbers are unique per (section + academic year) |
| BR-STU-05 | A student can have only one enrollment per academic year |
| BR-STU-06 | Approving an application creates student + enrollment in a single transaction |
| BR-STU-07 | Student statuses: `ACTIVE`, `DROPPED`, `TRANSFERRED`, `GRADUATED` (see Prisma `StudentStatus` enum) |
| BR-STU-08 | Enrollment statuses: `ACTIVE`, `PROMOTED`, `TRANSFERRED`, `GRADUATED` |

#### Edge Cases & Error States

| Scenario | Behavior |
|----------|----------|
| Approve already-approved application | 409 ALREADY_REVIEWED |
| Reject already-rejected application | 409 ALREADY_REVIEWED |
| Approve with invalid section_id | 422 section does not belong to class |
| Bulk promote with mixed outcomes | Transaction fails — all or nothing |
| Registration sequence race condition | Handled by `SELECT ... FOR UPDATE` in transaction |

#### Open Questions

| # | Question |
|---|----------|
| Q-STU-01 | Is admission form public (no auth) or requires registration? |
| Q-STU-02 | What is the application number format? `APP-{year}-{seq}` or simpler? |
| Q-STU-03 | Should we use a CAPTCHA on public admission form? |
| Q-STU-04 | Can a student be re-activated after being set to TRANSFERRED or GRADUATED? |

---

### 3.8 Attendance Management

**Quick Summary:** Manager selects class → section → date, sees all enrolled students (default Present), unchecks absentees, and saves. Attendance has a workflow (DRAFT → SUBMITTED) and supports corrections with audit logging. Absentee SMS can be sent after attendance is saved.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-ATT-01 | Manager shall select class → section → date and load enrolled students | P0 | Academic year active, enrollment exists | GET /api/attendance/sessions/init?class_id=...&section_id=...&date=... |
| FR-ATT-02 | System shall default all students to Present | P0 | Session initialized | Before save |
| FR-ATT-03 | Manager shall save attendance (all students marked) | P0 | Session in DRAFT | POST /api/attendance/sessions |
| FR-ATT-04 | Manager shall submit attendance (DRAFT → SUBMITTED) | P0 | Session saved | POST /api/attendance/sessions/{id}/submit |
| FR-ATT-05 | Manager shall edit attendance after submission (audit logged) | P1 | Session exists | PATCH /api/attendance/sessions/{id}/records |
| FR-ATT-06 | Manager shall send SMS to guardians of absent students | P1 | Session SUBMITTED, students absent | POST /api/attendance/sessions/{id}/send-sms |
| FR-ATT-07 | School Admin shall view attendance reports (per class, per student, date range) | P0 | Attendance exists | GET /api/attendance/reports |

#### API Endpoints

##### POST /api/attendance/sessions

Create (or update) an attendance session.

**Request:**
```json
{
  "academic_year_id": "uuid",
  "class_id": "uuid",
  "section_id": "uuid",
  "attendance_date": "2026-07-10",
  "records": [
    { "student_enrollment_id": "uuid", "attendance_status": "PRESENT" },
    { "student_enrollment_id": "uuid", "attendance_status": "ABSENT", "remarks": "Sick" }
  ]
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "status": "DRAFT",
  "total_present": 49,
  "total_absent": 1
}
```

**ATTENDANCE_STATUS ENUM:** `PRESENT`, `ABSENT`, `LATE`, `LEAVE`

##### POST /api/attendance/sessions/{id}/submit

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "status": "SUBMITTED"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 409 | `ALREADY_SUBMITTED` | Session already submitted |

##### POST /api/attendance/sessions/{id}/send-sms

**Response `200 OK`:**
```json
{
  "sent": 1,
  "failed": 0,
  "sms_sent_at": "2026-07-10T10:30:00Z"
}
```

#### Data Model

**Table: `attendance_sessions`** — DB Dictionary §Table 15.

| Column | Example |
|--------|---------|
| academic_year_id | `uuid` |
| class_id | `uuid` |
| section_id | `uuid` |
| attendance_date | `2026-07-10` |
| status | `DRAFT` or `SUBMITTED` |

**Table: `attendance_records`** — DB Dictionary §Table 16.

| Column | Example |
|--------|---------|
| attendance_session_id | `uuid` |
| student_enrollment_id | `uuid` |
| attendance_status | `ABSENT` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-ATT-01 | A section can have only one attendance session per date per academic year |
| BR-ATT-02 | Attendance defaults to `PRESENT` — only absentees are explicitly marked |
| BR-ATT-03 | Attendance corrections after submission are logged in `audit_logs` |
| BR-ATT-04 | Each SMS sent consumes 1 credit from `tenants.sms_balance` |
| BR-ATT-05 | Only today's and future dates can have attendance taken (configurable backdate limit) |

#### Open Questions

| # | Question |
|---|----------|
| Q-ATT-01 | Should attendance status include more than PRESENT/ABSENT/LATE/LEAVE? |
| Q-ATT-02 | Is "month closure" (making attendance read-only) in MVP scope? |
| Q-ATT-03 | Backdate limit — 7 days configurable? Or fixed? |

---

### 3.9 Result Management

**Quick Summary:** School Admin creates exams for a class, adds subjects with full/pass marks, enters student marks, and publishes results. Grade scales define letter grade + GPA mapping. Results trigger SMS/Email notifications on publish. Publishing is School Admin-only.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-RES-01 | School Admin shall create exams with name, start/end date, class | P1 | Academic year exists, class exists | POST /api/exams |
| FR-RES-02 | School Admin shall configure subjects for an exam (full marks, pass marks, display order) | P1 | Exam in DRAFT | POST /api/exams/{id}/subjects |
| FR-RES-03 | School Admin shall create grade scales for the tenant | P1 | Authenticated | POST /api/grade-scales |
| FR-RES-04 | Manager shall enter marks per student per subject | P1 | Exam is PUBLISHED | PUT /api/exams/{id}/marks |
| FR-RES-05 | System shall auto-calculate total, percentage, grade, GPA | P1 | Marks entered | During calculation/display |
| FR-RES-06 | School Admin shall publish exam results | P1 | All marks entered | POST /api/exams/{id}/publish |
| FR-RES-07 | School Admin shall unpublish results for editing | P1 | Results are published | POST /api/exams/{id}/unpublish |
| FR-RES-08 | School Admin shall generate rank list | P1 | Results published | GET /api/exams/{id}/rank-list |

#### API Endpoints

##### POST /api/exams

**Request:**
```json
{
  "academic_year_id": "uuid",
  "class_id": "uuid",
  "name": "Mid-Term Examination",
  "start_date": "2026-07-15",
  "end_date": "2026-07-20"
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "name": "Mid-Term Examination",
  "status": "DRAFT"
}
```

##### POST /api/exams/{id}/subjects

**Request:**
```json
{
  "subjects": [
    {
      "subject_id": "uuid",
      "full_marks": 100,
      "pass_marks": 33,
      "display_order": 1
    }
  ]
}
```

##### PUT /api/exams/{id}/marks

**Request:**
```json
{
  "marks": [
    {
      "student_enrollment_id": "uuid",
      "exam_subject_id": "uuid",
      "obtained_marks": 85,
      "is_absent": false
    }
  ]
}
```

##### POST /api/exams/{id}/publish

Sets `exams.result_published_at` and triggers SMS/Email notifications.

**Response `200 OK`:**
```json
{
  "result_published_at": "2026-07-25T10:00:00Z"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `MARKS_INCOMPLETE` | Not all students have marks |
| 403 | `FORBIDDEN` | Only School Admin can publish |

#### Data Model

**Table: `exams`** — DB Dictionary §Table 17.

| Column | Example |
|--------|---------|
| name | `Mid-Term Examination` |
| status | `DRAFT` / `PUBLISHED` / `COMPLETED` |
| result_published_at | `2026-07-25T10:00:00Z` |

**Table: `exam_subjects`** — DB Dictionary §Table 18.

| Column | Example |
|--------|---------|
| exam_id | `uuid` |
| subject_id | `uuid` |
| full_marks | `100` |
| pass_marks | `33` |

**Table: `marks`** — DB Dictionary §Table 19.

| Column | Example |
|--------|---------|
| exam_subject_id | `uuid` |
| student_enrollment_id | `uuid` |
| obtained_marks | `85.00` |
| is_absent | `false` |

**Table: `grade_scales`** — DB Dictionary §Table 20.

| Column | Example |
|--------|---------|
| min_marks | `80` |
| max_marks | `100` |
| grade_letter | `A+` |
| grade_point | `5.00` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-RES-01 | A class cannot have two exams with the same name in the same academic year |
| BR-RES-02 | Marks can only be entered for the current academic year |
| BR-RES-03 | Publish is exclusive to School Admin role |
| BR-RES-04 | IF marks are published THEN editing requires unpublish first |
| BR-RES-05 | Grades/GPA are calculated at runtime — never stored in `marks` table |
| BR-RES-06 | IF `is_absent = true` THEN `obtained_marks` must be null |

#### Open Questions

| # | Question |
|---|----------|
| Q-RES-01 | Should marks be enterable in a grid view (students × subjects) or one-by-one? |
| Q-RES-02 | Can multiple managers enter marks for different subjects in the same exam? |
| Q-RES-03 | Grade scale — is one per tenant enough, or per academic year? |

---

### 3.10 Notice Board

**Quick Summary:** School Admin or authorized Manager uploads PDF/Image notices. Notices can be scheduled (future publish_at), expire automatically, and are visible to all authenticated users within the tenant. Published notices cannot be edited — must archive and recreate.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-NTC-01 | Authorized user shall upload a notice (PDF or Image) with title | P1 | Authenticated | POST /api/notices |
| FR-NTC-02 | System shall schedule notice for future publication | P1 | Notice created with future `publish_at` | During create |
| FR-NTC-03 | System shall auto-publish scheduled notices when `publish_at` is reached | P1 | Scheduled notice exists | Cron/queue job |
| FR-NTC-04 | Authorized user shall archive a published notice | P1 | Notice is published | POST /api/notices/{id}/archive |
| FR-NTC-05 | All authenticated users shall view published notices | P1 | Notice is published (`is_published = true`, `publish_at` <= now) | GET /api/notices |
| FR-NTC-06 | System shall auto-hide expired notices based on `expires_at` | P1 | `expires_at` is reached | Cron/queue job |

#### API Endpoints

##### POST /api/notices

**Request (multipart/form-data):**
```
title: "Final Exam Routine"
file: (pdf or image upload)
publish_at: "2026-12-01T08:00:00Z"  (optional, defaults to now)
expires_at: "2026-12-20T00:00:00Z" (optional)
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "title": "Final Exam Routine",
  "file_url": "https://res.cloudinary.com/.../notice.pdf",
  "file_type": "PDF",
  "is_published": true,
  "publish_at": "2026-12-01T08:00:00Z"
}
```

##### GET /api/notices

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Final Exam Routine",
      "file_url": "...",
      "file_type": "PDF",
      "published_at": "2026-12-01T08:00:00Z"
    }
  ]
}
```

#### Data Model

**Table: `notices`** — DB Dictionary §Table 21.

| Column | Example |
|--------|---------|
| title | `Final Exam Routine` |
| file_url | `https://.../notice.pdf` |
| file_type | `PDF` |
| publish_at | `2026-12-01T08:00:00Z` |
| expires_at | `2026-12-20T00:00:00Z` |
| is_published | `true` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-NTC-01 | Published notices cannot be edited (must archive and recreate) |
| BR-NTC-02 | Only PDF and IMAGE file types are supported |
| BR-NTC-03 | All authenticated users in the tenant can view published notices |
| BR-NTC-04 | IF `expires_at` is set and the date has passed THEN notice is hidden |

---

### 3.11 Notification System

**Quick Summary:** The system sends SMS and Email through a centralized gateway (no per-tenant provider config). SMS quota is managed per tenant by Platform Admin. Notifications are triggered by attendance marking (absentees), result publishing, forgot-password, and manual ad-hoc sends. All notifications are logged in the `notifications` table for delivery tracking and debugging.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-NOT-01 | System shall send SMS to guardians of absent students when triggered | P1 | Attendance session submitted, absent students exist | POST /api/attendance/sessions/{id}/send-sms |
| FR-NOT-02 | System shall send SMS/Email to guardians when results are published | P1 | Exam results published | Auto-triggered on publish |
| FR-NOT-03 | System shall send password reset email | P0 | Forgot-password requested | POST /api/auth/forgot-password |
| FR-NOT-04 | Authorized user shall send ad-hoc SMS/Email | P1 | Authenticated | POST /api/notifications/send |
| FR-NOT-05 | System shall deduct SMS balance per SMS sent | P1 | SMS sent successfully | After send |
| FR-NOT-06 | System shall block SMS sending when balance is 0 | P1 | `sms_balance` = 0 | Before send |
| FR-NOT-07 | System shall log every notification with status, recipient, message | P0 | Notification sent or failed | After attempt |
| FR-NOT-08 | System shall retry failed notifications (configurable) | P1 | Status = FAILED | Cron/queue job |

#### API Endpoints

##### POST /api/notifications/send

Ad-hoc notification.

**Request:**
```json
{
  "channel": "SMS",
  "recipient": "+8801712345678",
  "message": "Your child was absent today."
}
```

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "status": "SENT"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 402 | `SMS_QUOTA_EXCEEDED` | sms_balance = 0 |

##### GET /api/notifications

View notification logs.

**Query params:** `?channel=SMS&status=FAILED&page=1`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "channel": "SMS",
      "recipient": "+8801712345678",
      "status": "FAILED",
      "retry_count": 2,
      "provider_response": "Invalid number",
      "created_at": "2026-07-10T10:00:00Z"
    }
  ]
}
```

#### Data Model

**Table: `notifications`** — DB Dictionary §Table 22.

| Column | Example |
|--------|---------|
| channel | `SMS` |
| recipient | `+8801712345678` |
| message | `Your child was absent today.` |
| status | `SENT` / `FAILED` / `PENDING` |
| trigger_type | `ATTENDANCE` / `RESULT` / `MANUAL` / `SYSTEM` |
| retry_count | `0` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-NOT-01 | Notification failures must NOT roll back the source transaction (attendance, result publish) |
| BR-NOT-02 | Each SMS deducts 1 credit from `tenants.sms_balance` |
| BR-NOT-03 | IF `sms_balance` = 0 THEN SMS is blocked (email still works) |
| BR-NOT-04 | Notification logs are immutable once written |
| BR-NOT-05 | All notifications go through the platform's centralized gateway (no per-tenant provider config) |

#### Open Questions

| # | Question |
|---|----------|
| Q-NOT-01 | What SMS/Email provider will be used in MVP? (Twilio, SendGrid, local?) |
| Q-NOT-02 | Are notification templates customizable per tenant? (No template table yet) |

---

### 3.12 Reports

**Quick Summary:** The Reports module generates PDF ID cards, admission forms, student profiles, Transfer Certificates, character certificates, attendance reports, and result reports. All reports include school branding from Settings. Data is read-only at generation time.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-RPT-01 | System shall generate student ID card PDF with photo, name, registration no, class, section, roll, school details | P1 | Student exists, school branding configured | GET /api/reports/students/{id}/id-card |
| FR-RPT-02 | System shall generate Transfer Certificate PDF | P1 | Student status = TRANSFERRED | GET /api/reports/students/{id}/tc |
| FR-RPT-03 | System shall generate attendance report PDF/Excel | P1 | Attendance records exist | GET /api/reports/attendance |
| FR-RPT-04 | System shall generate result report PDF (per student or per exam) | P1 | Results published | GET /api/reports/results |
| FR-RPT-05 | System shall embed school logo and name from Settings in all reports | P1 | Settings configured | During generation |

#### API Endpoints

##### GET /api/reports/students/{id}/id-card

**Response:** Binary PDF stream (Content-Type: application/pdf)

##### GET /api/reports/attendance?class_id=uuid&academic_year_id=uuid&format=excel

**Response:** Binary Excel stream or PDF

#### Business Rules

| ID | Rule |
|----|------|
| BR-RPT-01 | Reports include school branding from `tenant_settings` |
| BR-RPT-02 | Reports are generated for current academic year by default |
| BR-RPT-03 | Data is read-only — reports never modify data |
| BR-RPT-04 | Reports only include data belonging to the current tenant |

#### Open Questions

| # | Question |
|---|----------|
| Q-RPT-01 | PDF generation library? (DomPDF, Puppeteer, jsPDF?) |
| Q-RPT-02 | Excel export library? (PhpSpreadsheet, xlsx?) |
| Q-RPT-03 | Are reports generated sync or async (queued)? |

---

### 3.13 Dashboard

**Quick Summary:** Role-specific dashboards showing at-a-glance metrics. School Admin sees student count, today's attendance %, recent notices, SMS balance, and quick actions. Manager sees a subset based on their permissions.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-DSH-01 | System shall display School Admin dashboard with all metrics | P0 | Authenticated as School Admin | GET /api/dashboard/school-admin |
| FR-DSH-02 | System shall display Manager dashboard (permissions-dependent) | P0 | Authenticated as Manager | GET /api/dashboard/manager |
| FR-DSH-03 | Dashboard metrics shall be scoped to current academic year | P0 | Current academic year exists | During load |

#### API Endpoints

##### GET /api/dashboard/school-admin

**Response `200 OK`:**
```json
{
  "total_students": 350,
  "attendance_percentage": 95.2,
  "today_attendance_taken": true,
  "recent_notices": [
    { "id": "uuid", "title": "Exam Routine", "published_at": "..." }
  ],
  "sms_balance": 420,
  "sms_quota": 500,
  "quick_actions": [
    { "label": "Add Student", "path": "/students/add" },
    { "label": "Take Attendance", "path": "/attendance" }
  ]
}
```

##### GET /api/dashboard/manager?modules=STUDENT_MANAGEMENT,ATTENDANCE_MANAGEMENT

**Response:** Same format, but only shows widgets for assigned modules.

#### Business Rules

| ID | Rule |
|----|------|
| BR-DSH-01 | Dashboard data is scoped to the current academic year |
| BR-DSH-02 | Managers only see metrics for modules they have View permission on |
| BR-DSH-03 | Dashboard fetches real-time data (no caching for MVP) |

---

### 3.14 Audit Logging

**Quick Summary:** Every important business action (create, update, delete, publish, permission change, etc.) is recorded in `audit_logs` with actor, action, old/new values, IP, and timestamp. Logs are append-only and immutable.

#### Functional Requirements

| ID | Description | Priority | Preconditions | Trigger |
|----|-------------|----------|---------------|---------|
| FR-AUD-01 | System shall log all CUD operations on student data | P0 | Action performed | After successful action |
| FR-AUD-02 | System shall log attendance corrections after submission | P0 | Attendance session modified | After update |
| FR-AUD-03 | System shall log result publish/unpublish | P0 | Result published/unpublished | After action |
| FR-AUD-04 | System shall log permission changes | P0 | Manager permissions updated | After update |
| FR-AUD-05 | System shall log SMS balance changes | P0 | Balance adjusted | After adjustment |
| FR-AUD-06 | System shall log tenant activation/deactivation | P0 | Tenant status changed | After change |
| FR-AUD-07 | School Admin shall view audit logs for their tenant | P0 | Authenticated | GET /api/audit-logs |
| FR-AUD-08 | Platform Admin shall view audit logs across all tenants | P1 | Authenticated as Platform Admin | GET /api/admin/audit-logs |

#### API Endpoints

##### GET /api/audit-logs

**Query params:** `?module=STUDENT_MANAGEMENT&action=CREATED&page=1`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "user": { "id": "uuid", "full_name": "John Doe", "role": "SCHOOL_ADMIN" },
      "module": "STUDENT_MANAGEMENT",
      "action": "CREATED",
      "entity_type": "Student",
      "description": "Created student Alice Johnson",
      "old_values": null,
      "new_values": { "registration_no": "26000001", "full_name": "Alice Johnson" },
      "ip_address": "192.168.1.1",
      "created_at": "2026-07-10T10:00:00Z"
    }
  ],
  "meta": { "total": 1, "page": 1 }
}
```

#### Data Model

**Table: `audit_logs`** — DB Dictionary §Table 23.

| Column | Example |
|--------|---------|
| tenant_id | `uuid` |
| user_id | `uuid` |
| module | `STUDENT_MANAGEMENT` |
| action | `CREATED` |
| entity_type | `Student` |
| entity_id | `uuid` |
| description | `Created student Alice Johnson` |
| old_values | `null` |
| new_values | `{"registration_no": "26000001"}` |

#### Business Rules

| ID | Rule |
|----|------|
| BR-AUD-01 | Audit logs are append-only — existing records must never be modified or deleted |
| BR-AUD-02 | Routine reads and simple queries are NOT logged |
| BR-AUD-03 | Only business-critical actions are logged (see AUDIT_ACTION enum) |
| BR-AUD-04 | `old_values` and `new_values` use JSONB for flexible schema |

---

## 4. Non-Functional Requirements

### 4.1 Performance

| Metric | Target | Notes |
|--------|--------|-------|
| API response time (P95) | < 500ms | Most CRUD endpoints; report generation may be slower |
| Page load time (P95) | < 2-3 seconds | Can improve with Redis caching post-MVP |
| Concurrent users per tenant | 50-100+ | Scales horizontally |
| Total tenants (MVP) | 100+ | Shared DB with tenant_id scoping |

### 4.2 Security

| Requirement | Detail |
|-------------|--------|
| Authentication | JWT access (15 min) + refresh tokens (7 days) with rotation |
| Password storage | bcrypt |
| Data in transit | TLS 1.3 |
| Data at rest | AES-256 |
| API security | Rate limiting (login: 5/min/IP), CORS, input sanitization |
| Audit logging | All CUD operations logged with user, tenant, timestamp, IP |

### 4.3 Scalability

- Application layer: horizontally scalable (stateless)
- Database: read replicas for reporting (future)
- Caching: Redis for permission lookups, class lists, settings
- File storage: Cloudinary (CDN, auto-optimization, transformations)

### 4.4 Availability

| Metric | Target |
|--------|--------|
| Uptime SLA | 99.5% (MVP), 99.9% (post-MVP) |
| Backup | Daily automated, 30-day retention |

---

## 5. Appendices

### A. Requirements Traceability Matrix

| Module | FR IDs | PRD § | DB Tables |
|--------|--------|-------|-----------|
| Platform & Multi-Tenant | FR-PLT-01 → FR-PLT-10 | §4, §5, §9 | tenants, tenant_settings, tenant_modules |
| Authentication | FR-AUTH-01 → FR-AUTH-14 | §6.1, §10 | platform_admins, users |
| Academic Structure | FR-ACA-01 → FR-ACA-08 | §6.9, §7.3 | academic_years, classes, sections, subjects |
| Settings | FR-SET-01 → FR-SET-03 | §6.9 | tenant_settings |
| User & Permission Mgmt | FR-UP-01 → FR-UP-06 | §7.1, §8 | users, manager_permissions |
| Module Management | FR-MM-01 → FR-MM-03 | §7.2 | tenant_modules |
| Student Management | FR-STU-01 → FR-STU-13 | §6.3, §9 | applications, students, student_enrollments |
| Attendance | FR-ATT-01 → FR-ATT-07 | §6.4 | attendance_sessions, attendance_records |
| Results | FR-RES-01 → FR-RES-08 | §6.5 | exams, exam_subjects, marks, grade_scales |
| Notice Board | FR-NTC-01 → FR-NTC-06 | §6.6 | notices |
| Notifications | FR-NOT-01 → FR-NOT-08 | §6.7 | notifications |
| Reports | FR-RPT-01 → FR-RPT-05 | §6.8 | (read-only) |
| Dashboard | FR-DSH-01 → FR-DSH-03 | §6.2 | (aggregated) |
| Audit Logging | FR-AUD-01 → FR-AUD-08 | §10 | audit_logs |

### B. Error Code Reference

| Code | HTTP Status | Description | Used By |
|------|-------------|-------------|---------|
| `INVALID_CREDENTIALS` | 401 | Wrong email or password | Auth |
| `ACCOUNT_INACTIVE` | 403 | User deactivated | Auth |
| `TENANT_INACTIVE` | 403 | Tenant deactivated | Auth |
| `INVALID_REFRESH_TOKEN` | 401 | Refresh token not found or revoked | Auth |
| `TOKEN_EXPIRED` | 401 | Token has expired | Auth |
| `TOKEN_REUSE_DETECTED` | 401 | Revoked token reused — all tokens revoked | Auth |
| `TOKEN_ALREADY_USED` | 400 | Reset token already used | Auth |
| `INCORRECT_CURRENT_PASSWORD` | 400 | Wrong current password | Auth |
| `SAME_PASSWORD` | 400 | New password same as current | Auth |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests | Auth |
| `VALIDATION_ERROR` | 422 | Request body validation failed | All |
| `SUBDOMAIN_TAKEN` | 409 | Subdomain already in use | Platform |
| `ALREADY_REVIEWED` | 409 | Application already processed | Student |
| `SMS_QUOTA_EXCEEDED` | 402 | No SMS credits remaining | Notifications |
| `DUPLICATE_NAME` | 409 | Duplicate entity name | Academic |
| `OVERLAPPING_DATES` | 409 | Date range overlaps | Academic |
| `CURRENT_YEAR_EXISTS` | 400 | Another academic year is already current | Academic |
| `EMAIL_EXISTS` | 409 | Email already registered | User Mgmt |
| `INSUFFICIENT_BALANCE` | 400 | SMS balance adjustment would go negative | Platform |
| `MARKS_INCOMPLETE` | 400 | Not all marks entered | Results |
| `ALREADY_SUBMITTED` | 409 | Attendance already submitted | Attendance |

### C. Open Questions Log

| ID | Module | Question | Status |
|----|--------|----------|--------|
| Q-PLT-01 | Platform | Should tenant deletion be implemented in MVP? | Open |
| Q-PLT-02 | Platform | What is the default module set for new tenants? | Open |
| Q-AUTH-01 | Auth | Access token expiry configurable? | Open |
| Q-AUTH-02 | Auth | Refresh token storage — add `refresh_tokens` model to Prisma, or use signed JWTs without DB lookup? | Open |
| Q-AUTH-03 | Auth | Single session per user enforced in MVP? | Open |
| Q-ACA-01 | Academic | Close endpoint or just is_current toggle? | Open |
| Q-ACA-02 | Academic | Verify no pending data before closing year? | Open |
| Q-SET-01 | Settings | Where are notification templates stored? | Open |
| Q-SET-02 | Settings | Password policy configurable or hardcoded? | Open |
| Q-UP-01 | Permissions | Can School Admin edit Manager email/password? | Open |
| Q-UP-02 | Permissions | can_print separate from can_export in UI? | Open |
| Q-STU-01 | Student | Admission form public or authenticated? | Open |
| Q-STU-02 | Student | Application number format? | Open |
| Q-STU-03 | Student | CAPTCHA on public admission form? | Open |
| Q-STU-04 | Student | Reactivate TRANSFERRED or GRADUATED students? | Open |
| Q-ATT-01 | Attendance | Attendance status granularity? | Open |
| Q-ATT-02 | Attendance | Month closure in MVP? | Open |
| Q-ATT-03 | Attendance | Backdate limit value? | Open |
| Q-RES-01 | Results | Grid view or one-by-one marks entry? | Open |
| Q-RES-02 | Results | Multiple managers per exam subject-wise? | Open |
| Q-RES-03 | Results | Grade scale per tenant or per year? | Open |
| Q-NOT-01 | Notifications | SMS/Email provider in MVP? | Open |
| Q-NOT-02 | Notifications | Customizable notification templates? | Open |
| Q-RPT-01 | Reports | PDF generation library? | Open |
| Q-RPT-02 | Reports | Excel export library? | Open |
| Q-RPT-03 | Reports | Sync or async report generation? | Open |

---

*End of SRS*

