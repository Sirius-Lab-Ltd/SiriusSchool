# Sirius-Skool — Software Requirements Specification

> **Version:** 1.0  
> **Status:** Draft  
> **Date:** July 18, 2026  
> **Source Documents:** PRD v1.0 (`docs/PRD.md`), DB Dictionary (`docs/DB/DB-Dictionary.md`)

---

## Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | Jul 6 | — | Initial Auth module draft |
| 0.2 | Jul 10 | — | Full rewrite: all MVP modules, API specs, data models, business rules |
| 1.0 | Jul 18 | — | PRD-vs-SRS gap analysis — added FR-STU-14, FR-NTC-07, FR-RPT-06/07/08, FR-NOT-09, FR-ACA-09, missing API specs, BRs, updated RTM ranges, settings fields, dashboard warning |
| 1.1 | Jul 21 | — | Student soft-delete (FR-STU-08), result sheet view (FR-RES-09), Manager profile edit (FR-UP-08), unified report load (FR-RPT-06), Platform Admin dashboard/analytics/settings (FR-DSH-04, FR-PLT-12/13) |

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
   - A. Complete API Endpoint Reference
   - B. Complete Business Rules
   - C. Requirements Traceability Matrix
   - D. Error Code Reference
   - E. Open Questions Log

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
| **Manager** | Staff member with module-level permissions (stored in `users` table, role `MANAGER`) |
| **JWT** | JSON Web Token for API authentication |
| **Access Token** | Short-lived JWT (15 min default) for endpoint authorization |
| **SMS** | Short Message Service for guardian notifications |
| **Registration No** | Permanent student identifier (`YY + sequence`, e.g. `26000001`) |
| **Roll Number** | Per-section, per-year numeric identifier for daily use |
| **PRD** | Product Requirements Document |
| **SRS** | Software Requirements Specification |

#### Requirement ID Prefixes

Each functional requirement (FR), business rule (BR), and open question (Q) carries a module prefix. The prefixes and their meanings are:

| Prefix | Module |
|--------|--------|
| `PLT` | Platform & Multi-Tenant Foundation |
| `AUTH` | Authentication |
| `ACA` | Academic Structure |
| `SET` | Settings |
| `UP` | User & Permission Management |
| `MM` | Module Management |
| `STU` | Student Management & Admission |
| `ATT` | Attendance Management |
| `RES` | Result Management |
| `NTC` | Notice Board |
| `NOT` | Notification System |
| `RPT` | Reports |
| `DSH` | Dashboard |
| `AUD` | Audit Logging |

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
| **Tier 2** | School Admin → Manager | Module-level (single toggle) | School Admin grants module access via a single toggle per module. Manager gets predefined actions per PRD (V/C/E for most modules; V/C for SMS Log; View-only for Reports). Delete is always Admin-only. |

**School Admin has full access** to all enabled modules — no explicit permission records are stored. **Managers** need explicit records in `manager_permissions` — one record per granted module.

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
  │    ├── Classes → Shifts → Sections
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

> **Data Model:** Full schema in DB Dictionary §Table 2 (tenants), §Table 3 (tenant_settings), §Table 4 (tenant_modules).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-PLT-01 | Tenant Creation | Platform Admin shall create a new tenant with name, subdomain, assigned modules, SMS quota, starting registration sequence, and initial School Admin credentials | P0 | Platform Admin is authenticated | POST /api/v1/admin/tenants |
| 2 | FR-PLT-02 | Subdomain Uniqueness | System shall validate that subdomain is globally unique | P0 | Tenant creation request | Validation rule |
| 3 | FR-PLT-03 | Auto-Create School Admin | System shall create School Admin user in `users` table during tenant creation (same transaction) | P0 | Tenant record created | Side-effect of tenant creation |
| 4 | FR-PLT-04 | Auto-Create Tenant Settings | System shall create a `tenant_settings` record for the new tenant | P0 | Tenant record created | Side-effect of tenant creation |
| 5 | FR-PLT-05 | Seed Tenant Modules | System shall seed `tenant_modules` records for all assigned modules | P0 | Tenant record created | Side-effect of tenant creation |
| 6 | FR-PLT-06 | Activate/Deactivate Tenant | Platform Admin shall activate/deactivate a tenant | P0 | Tenant exists | PATCH /api/v1/admin/tenants/{id} |
| 7 | FR-PLT-07 | Deactivated Tenant Access | System shall allow login for deactivated tenants but block all API operations with 403. Login response includes `tenant.is_active = false` for frontend to render deactivation page. | P0 | `tenants.is_active = false` | Runtime middleware check |
| 8 | FR-PLT-08 | List Tenants | Platform Admin shall view a list of all tenants with status | P0 | Platform Admin is authenticated | GET /api/v1/admin/tenants |
| 9 | FR-PLT-09 | Adjust SMS Balance | Platform Admin shall adjust SMS balance for a tenant | P0 | Tenant exists | PATCH /api/v1/admin/tenants/{id}/sms-balance |
| 10 | FR-PLT-10 | View Cross-Tenant Notification Logs | Platform Admin shall view notification logs across all tenants | P1 | Platform Admin is authenticated | GET /api/v1/admin/notifications |
| 11 | FR-PLT-11 | Reset Tenant User Password | Platform Admin shall reset any tenant user's password (School Admin or Manager) without current password | P0 | Platform Admin is authenticated, user exists | PATCH /api/v1/admin/users/{id}/reset-password |
| 12 | FR-PLT-12 | Platform Analytics | Platform Admin shall view usage analytics across all tenants | P1 | Authenticated as Platform Admin | GET /api/v1/admin/analytics |
| 13 | FR-PLT-13 | Platform Settings | Platform Admin shall update platform-wide settings | P1 | Authenticated as Platform Admin | PATCH /api/v1/admin/settings |

---





#### **1.** FR-PLT-01: Tenant Creation

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-01 |
| **Description** | Platform Admin shall create a new tenant with name, subdomain, assigned modules, SMS quota, starting registration sequence, and initial School Admin credentials |
| **Priority** | P0 |
| **Preconditions** | Platform Admin is authenticated |
| **Trigger** | POST /api/v1/admin/tenants |

**Explanation:**

**FR-PLT-01** is the first and most foundational requirement — it creates a new tenant (school) in the system. When a Platform Admin calls `POST /api/v1/admin/tenants`, they provide:

| Field | Explanation |
|-------|-------------|
| **name** | School name (e.g., "Springfield School") |
| **subdomain** | Unique slug used for the tenant's login URL (`{subdomain}.sirius-skool.com`). Must be globally unique. |
| **modules** | Array of module keys that pre-authorizes this tenant to use specific features. School Admin can toggle these on/off later via Module Management, but cannot enable modules outside this list. |
| **sms_quota** | The number of SMS credits pre-allocated to the tenant. This acts as a prepaid wallet for sending text message notifications (e.g., absence alerts, result publication). Each SMS sent deducts 1 credit. When the balance reaches 0, SMS sending is blocked but email continues to work. Platform Admin can top up later via FR-PLT-09. |
| **starting_sequence** | The starting value for student registration numbers (e.g., `1` → first student gets `26000001`). |
| **school_admin** | Initial admin credentials — `full_name`, `email`, `password`. This user is auto-created in the `users` table with role `SCHOOL_ADMIN`. |

**What happens atomically (single transaction, BR-PLT-01):**
1. `tenants` row created (with `is_active: true`, `current_student_sequence: <starting_sequence>`)
2. `users` row created for the School Admin
3. `tenant_settings` row created (empty/defaults)
4. `tenant_modules` rows seeded for each assigned module

**Validation:**
- Subdomain uniqueness → `409 SUBDOMAIN_TAKEN`
- Invalid fields → `422 VALIDATION_ERROR`

**FR-PLT-01.1 Module Pre-Authorization — Detailed Breakdown**

The `modules` field determines **what features the tenant is allowed to use**. It implements a two-tier permission gating:

| Concept | Detail |
|---------|--------|
| **Purpose** | Controls feature access per tenant. Platform Admin decides which modules a school is entitled to use. |
| **Module ENUM** | `STUDENT_MANAGEMENT`, `ATTENDANCE_MANAGEMENT`, `RESULT_MANAGEMENT`, `NOTICE_BOARD`, `REPORTS`, `NOTIFICATIONS` |
| **Infrastructure modules** | Authentication, Academic Structure, Settings, User & Permission Mgmt, Module Management, Dashboard, Audit — these are always on and not part of the toggle system |
| **Storage** | Rows written to `tenant_modules` table during tenant creation (FR-PLT-05). Each row has `{tenant_id, module, is_enabled}`. |
| **Initial state** | All assigned modules created with `is_enabled: true`. |
| **Toggle authority** | School Admin can disable/re-enable via Module Management (FR-MM-02), but only within the assigned pool (BR-MM-02). |
| **Disable effect** | Module hidden from sidebar, API endpoints return 403, existing data is preserved (FR-MM-03, BR-MM-04). |
| **Re-enable** | Restores sidebar visibility and API access to existing data (BR-MM-04). |
| **Adding modules later** | Not covered by a Platform Admin endpoint in MVP — requires direct DB update or future admin feature. |

**Example flow:**
1. Platform Admin creates tenant with `modules: ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT", "RESULT_MANAGEMENT"]`
2. `tenant_modules` seeded with 3 rows, all `is_enabled: true`
3. School Admin can toggle any of these 3 on/off from the Module Management page
4. School Admin **cannot** enable `NOTICE_BOARD` — it was not in the assigned list
5. If Platform Admin later adds `NOTICE_BOARD` to `tenant_modules`, it becomes available for toggling

**Separation of concerns:**
- **Platform Admin** decides *what's available* — licensing/business decision
- **School Admin** decides *what's active* — operational decision

**FR-PLT-01.2 SMS Credit System — Detailed Breakdown**

The SMS credit system is a prepaid balance model that controls how many text messages a tenant can send:

| Concept | Detail |
|---------|--------|
| **Where stored** | `tenants.sms_balance` column (integer, default = `sms_quota` at creation) |
| **Initial balance** | Set by `sms_quota` during tenant creation. Stored in `sms_balance`. |
| **Consumption rate** | 1 credit per SMS sent. Applies to absent alerts (FR-ATT-06), result notifications (FR-NOT-02), and ad-hoc sends (FR-NOT-04). |
| **When deducted** | Deducted atomically after the SMS provider confirms successful delivery (FR-NOT-05). |
| **When blocked** | Balance = 0 → `402 SMS_QUOTA_EXCEEDED`. Email is NOT affected. |
| **Top-up** | Platform Admin adds credits via PATCH `/api/v1/admin/tenants/{id}/sms-balance` (FR-PLT-09). |
| **Negative protection** | Balance can never go below 0. Deduction attempts that would underflow are rejected. |
| **Audit trail** | Every adjustment and deduction is logged in `audit_logs` (FR-AUD-05). |

**Example lifecycle:**
1. Tenant created with `sms_quota: 500` → `sms_balance = 500`
2. Daily attendance: 5 absent students → 5 SMS sent → `sms_balance = 495`
3. Results published: 30 SMS sent to guardians → `sms_balance = 465`
4. Over weeks, balance gradually depletes to `sms_balance = 0`
5. Next SMS attempt → `402 SMS_QUOTA_EXCEEDED` — admin sees warning on dashboard (FR-DSH-01)
6. Platform Admin adds 200 credits → `sms_balance = 200` — SMS resumes

**FR-PLT-01.3 Registration Number Sequence — Detailed Breakdown**

`starting_sequence` is the initial value for the tenant's student registration counter. Each time a student is admitted, the counter is atomically read, used, and incremented by 1.

**Registration number format:** `YY` (2-digit admission year) + `NNNNNN` (6-digit zero-padded sequence)

- Example: `26000001` = admitted in 20**26**, sequence **000001**
- The `YY` prefix changes with the admission year, and **the sequence counter resets** at the start of each new academic year

**How the sequence counter works:**

| Concept | Detail |
|---------|--------|
| **Stored in** | `tenants.current_student_sequence` column |
| **Initial value** | Set to `starting_sequence` at tenant creation |
| **Increment** | +1 after each successful student creation |
| **Atomicity** | Read + increment uses `SELECT ... FOR UPDATE` to prevent race conditions (BR-STU-02) |
| **Scope** | Per-tenant, resets at the start of each academic year |
| **Reset trigger** | Setting a new academic year as current (FR-ACA-08) resets the counter to `starting_sequence` |
| **Permanence** | Registration numbers never change, even if a student leaves or graduates (BR-STU-03) |

**Example lifecycle (starting_sequence = 1):**

| # | Event | Year | Sequence read | Registration No |
|---|-------|------|---------------|-----------------|
| 1 | First admission | 2026 | 1 | `26000001` |
| 2 | Second admission | 2026 | 2 | `26000002` |
| ... | ... | ... | ... | ... |
| 150 | 150th admission | 2026 | 150 | `26000150` |
| 151 | First admission next year | 2027 | 1 | `27000001` |
| 152 | Another admission | 2027 | 2 | `27000002` |

Notice: the `YY` prefix changed from `26` to `27`, and the sequence counter reset to 1 for the new academic year.

**Why make it configurable?** `starting_sequence` allows schools migrating from a legacy system to avoid conflicts with existing student IDs. For example, if a school already has 500 students in their old system and is migrating to Sirius-Skool, they can set `starting_sequence: 501` so new registration numbers start at `26000501` instead of `26000001`. The sequence resets each academic year, so `starting_sequence` serves as the base for every year's registration counter.

**API Endpoint(s):**

##### POST /api/v1/admin/tenants

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-01 | IF tenant is created THEN School Admin user, settings record, and module assignments are created in the same transaction |
| BR-PLT-03 | IF subdomain already exists THEN return 409 SUBDOMAIN_TAKEN |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-PLT-02 | What is the default module set for a new tenant? |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-PLT-02: Subdomain Uniqueness

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-02 |
| **Description** | System shall validate that subdomain is globally unique |
| **Priority** | P0 |
| **Preconditions** | Tenant creation request |
| **Trigger** | Validation rule |

**Explanation:**

**FR-PLT-02** ensures that each tenant's subdomain is globally unique, preventing URL collisions and tenant impersonation. When a Platform Admin submits a tenant creation request, the system checks the `tenants` table for an existing row with the same subdomain before proceeding.

**Validation:**
- Subdomain already exists → `409 SUBDOMAIN_TAKEN`

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-03 | IF subdomain already exists THEN return 409 SUBDOMAIN_TAKEN |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-PLT-03: Auto-Create School Admin

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-03 |
| **Description** | System shall create School Admin user in `users` table during tenant creation (same transaction) |
| **Priority** | P0 |
| **Preconditions** | Tenant record created |
| **Trigger** | Side-effect of tenant creation |

**Explanation:**

**FR-PLT-03** automatically creates the School Admin user during tenant creation. The `users` table gets a new row with the provided name, email, hashed password, `role = SCHOOL_ADMIN`, and `tenant_id` pointing to the newly created tenant. This happens in the same database transaction (BR-PLT-01) — if user creation fails, the entire tenant creation rolls back.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-01 | IF tenant is created THEN School Admin user, settings record, and module assignments are created in the same transaction |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-PLT-04: Auto-Create Tenant Settings

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-04 |
| **Description** | System shall create a `tenant_settings` record for the new tenant |
| **Priority** | P0 |
| **Preconditions** | Tenant record created |
| **Trigger** | Side-effect of tenant creation |

**Explanation:**

**FR-PLT-04** creates a `tenant_settings` record for every new tenant with default values (empty logo, address, phone, email). This guarantees that BR-SET-02 ("Every tenant has exactly one settings record") is satisfied from day one. School Admin can update these settings later via the Settings module (FR-SET-02).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-01 | IF tenant is created THEN School Admin user, settings record, and module assignments are created in the same transaction |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-PLT-05: Seed Tenant Modules

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-05 |
| **Description** | System shall seed `tenant_modules` records for all assigned modules |
| **Priority** | P0 |
| **Preconditions** | Tenant record created |
| **Trigger** | Side-effect of tenant creation |

**Explanation:**

**FR-PLT-05** seeds `tenant_modules` rows for all modules assigned by the Platform Admin during tenant creation. Each assigned module gets a record with `is_enabled = true`. This determines the pool of modules the School Admin can later toggle on/off — they cannot enable modules not assigned here (BR-MM-02).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-01 | IF tenant is created THEN School Admin user, settings record, and module assignments are created in the same transaction |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-PLT-06: Activate/Deactivate Tenant

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-06 |
| **Description** | Platform Admin shall activate/deactivate a tenant |
| **Priority** | P0 |
| **Preconditions** | Tenant exists |
| **Trigger** | PATCH /api/v1/admin/tenants/{id} |

**Explanation:**

**FR-PLT-06** allows the Platform Admin to activate or deactivate a tenant via `PATCH /api/v1/admin/tenants/{id}`. The request body includes `{ "is_active": false }` to deactivate. Deactivation immediately blocks all user access (FR-PLT-07). Reactivation restores login capability, but previous sessions are not restored.

**FR-PLT-06.1 Activate/Deactivate — Detailed Breakdown**

**FR-PLT-06** allows the Platform Admin to toggle a tenant's active status via `PATCH /api/v1/admin/tenants/{id}`. This is how the Platform Admin controls which schools can access the system.

**Request:**
```json
{ "is_active": false }
```

**What happens on deactivation:**
1. `tenants.is_active` set to `false`
2. Login still succeeds — the login response includes `tenant.is_active = false` (FR-PLT-07)
3. The frontend renders a deactivation info page with message: *"Your school has been deactivated. Contact admin@support.com for reactivation."*
4. All API operations are blocked by middleware with `403 TENANT_INACTIVE`
5. Active sessions are not proactively revoked, but each API request validates tenant activity

**What happens on reactivation:**
1. `tenants.is_active` set to `true`
2. Existing users who were logged in see the deactivation page next time they make an API request — after reactivation, subsequent requests succeed
3. Users who need to re-authenticate can log in normally; the login response now returns `tenant.is_active = true`

**Why PATCH instead of PUT?**

| Method | Behavior | Example |
|--------|----------|---------|
| **PATCH** | Partial update — only send the fields that change | `PATCH /api/v1/admin/tenants/{id}` with `{ "is_active": false }` |
| **PUT** | Full replacement — must send every field | Would require `{ "name": "...", "subdomain": "...", "is_active": false, ... }` |

**Edge cases:**

| Scenario | Behavior |
|----------|----------|
| Deactivate active tenant | Login still works but returns `tenant.is_active = false`. Frontend shows deactivation page. Middleware blocks all API operations with 403. |
| Reactivate deactivated tenant | Login now returns `tenant.is_active = true`. Users who kept their session can make API requests again (tokens still valid). Users on the deactivation page refresh and see the dashboard. |
| Deactivate already-deactivated tenant | No-op — `is_active` stays `false`, response returns `{ "is_active": false }`. |
| Attendance session in progress during deactivation | The session save fails with 403. Data integrity is maintained — no partial writes. |

**API Endpoint(s):**

##### PATCH /api/v1/admin/tenants/{id}

Update tenant (activate/deactivate, change name, etc.). Deactivated tenants can still log in but are shown a deactivation info page. All API operations are blocked with 403 TENANT_INACTIVE.

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-02 | IF tenant.is_active = false THEN login proceeds but tenant.is_active=false is returned in the login response. All API operations return 403 TENANT_INACTIVE. |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-PLT-01 | Should tenant deletion be implemented in MVP? — Closed: Only soft-deactivation (is_active = false). Records are never hard-deleted. |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-PLT-07: Deactivated Tenant Access

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-07 |
| **Description** | System shall allow login for deactivated tenants but block all API operations with 403. Login response includes `tenant.is_active = false` for frontend to render deactivation page. |
| **Priority** | P0 |
| **Preconditions** | `tenants.is_active = false` |
| **Trigger** | Runtime middleware check |

**Explanation:**

**FR-PLT-07** allows login for deactivated tenants but blocks all API operations. When `tenants.is_active = false`, the login response includes `tenant.is_active = false`. The frontend renders a deactivation info page instead of the dashboard. All subsequent API requests are blocked by middleware with `403 TENANT_INACTIVE`, making deactivation effectively immediate for all operations.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-02 | IF tenant.is_active = false THEN login proceeds but tenant.is_active=false is returned in the login response. All API operations return 403 TENANT_INACTIVE. |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **8.** FR-PLT-08: List Tenants

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-08 |
| **Description** | Platform Admin shall view a list of all tenants with status |
| **Priority** | P0 |
| **Preconditions** | Platform Admin is authenticated |
| **Trigger** | GET /api/v1/admin/tenants |

**Explanation:**

**FR-PLT-08** provides the Platform Admin a list of all tenants with their name, subdomain, active status, SMS balance, and creation date via `GET /api/v1/admin/tenants`. This is the main tenant management overview.

**API Endpoint(s):**

##### GET /api/v1/admin/tenants

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **9.** FR-PLT-09: Adjust SMS Balance

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-09 |
| **Description** | Platform Admin shall adjust SMS balance for a tenant |
| **Priority** | P0 |
| **Preconditions** | Tenant exists |
| **Trigger** | PATCH /api/v1/admin/tenants/{id}/sms-balance |

**Explanation:**

**FR-PLT-09** allows the Platform Admin to adjust a tenant's SMS balance via `PATCH /api/v1/admin/tenants/{id}/sms-balance`. The adjustment can be positive (add credits) or negative (deduct). Negative adjustments are validated to prevent the balance from going below zero.

**FR-PLT-09.1 SMS Balance Adjustment — Detailed Breakdown**

**FR-PLT-09** allows the Platform Admin to add or deduct SMS credits for a tenant.

**Request:**
```json
{ "adjustment": 200 }
```
- **Positive value** → adds credits (e.g., `+200` → balance goes up)
- **Negative value** → deducts credits (e.g., `-50` → balance goes down)

**What happens:**
1. Current `sms_balance` is read from the tenant
2. Adjustment is applied: `new_balance = current_balance + adjustment`
3. New balance is saved to `tenants.sms_balance`
4. The adjustment is logged in `audit_logs` (FR-AUD-05) with the old value, new value, and adjustment amount

**Validation:**
- Adjustment would make balance negative → `400 INSUFFICIENT_BALANCE`
- Tenant not found → `404`

**Example lifecycle:**

| Event | Adjustment | Balance Before | Balance After |
|-------|-----------|----------------|---------------|
| Tenant created | — | — | 500 |
| SMSes sent over time | — | 500 | 0 |
| Platform Admin tops up | +200 | 0 | 200 |
| More SMSes sent | — | 200 | 50 |
| Platform Admin deducts for overuse | -100 | 50 | ❌ `400 INSUFFICIENT_BALANCE` |

**API Endpoint(s):**

##### PATCH /api/v1/admin/tenants/{id}/sms-balance

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-PLT-04 | IF SMS balance adjustment would go below 0 THEN return 400 |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **10.** FR-PLT-10: View Cross-Tenant Notification Logs

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-10 |
| **Description** | Platform Admin shall view notification logs across all tenants |
| **Priority** | P1 |
| **Preconditions** | Platform Admin is authenticated |
| **Trigger** | GET /api/v1/admin/notifications |

**Explanation:**

**FR-PLT-10** provides a cross-tenant notification log view for the Platform Admin via `GET /api/v1/admin/notifications`. This enables monitoring of SMS/Email delivery status across all tenants for troubleshooting and auditing.

**API Endpoint(s):**

##### GET /api/v1/admin/notifications

Platform Admin views notification logs across all tenants.

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
  ],
  "meta": { "total": 1, "page": 1 }
}
```

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **11.** FR-PLT-11: Reset Tenant User Password

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-11 |
| **Description** | Platform Admin shall reset any tenant user's password (School Admin or Manager) without current password |
| **Priority** | P0 |
| **Preconditions** | Platform Admin is authenticated, user exists |
| **Trigger** | PATCH /api/v1/admin/users/{id}/reset-password |

**Explanation:**

**FR-PLT-11** allows the Platform Admin to reset any tenant user's password (School Admin or Manager) without requiring the current password. There is no self-service forgot-password flow.

**How it works:**
1. Platform Admin calls `PATCH /api/v1/admin/users/{id}/reset-password` with `{ "new_password": "NewPass123" }`
2. System updates the user's `password_hash` in the `users` table
3. System increments `token_version` to invalidate all existing sessions (BR-AUTH-08)
4. User must log in again with the new password

**Validation:**
- User not found → `404 USER_NOT_FOUND`
- Only Platform Admin → `403 FORBIDDEN`

**API Endpoint(s):**

##### PATCH /api/v1/admin/users/{id}/reset-password

Reset any tenant user's password (Platform Admin only).

**Request:**
```json
{
  "new_password": "NewPass123"
}
```

**Response `200 OK`:**
```json
{
  "message": "Password reset successfully",
  "user_id": "uuid",
  "role": "SCHOOL_ADMIN"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `USER_NOT_FOUND` | User not found |
| 403 | `FORBIDDEN` | Only Platform Admin can reset passwords |

**No related business rules or open questions.**

---

#### Edge Cases & Error States

| Scenario | Behavior |
|----------|----------|
| Create tenant with existing subdomain | 409 CONFLICT |
| Deactivate active tenant | All active sessions invalidated immediately |
| Reactivate tenant | All users can log in again (sessions not restored) |
| Set SMS balance to 0 | SMS sending blocked, email still works |
| Hard-delete tenant | Not supported — only soft-deactivation (is_active = false). Data is preserved; record is never hard-deleted. |

---

### 3.2 Authentication

**Quick Summary:** All users authenticate through this module. Platform Admin uses a separate login page (`admin.sirius-skool.com`) and authenticates against the `platform_admins` table. School Admin and Managers log in via their tenant subdomain (`{tenant}.sirius-skool.com`) and authenticate against the `users` table. The system issues JWT access tokens (15 min) and refresh tokens (7 days) with rotation and reuse detection. Session management supports single-session-per-user enforcement (optional, configurable per tenant). Password management: School Admin and Platform Admin can change their own password (requires current password). Platform Admin can reset any tenant user's password. School Admin can reset Manager passwords. Manager cannot change or reset passwords.

> **Data Model:** Full schema in DB Dictionary §Table 1 (platform_admins), §Table 5 (users), §Table 6 (refresh_tokens).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-AUTH-01 | Platform Admin Authentication | System shall authenticate Platform Admin using email + password against `platform_admins` table | P0 | Admin page (`admin.sirius-skool.com`) | POST /api/v1/admin/auth/login |
| 2 | FR-AUTH-02 | Tenant-Scoped Authentication | System shall authenticate School Admin/Manager using email + password against `users` table, scoped by tenant subdomain | P0 | Tenant is active, user is active | POST /api/v1/auth/login |
| 3 | FR-AUTH-03 | Tenant Resolution | System shall extract tenant from subdomain in Host header (with X-Tenant-Slug fallback for dev) | P0 | Login page loaded | Before authentication |
| 4 | FR-AUTH-04 | Token Issuance | System shall issue JWT access token (15 min) + refresh token (UUID, 7 days) on successful login | P0 | Credentials valid | After authentication |
| 5 | FR-AUTH-05 | Role-Based Redirection | System shall redirect to role-specific dashboard after login | P0 | Login succeeds | After token issuance |
| 6 | FR-AUTH-06 | Logout & Token Revocation | System shall revoke refresh token on logout | P0 | User is authenticated | POST /api/v1/auth/logout |
| 7 | FR-AUTH-07 | Token Refresh & Rotation | System shall issue new access token using valid refresh token (rotation + reuse detection) | P0 | Refresh token is valid | POST /api/v1/auth/refresh |
| 8 | FR-AUTH-08 | Change Own Password | System shall allow School Admin and Platform Admin to change their own password (requires current password). Manager cannot change password. | P0 | User is authenticated | POST /api/v1/auth/change-password |
| 9 | FR-AUTH-09 | User Profile & Permissions | System shall return current user profile + permissions | P0 | Valid access token | GET /api/v1/auth/me |
| 10 | FR-AUTH-10 | Inactive User Login Block | System shall block login for inactive users | P0 | `users.is_active = false` | Login attempt |
| 11 | FR-AUTH-11 | Inactive Tenant Access | System shall allow login for inactive tenants but return 403 FORBIDDEN on all subsequent API requests. Frontend displays deactivation info page. | P0 | `tenants.is_active = false` | Login response |
| 12 | FR-AUTH-12 | Rate Limiting | System shall rate-limit login attempts (5 failed per minute per IP) | P1 | Rate exceeded | Login attempt |

---





#### **1.** FR-AUTH-01: Platform Admin Authentication

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-01 |
| **Description** | System shall authenticate Platform Admin using email + password against `platform_admins` table |
| **Priority** | P0 |
| **Preconditions** | Admin page (`admin.sirius-skool.com`) |
| **Trigger** | POST /api/v1/admin/auth/login |

**Explanation:**

**FR-AUTH-01** authenticates Platform Admin credentials against the `platform_admins` table at the dedicated admin login page (`admin.sirius-skool.com`). When the user submits `POST /api/v1/admin/auth/login`, the system verifies the email and bcrypt-hashed password, then issues a JWT access token and refresh token on success.

**Validation:**
- Invalid email or password → `401 INVALID_CREDENTIALS`
- Rate limit exceeded → `429 RATE_LIMIT_EXCEEDED`

**API Endpoint(s):**

##### POST /api/v1/admin/auth/login

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-01 | IF user logs in via `admin.sirius-skool.com` THEN authenticate against `platform_admins` |
| BR-AUTH-12 | IF token_version embedded in the JWT does not match users.token_version in the database THEN the request is rejected with 401 FORBIDDEN and the user must re-authenticate |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-AUTH-01 | Access token expiry — 15 minutes or configurable? — Closed: Hardcoded 15 min for MVP |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-AUTH-02: Tenant-Scoped Authentication

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-02 |
| **Description** | System shall authenticate School Admin/Manager using email + password against `users` table, scoped by tenant subdomain |
| **Priority** | P0 |
| **Preconditions** | Tenant is active, user is active |
| **Trigger** | POST /api/v1/auth/login |

**Explanation:**

**FR-AUTH-02** authenticates School Admin and Manager credentials against the `users` table, scoped to the tenant identified by the subdomain in the HTTP Host header. The tenant is resolved from the URL (e.g., `springfield.sirius-skool.com` → tenant with subdomain `springfield`), and credentials are checked only within that tenant's user records.

**FR-AUTH-02.1 Tenant-Scoped Authentication — Detailed Breakdown**

**FR-AUTH-02** is the main authentication flow for tenant users (School Admin and Manager). Unlike Platform Admin login (FR-AUTH-01) against the `platform_admins` table, this authenticates against the `users` table and is scoped to a specific tenant via subdomain.

**How it works:**
1. **Tenant resolution** — Subdomain is extracted from the Host header (FR-AUTH-03). `springfield.sirius-skool.com` → tenant slug `springfield`.
2. **Credential check** — System looks up the user by email in `users` where `tenant_id` matches the resolved tenant.
3. **Password verification** — bcrypt comparison against `password_hash`.
4. **Active checks** — Both `tenants.is_active` and `users.is_active` must be `true`.

**What happens on success:**
- JWT access token (15 min) + refresh token (7 days) issued
- Response includes user profile, tenant info, and role (`SCHOOL_ADMIN` or `MANAGER`)
- User is redirected to their role-specific dashboard (FR-AUTH-05)

**Validation:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 403 | `ACCOUNT_INACTIVE` | `users.is_active = false` |
| 403 | `TENANT_INACTIVE` | `tenants.is_active = false` |
| 429 | `RATE_LIMIT_EXCEEDED` | >5 failed attempts/min from same IP |

**Key differences from FR-AUTH-01 (Platform Admin login):**

| Aspect | FR-AUTH-01 (Platform Admin) | FR-AUTH-02 (School Admin/Manager) |
|--------|----------------------------|-----------------------------------|
| **Auth table** | `platform_admins` | `users` |
| **Login URL** | `admin.sirius-skool.com` | `{tenant}.sirius-skool.com` |
| **Tenant scoping** | None (platform-wide) | Scoped by subdomain |
| **Active checks** | None specified | Both user + tenant must be active |

**Edge cases:**
- Correct email/password but deactivated user → `403 ACCOUNT_INACTIVE` (not 401 — this distinguishes from wrong credentials)
- Login to a deactivated tenant → `403 TENANT_INACTIVE` — blocks all users in that school
- Same email in two different tenants → each tenant's users are fully isolated; login only resolves within the subdomain's tenant

**API Endpoint(s):**

##### POST /api/v1/auth/login

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
  },
  "tenant": {
    "is_active": true
  }
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 403 | `ACCOUNT_INACTIVE` | User is deactivated |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many attempts |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-02 | IF user logs in via `{tenant}.sirius-skool.com` THEN authenticate against `users` scoped to that tenant |
| BR-AUTH-14 | IF a tenant user exists in Tenant A THEN they cannot authenticate via Tenant B's subdomain — users.tenant_id scopes all authentication |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-AUTH-03: Tenant Resolution

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-03 |
| **Description** | System shall extract tenant from subdomain in Host header (with X-Tenant-Slug fallback for dev) |
| **Priority** | P0 |
| **Preconditions** | Login page loaded |
| **Trigger** | Before authentication |

**Explanation:**

**FR-AUTH-03** extracts the tenant identifier from the HTTP Host header before authentication occurs. For example, a request to `springfield.sirius-skool.com` extracts `springfield` as the subdomain, which is used to look up the `tenants` record. The `X-Tenant-Slug` header provides a fallback for local development environments.

**FR-AUTH-03.1 Tenant Resolution — Detailed Breakdown**

**FR-AUTH-03** resolves which tenant the user belongs to **before** authentication happens. The system needs the tenant context first so it knows which `users` table records to check.

**How it works:**
1. User navigates to `https://springfield.sirius-skool.com/login`
2. Browser sends HTTP request with `Host: springfield.sirius-skool.com`
3. System parses the Host header and extracts `springfield` as the subdomain
4. Looks up the `tenants` table for a row with `subdomain = 'springfield'`
5. The resolved `tenant_id` is injected into the request context for downstream middleware and controllers

**Why two methods?**

| Method | Source | Environment | Example |
|--------|--------|-------------|---------|
| **Host header** | URL hostname | Production | `Host: springfield.sirius-skool.com` → `springfield` |
| **X-Tenant-Slug** | Custom header | Local development | `X-Tenant-Slug: springfield` |

In local development (`localhost:5173`), there's no subdomain in the URL. The `X-Tenant-Slug` header allows developers to specify the tenant directly for testing without needing DNS or subdomain configuration.

**When it runs:**
- Before authentication (FR-AUTH-02) — the tenant context is a prerequisite
- Before any tenant-scoped API operation
- As Express middleware that parses the header and attaches the tenant to `req.tenant`

**Validation & fallback:**
1. Parse `Host` header → extract subdomain
2. If no valid subdomain → fall back to `X-Tenant-Slug` header
3. If neither yields a tenant → `404` or authentication cannot proceed
4. If tenant found but inactive → handled by FR-AUTH-11 (`403 TENANT_INACTIVE`)

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-AUTH-04: Token Issuance

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-04 |
| **Description** | System shall issue JWT access token (15 min) + refresh token (UUID, 7 days) on successful login |
| **Priority** | P0 |
| **Preconditions** | Credentials valid |
| **Trigger** | After authentication |

**Explanation:**

**FR-AUTH-04** issues a JWT access token (15-minute expiry) and a UUID refresh token (7-day expiry) on successful authentication. The access token is sent on every API request for authorization. The refresh token enables session extension without requiring the user to re-enter credentials.

**FR-AUTH-04.1 Token Issuance — Detailed Breakdown**

**FR-AUTH-04** issues two tokens on successful login: a short-lived **access token** for API authorization and a longer-lived **refresh token** for session extension.

**Token pair:**

| Token | Type | Lifetime | Storage | Purpose |
|-------|------|----------|---------|---------|
| **Access token** | JWT | 15 minutes | Client (in-memory/header) | Sent on every API request in `Authorization: Bearer <token>` header. Contains user ID, tenant ID, role, and token version. |
| **Refresh token** | UUID v4 | 7 days | Client + Server (DB or signed JWT) | Used only at `POST /api/v1/auth/refresh` to get a new access token when the current one expires. |

**Why two tokens?**

| Benefit | Explanation |
|----------|-------------|
| **Security** | Short-lived access tokens limit damage if a token is leaked. A stolen 15-min token is far less risky than a stolen 7-day token. |
| **UX** | Users don't need to re-login every 15 minutes. The refresh token handles silent background renewal. |
| **Revocation** | Access tokens can't be revoked (stateless JWT). Refresh tokens can be revoked server-side, allowing session termination. |

**What happens on login:**
1. Credentials validated (FR-AUTH-01 or FR-AUTH-02)
2. Access token signed with user claims: `{ sub: user_id, tenant_id, role, token_version, iat, exp }`
3. Refresh token (UUID) generated and stored server-side
4. Both tokens returned in the response body

**Response:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "uuid",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": { ... }
}
```

Refresh tokens are stored in the `refresh_tokens` DB table. This enables reuse detection (BR-AUTH-06) — if a revoked token is presented, all tokens for that user are revoked. The `RefreshToken` model is defined in `docs/DB/schema.prisma` and documented in DB Dictionary §Table 6.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-11 | IF user is authenticated THEN access token must have been issued within the last 15 minutes. IF user calls POST /api/v1/auth/refresh THEN refresh token must have been issued within the last 7 days |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-AUTH-02 | Refresh token storage — add `refresh_tokens` model to Prisma, or use signed JWTs without DB lookup? — Closed: DB-backed `refresh_tokens` model added to Prisma and DB Dictionary |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-AUTH-05: Role-Based Redirection

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-05 |
| **Description** | System shall redirect to role-specific dashboard after login |
| **Priority** | P0 |
| **Preconditions** | Login succeeds |
| **Trigger** | After token issuance |

**Explanation:**

**FR-AUTH-05** redirects users to their role-specific dashboard after successful login. Platform Admin → `/admin/dashboard`. School Admin and Manager → their tenant dashboard (`/dashboard`), with the Manager view scoped to their assigned permissions.

**FR-AUTH-05.1 Role-Based Redirection — Detailed Breakdown**

**FR-AUTH-05** handles post-login navigation — after tokens are issued (FR-AUTH-04), the system redirects the user to the appropriate dashboard based on their role.

**Role-based routing:**

| Role | Redirect Destination | What they see |
|------|---------------------|---------------|
| **Platform Admin** | `/admin/dashboard` | Platform-wide overview — all tenants, system health, SMS monitoring across schools |
| **School Admin** | `/dashboard` | Full school dashboard — student count, attendance %, SMS balance, recent notices, quick actions |
| **Manager** | `/dashboard` | Permission-scoped dashboard — only widgets for modules they have View access to (FR-DSH-02). A teacher with only attendance permission sees attendance widgets; a teacher with student + attendance sees both. |

**How it works:**
1. Login successful → tokens issued → response includes user `role`
2. Frontend reads the `role` from the login response body
3. Frontend router redirects to the role-specific path
4. Dashboard component renders widgets based on permissions (Manager) or full view (School Admin/Platform Admin)

**Why per-role dashboards?**
- **Platform Admin** needs cross-tenant visibility, not school-level metrics
- **School Admin** needs full operational view — attendance %, student count, SMS balance, quick actions
- **Manager** only sees authorized modules — a teacher managing attendance shouldn't see SMS balance or system settings

**Implementation responsibility:**
- **Backend** — returns `role` in login response (`user.role`). No backend redirect needed.
- **Frontend** — reads role, performs the actual redirect, and renders role-appropriate widgets. This is a client-side routing concern.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-AUTH-06: Logout & Token Revocation

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-06 |
| **Description** | System shall revoke refresh token on logout |
| **Priority** | P0 |
| **Preconditions** | User is authenticated |
| **Trigger** | POST /api/v1/auth/logout |

**Explanation:**

**FR-AUTH-06** revokes the refresh token on logout via `POST /api/v1/auth/logout`. The provided refresh token is invalidated in storage, making it unusable for future refresh attempts. The user must re-authenticate to obtain new tokens.

**FR-AUTH-06.1 Logout & Token Revocation — Detailed Breakdown**

**FR-AUTH-06** invalidates the refresh token when a user explicitly logs out, ensuring the session cannot be resumed.

**How it works:**
1. User clicks "Logout" → frontend sends `POST /api/v1/auth/logout` with the current refresh token
2. Server marks the refresh token as revoked in storage (DB table or blacklist)
3. Server returns `200 OK` with `{ "message": "Logged out successfully" }`
4. Frontend discards the access token from memory

**What about the access token?**

| Token | Revoked on logout? | Why |
|-------|-------------------|-----|
| **Refresh token** | Yes — marked revoked in server storage | Prevents future refresh attempts |
| **Access token** | No — stateless JWT can't be revoked server-side | It expires naturally after 15 min. Frontend discards it, so it's effectively dead on the client. |

**Validation:**
- Token not found or already revoked → `401 INVALID_REFRESH_TOKEN`

**API Endpoint(s):**

##### POST /api/v1/auth/logout

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-10 | IF user calls POST /api/v1/auth/logout THEN the provided refresh token is marked as revoked |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-AUTH-07: Token Refresh & Rotation

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-07 |
| **Description** | System shall issue new access token using valid refresh token (rotation + reuse detection) |
| **Priority** | P0 |
| **Preconditions** | Refresh token is valid |
| **Trigger** | POST /api/v1/auth/refresh |

**Explanation:**

**FR-AUTH-07** issues new tokens via `POST /api/v1/auth/refresh` using refresh token rotation with reuse detection.

**FR-AUTH-07.1 Token Refresh, Rotation & Reuse Detection — Detailed Breakdown**

**FR-AUTH-07** extends the session by issuing new tokens when the access token expires. It uses **rotation** (old token revoked, new one issued) and **reuse detection** (stolen token triggers full revocation).

**Normal flow:**
1. Access token expires (~15 min) → frontend calls `POST /api/v1/auth/refresh` with the current refresh token
2. Server validates the refresh token (exists, not revoked, not expired)
3. Server issues a **new** access token + **new** refresh token
4. Server revokes the **old** refresh token
5. Frontend updates stored tokens — session continues seamlessly

**Reuse detection flow (token theft scenario):**

| Step | Attacker | Legitimate User |
|------|----------|-----------------|
| 1 | Steals refresh token (e.g., from localStorage XSS) | Unaware |
| 2 | Uses stolen token → gets new tokens (step 3 normal flow) | Still has the old, now-revoked token |
| 3 | Attacker's session works | Tries to use old token → server detects it's already revoked |
| 4 | — | Server sees **presentation of a revoked token** → treats this as theft evidence |
| 5 | Attacker's next refresh fails | **All refresh tokens for the user revoked** — both sessions killed |
| 6 | Attacker loses access | User must re-authenticate with credentials |

**Validation:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `INVALID_REFRESH_TOKEN` | Token not found or already revoked (first use after rotation) |
| 401 | `TOKEN_EXPIRED` | Token lifetime (7 days) exceeded |
| 401 | `TOKEN_REUSE_DETECTED` | Revoked token presented — all tokens for user revoked |

**Why rotation + reuse detection?**

| Mechanism | Purpose |
|-----------|---------|
| **Rotation** | Each refresh token is single-use. Even if stolen, it can only be used once before becoming invalid. |
| **Reuse detection** | If a revoked token is used, it signals theft. All tokens are revoked, forcing the attacker out. |
| **Combined effect** | Limits the window of a stolen token and provides automatic defense against token theft. |

This is the recommended pattern from OAuth2 security best practices (RFC 6749 / RFC 6819).

**API Endpoint(s):**

##### POST /api/v1/auth/refresh

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-06 | IF revoked refresh token is presented THEN revoke ALL refresh tokens for that user (reuse detection) |
| BR-AUTH-11 | IF user is authenticated THEN access token must have been issued within the last 15 minutes. IF user calls POST /api/v1/auth/refresh THEN refresh token must have been issued within the last 7 days |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **8.** FR-AUTH-08: Change Own Password

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-08 |
| **Description** | System shall allow School Admin and Platform Admin to change their own password (requires current password). Manager cannot change password. |
| **Priority** | P0 |
| **Preconditions** | User is authenticated |
| **Trigger** | POST /api/v1/auth/change-password |

**Explanation:**

**FR-AUTH-08** allows School Admin and Platform Admin to change their own password via `POST /api/v1/auth/change-password`. The user provides their current password and a new password. The system verifies the current password before updating. Manager role is blocked from using this endpoint.

**Who can use it:**

| Role | Can change own password? |
|------|-------------------------|
| Platform Admin | Yes |
| School Admin | Yes |
| Manager | No — must contact School Admin for password reset |

**Validation:**
- Current password incorrect → `400 INCORRECT_CURRENT_PASSWORD`
- New password same as current → `400 SAME_PASSWORD`
- Manager attempts → `403 MANAGER_CANNOT_CHANGE_PASSWORD`
- Password does not meet policy (min 8 chars, one uppercase, one lowercase, one digit) → `422 VALIDATION_ERROR`

**API Endpoint(s):**

##### POST /api/v1/auth/change-password

Change own password (School Admin and Platform Admin only — Manager cannot change password).

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
| 403 | `MANAGER_CANNOT_CHANGE_PASSWORD` | Manager role cannot change password |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-05 | IF user role is MANAGER THEN POST /api/v1/auth/change-password returns 403 MANAGER_CANNOT_CHANGE_PASSWORD. IF user role is SCHOOL_ADMIN or PLATFORM_ADMIN THEN current_password must match the stored password_hash |
| BR-AUTH-13 | IF a created or updated password does not meet policy (min 8 chars, at least one uppercase letter, one lowercase letter, one digit) THEN return 422 VALIDATION_ERROR |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **9.** FR-AUTH-09: User Profile & Permissions

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-09 |
| **Description** | System shall return current user profile + permissions |
| **Priority** | P0 |
| **Preconditions** | Valid access token |
| **Trigger** | GET /api/v1/auth/me |

**Explanation:**

**FR-AUTH-09** returns the current authenticated user's profile, tenant information, and effective permissions via `GET /api/v1/auth/me`. Platform Admin sees platform-level data. School Admin sees their tenant info plus all enabled module permissions. Manager sees their tenant info plus their assigned module-level permissions.

**FR-AUTH-09.1 User Profile & Permissions — Detailed Breakdown**

**FR-AUTH-09** provides the authenticated user's identity, tenant context, and granular permissions after login. The frontend calls this endpoint on page load to render the correct UI — which sidebar items to show, which buttons to display, etc.

**Why a dedicated endpoint?**

Instead of embedding all user/tenant/permission data in the login response (which would make login responses bulky), the login endpoint returns minimal user info plus tokens. The frontend then calls `GET /api/v1/auth/me` to get the full profile. This keeps the login response lightweight and allows the profile to be refreshed without re-authentication.

**Per-role response structure:**

| Role | Permission Data |
|------|-----------------|
| **Platform Admin** | No module permissions — platform-level access is implicit. Response includes admin profile fields. |
| **School Admin** | All modules assigned by Platform Admin (from `tenant_modules`) with implicit full action permissions (View/Create/Edit/Delete) across all modules. |
| **Manager** | Only explicitly assigned module-level permissions (from `manager_permissions`). Modules not assigned are excluded. |

**Response shape:**
```json
{
  "user": {
    "id": "uuid",
    "name": "string",
    "email": "string",
    "role": "SCHOOL_ADMIN | MANAGER",
    "is_active": true
  },
  "tenant": {
    "id": "uuid",
    "name": "string",
    "subdomain": "string",
    "is_active": true,
    "settings": { ... }
  },
  "permissions": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT"]
}
```

**Validation & errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `UNAUTHORIZED` | No valid access token provided |
| 401 | `TOKEN_EXPIRED` | Access token has expired |

**API Endpoint(s):**

##### GET /api/v1/auth/me

Current user profile + permissions.

**Response `200 OK` (School Admin):**
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
  "permissions": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT", "RESULT_MANAGEMENT"]
}
```

**Response `200 OK` (Manager):**
```json
{
  "id": "uuid",
  "email": "jane@school.com",
  "full_name": "Jane Smith",
  "role": "MANAGER",
  "tenant": {
    "id": "uuid",
    "name": "Springfield School",
    "slug": "springfield",
    "is_active": true
  },
  "permissions": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT"]
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 401 | `UNAUTHORIZED` | No valid access token provided |
| 401 | `TOKEN_EXPIRED` | Access token has expired |

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **10.** FR-AUTH-10: Inactive User Login Block

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-10 |
| **Description** | System shall block login for inactive users |
| **Priority** | P0 |
| **Preconditions** | `users.is_active = false` |
| **Trigger** | Login attempt |

**Explanation:**

**FR-AUTH-10** blocks login for deactivated users. When `users.is_active = false`, the login endpoint returns `403 ACCOUNT_INACTIVE` regardless of whether the password is correct. This applies to both School Admin and Manager accounts.

**FR-AUTH-10.1 Inactive User Login Block — Detailed Breakdown**

**FR-AUTH-10** prevents deactivated users from authenticating. When a School Admin (via FR-UP-02) or Platform Admin (via FR-PLT-11) deactivates a user, the `users.is_active` field is set to `false`, and all subsequent login attempts are blocked.

**How it works:**
1. User submits credentials to `POST /api/v1/auth/login`
2. System looks up the user by email in the `users` table (scoped by tenant)
3. User is found → system checks `users.is_active`
4. If `false` → return `403 ACCOUNT_INACTIVE` **before** password verification
5. If `true` → proceed with password verification

**Why `403` instead of `401`?**

| Status | Meaning | Use Case |
|--------|---------|----------|
| `401 INVALID_CREDENTIALS` | Wrong email or password | Ambiguous — doesn't reveal whether the email exists or the password was wrong |
| `403 ACCOUNT_INACTIVE` | Credentials correct but account disabled | Tells the user their account has been deactivated so they can contact their School Admin |

**Edge cases:**
- User is deactivated mid-session → existing JWT access tokens remain valid until expiry (15 min max). Refresh tokens are revoked on the next refresh attempt (FR-AUTH-07).
- Reactivated user → can log in again immediately. No password reset required unless explicitly changed.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-03 | IF user.is_active = false THEN return 403 ACCOUNT_INACTIVE |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **11.** FR-AUTH-11: Inactive Tenant Access

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-11 |
| **Description** | System shall allow login for inactive tenants but return 403 FORBIDDEN on all subsequent API requests. Frontend displays deactivation info page. |
| **Priority** | P0 |
| **Preconditions** | `tenants.is_active = false` |
| **Trigger** | Login response |

**Explanation:**

**FR-AUTH-11** allows login for deactivated tenants but returns `403 FORBIDDEN` on all subsequent API requests. The login response includes `tenant.is_active = false` so the frontend can render a deactivation info page.

**FR-AUTH-11.1 Inactive Tenant — Login & Deactivation Page — Detailed Breakdown**

**FR-AUTH-11** handles the deactivated tenant experience at the authentication layer. Unlike the previous design (block login entirely), the system now **allows login** but immediately informs the frontend that the tenant is deactivated. The frontend renders a deactivation info page instead of the dashboard.

**Why allow login instead of blocking it?**

| Reason | Explanation |
|--------|-------------|
| **User clarity** | Users can confirm their credentials work. They see a clear message rather than being silently locked out with a generic error. |
| **Contact info** | The deactivation page provides the reactivation contact email (`admin@support.com`), giving users a path to resolution. |
| **Security preserved** | Although login succeeds, all API operations are blocked by middleware. No data can be accessed or modified. |
| **Seamless reactivation** | When the Platform Admin reactivates the tenant, the user's session is already valid — no re-login needed. |

**How it works (check order matters):**

```
Login request → Resolve tenant from subdomain (FR-AUTH-03)
             → Is tenant active?
                  ├── NO  → Proceed with authentication
                  │          → Is user active? (FR-AUTH-10)
                  │               ├── NO  → 403 ACCOUNT_INACTIVE (stop)
                  │               └── YES → Authenticate → Issue tokens
                  │                        → Response includes tenant.is_active = false
                  │                        → Frontend shows deactivation page
                  └── YES → Is user active? (FR-AUTH-10)
                              ├── NO  → 403 ACCOUNT_INACTIVE (stop)
                              └── YES → Authenticate → Issue tokens
                                       → Normal dashboard
```

**Frontend behavior:**

| Step | Action |
|------|--------|
| 1 | Submit login form → wait for response |
| 2 | Response contains `tenant.is_active = false` |
| 3 | Store tokens (needed for potential reactivation) |
| 4 | Redirect to `/deactivated` page instead of dashboard |
| 5 | Deactivation page shows the message and a "Try Again" button (which re-fetches `/api/v1/auth/me` to check if tenant has been reactivated) |

**Edge cases:**

| Scenario | Behavior |
|----------|----------|
| Tenant deactivated mid-session | Next API request middleware check detects `tenants.is_active = false` and returns `403 TENANT_INACTIVE`. Frontend redirects to deactivation page. |
| Reactivated tenant | User on the deactivation page clicks "Try Again" → `/api/v1/auth/me` returns `tenant.is_active = true` → frontend redirects to dashboard. |
| Reactivated tenant (new login) | Login proceeds normally — `tenant.is_active = true` in response — normal dashboard. |
| User deactivated within deactivated tenant | Authentication fails with `403 ACCOUNT_INACTIVE` (user-level check still runs before issuing tokens). |
| Platform Admin login | Not affected — Platform Admin authenticates against `platform_admins`, not the `users` table. |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-04 | IF tenant.is_active = false THEN login proceeds but returns tenant.is_active=false in the response. All subsequent API requests return 403 TENANT_INACTIVE. |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **12.** FR-AUTH-12: Rate Limiting

| Property | Value |
|----------|-------|
| **ID** | FR-AUTH-12 |
| **Description** | System shall rate-limit login attempts (5 failed per minute per IP) |
| **Priority** | P1 |
| **Preconditions** | Rate exceeded |
| **Trigger** | Login attempt |

**Explanation:**

**FR-AUTH-12** implements rate limiting on the login endpoint — 5 failed attempts per minute per IP address. When exceeded, the endpoint returns `429 RATE_LIMIT_EXCEEDED` and additional attempts are blocked until the window resets. This mitigates brute-force attacks.

**FR-AUTH-12.1 Rate Limiting — Detailed Breakdown**

**FR-AUTH-12** protects the login endpoint from brute-force and password-spraying attacks by limiting failed attempts per IP address.

**Limit window:**

| Parameter | Value |
|-----------|-------|
| Max failed attempts | 5 |
| Window duration | 60 seconds (rolling) |
| Scope | Per IP address |
| Reset | Successful login or window expiry |

**How it works:**
1. On each failed login attempt, the system increments a counter keyed by `ip_address`
2. If the counter exceeds 5 within the rolling 60-second window, all subsequent requests from that IP return `429 RATE_LIMIT_EXCEEDED` without checking credentials
3. After 60 seconds with no failed attempts, the counter resets to 0
4. A successful login from that IP resets the counter immediately

**What it protects against:**

| Attack | Description | How rate limiting helps |
|--------|-------------|------------------------|
| **Brute force** | Many passwords for one email | Each IP can only try 5 passwords/min per email; slows to 300 attempts/hour per IP |
| **Password spraying** | One common password across many emails | Same limit applies — can't spray efficiently from a single IP |
| **Distributed attack** | Many IPs, few attempts each | Not fully mitigated by IP-based limiting alone. Consider adding account lockout after N failed attempts across all IPs for a future iteration (not in MVP). |

**MVP implementation:** Use an in-memory store (e.g., `express-rate-limit` default memory store). This is sufficient for single-instance deployments.

**Production upgrade path:** When scaling to multiple server instances, replace the in-memory store with a Redis-backed store so rate limit state is shared across all instances.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUTH-07 | IF login fails >5 times in 1 minute from same IP THEN rate-limit (429) |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-AUTH-03 | Single session per user — enforced in MVP or optional? *(Closed: optional — token_version mechanism supports future enforcement)* |

---

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

---

### 3.3 Academic Structure

**Quick Summary:** This module defines the academic hierarchy that every other module depends on. School Admin creates academic years (e.g., `2026-2027`), classes (e.g., `Class 6`), shifts (e.g., `Morning`, `Day`, `Evening`), sections (e.g., `Section A`), and subjects (e.g., `Mathematics` per class). These entities are permanent and reused across academic years. Student enrollments link students to this structure per year.

> **Data Model:** Full schema in DB Dictionary §Table 7 (academic_years), §Table 8 (classes), §Table 9 (shifts), §Table 10 (sections), §Table 11 (subjects).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-ACA-01 | Create Academic Year | School Admin shall create academic years with name, start date, end date | P0 | Authenticated as School Admin | POST /api/v1/academic-years |
| 2 | FR-ACA-02 | Single Current Academic Year | System shall enforce exactly one active academic year per tenant | P0 | Academic years exist | Setting `is_current = true` |
| 3 | FR-ACA-03 | Prevent Overlapping Date Ranges | System shall prevent overlapping academic year date ranges | P0 | Creating or updating | Validation |
| 4 | FR-ACA-04 | Create Class | School Admin shall create classes with code, name, display_order | P0 | Authenticated | POST /api/v1/classes |
| 5 | FR-ACA-05 | Create Shift | School Admin shall create shifts within a class (e.g., Morning, Day, Evening) | P0 | Class exists | POST /api/v1/classes/{id}/shifts |
| 6 | FR-ACA-06 | Create Section | School Admin shall create sections within a shift | P0 | Shift exists | POST /api/v1/shifts/{id}/sections |
| 7 | FR-ACA-07 | Create Subject | School Admin shall create subjects per class | P0 | Class exists | POST /api/v1/classes/{id}/subjects |
| 8 | FR-ACA-08 | Set Current Academic Year | School Admin shall set one academic year as current | P0 | Academic year exists | PATCH /api/v1/academic-years/{id} |
| 9 | FR-ACA-09 | List Academic Entities | System shall provide list endpoints for all academic entities (dropdowns) | P0 | Entity exists | GET endpoints |
| 10 | FR-ACA-10 | Close/Reopen Academic Year | School Admin shall close an academic year (read-only) or reopen it | P0 | Academic year exists | POST /api/v1/academic-years/{id}/close |

---





#### **1.** FR-ACA-01: Create Academic Year

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-01 |
| **Description** | School Admin shall create academic years with name, start date, end date |
| **Priority** | P0 |
| **Preconditions** | Authenticated as School Admin |
| **Trigger** | POST /api/v1/academic-years |

**Explanation:**

**FR-ACA-01** allows School Admin to create academic years (e.g., "2026-2027") with a start date and end date via `POST /api/v1/academic-years`. Academic years are the top-level entity that scopes enrollments, attendance, exams, and other time-bound data.

**What happens:**
- A new `academic_years` row is created with the provided name and dates
- If `is_current` is set to `true`, the system ensures no other academic year is current (FR-ACA-02)

**Validation:**
- Duplicate name → `409 DUPLICATE_NAME`
- Overlapping dates → `409 OVERLAPPING_DATES`
- Another year already current → `400 CURRENT_YEAR_EXISTS`

**API Endpoint(s):**

##### POST /api/v1/academic-years

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-01 | A tenant can have only one current academic year at a time |
| BR-ACA-02 | Academic year names must be unique per tenant |
| BR-ACA-03 | Academic year date ranges must not overlap |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-ACA-02: Single Current Academic Year

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-02 |
| **Description** | System shall enforce exactly one active academic year per tenant |
| **Priority** | P0 |
| **Preconditions** | Academic years exist |
| **Trigger** | Setting `is_current = true` |

**Explanation:**

**FR-ACA-02** enforces that a tenant has exactly one active (current) academic year at any time. When a School Admin sets `is_current = true` on an academic year, all other years for that tenant are automatically set to `is_current = false`. This ensures that dashboards, reports, and data queries have an unambiguous current context.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-01 | A tenant can have only one current academic year at a time |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-ACA-03: Prevent Overlapping Date Ranges

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-03 |
| **Description** | System shall prevent overlapping academic year date ranges |
| **Priority** | P0 |
| **Preconditions** | Creating or updating |
| **Trigger** | Validation |

**Explanation:**

**FR-ACA-03** prevents academic year date ranges from overlapping. The system validates start_date and end_date against all existing academic years for the tenant. Overlapping years would create ambiguity for enrollment and attendance recording.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-03 | Academic year date ranges must not overlap |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-ACA-04: Create Class

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-04 |
| **Description** | School Admin shall create classes with code, name, display_order |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | POST /api/v1/classes |

**Explanation:**

**FR-ACA-04** allows School Admin to create classes (e.g., "Class 6", "Class 10") with a short code, display name, and sort order via `POST /api/v1/classes`. Classes are permanent entities reused across academic years — they are soft-disabled rather than deleted (BR-ACA-04).

**API Endpoint(s):**

##### POST /api/v1/classes

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-04 | Classes, shifts, sections, and subjects use `is_active` to disable (never delete) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-ACA-05: Create Shift

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-05 |
| **Description** | School Admin shall create shifts within a class (e.g., Morning, Day, Evening) |
| **Priority** | P0 |
| **Preconditions** | Class exists |
| **Trigger** | POST /api/v1/classes/{id}/shifts |

**Explanation:**

**FR-ACA-05** allows School Admin to create shifts within a class (e.g., "Morning Shift", "Day Shift", "Evening Shift") via `POST /api/v1/classes/{classId}/shifts`. A shift is a time-based grouping within a class — each shift can have multiple sections. Shifts are permanent entities reused across academic years; they are soft-disabled rather than deleted (BR-ACA-04).

**API Endpoint(s):**

##### POST /api/v1/classes/{classId}/shifts

**Request:**
```json
{
  "name": "Morning",
  "display_order": 1
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "class_id": "uuid",
  "name": "Morning",
  "display_order": 1
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-11 | A shift belongs to exactly one class |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━



#### **6.** FR-ACA-06: Create Section

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-06 |
| **Description** | School Admin shall create sections within a shift |
| **Priority** | P0 |
| **Preconditions** | Shift exists |
| **Trigger** | POST /api/v1/shifts/{id}/sections |

**Explanation:**

**FR-ACA-06** allows School Admin to create sections within a shift (e.g., "Section A", "Section B") via `POST /api/v1/shifts/{shiftId}/sections`. Each section belongs to exactly one shift (BR-ACA-05). Sections are where student enrollments and attendance are tracked.

**API Endpoint(s):**

##### POST /api/v1/shifts/{shiftId}/sections

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
  "shift_id": "uuid",
  "code": "A",
  "name": "Section A"
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-05 | A section belongs to exactly one shift |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-ACA-07: Create Subject

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-07 |
| **Description** | School Admin shall create subjects per class |
| **Priority** | P0 |
| **Preconditions** | Class exists |
| **Trigger** | POST /api/v1/classes/{id}/subjects |

**Explanation:**

**FR-ACA-07** allows School Admin to create subjects under a class (e.g., "Mathematics" for Class 6) via `POST /api/v1/classes/{classId}/subjects`. Subject codes must be unique within a class (BR-ACA-08). Subject types include `MANDATORY` and `OPTIONAL`. These subjects are later referenced when configuring exams and marks.

**API Endpoint(s):**

##### POST /api/v1/classes/{classId}/subjects

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-06 | A subject belongs to exactly one class |
| BR-ACA-07 | Subject codes must be unique within a class |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **8.** FR-ACA-08: Set Current Academic Year

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-08 |
| **Description** | School Admin shall set one academic year as current |
| **Priority** | P0 |
| **Preconditions** | Academic year exists |
| **Trigger** | PATCH /api/v1/academic-years/{id} |

**Explanation:**

**FR-ACA-08** allows School Admin to set an academic year as the current year via `PATCH /api/v1/academic-years/{id}` with `{ "is_current": true }`. The system automatically unmarks the previously current year, ensuring exactly one current year per tenant at all times. Additionally, setting a new current year resets the tenant's student registration counter (`current_student_sequence`) back to its `starting_sequence` value, ensuring registration numbers start fresh each academic year (see FR-PLT-01.3).

**API Endpoint(s):**

##### PATCH /api/v1/academic-years/{id}

Set an academic year as current.

**Request:**
```json
{
  "is_current": true
}
```

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "is_current": true
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `NOT_FOUND` | Academic year not found |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ACA-01 | A tenant can have only one current academic year at a time |
| BR-ACA-08 | Setting a new current academic year resets `current_student_sequence` to `starting_sequence` |
| BR-ACA-09 | A closed academic year is read-only — no data edits allowed, only reports viewing |
| BR-ACA-10 | Closing an academic year requires all attendance and results for that year to be finalized |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **9.** FR-ACA-09: List Academic Entities

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-09 |
| **Description** | System shall provide list endpoints for all academic entities (dropdowns) |
| **Priority** | P0 |
| **Preconditions** | Entity exists |
| **Trigger** | GET endpoints |

**Explanation:**

**FR-ACA-09** provides GET list endpoints for all academic entities (academic years, classes, shifts, sections, subjects). These are used to populate dropdown selectors throughout the UI — e.g., selecting a class and shift when taking attendance or creating an exam.

**API Endpoint(s):**

##### GET /api/v1/academic-years

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

##### GET /api/v1/classes/{classId}/shifts

List shifts within a class.

**Response `200 OK`:**
```json
{
  "data": [
    { "id": "uuid", "class_id": "uuid", "name": "Morning", "display_order": 1 },
    { "id": "uuid", "class_id": "uuid", "name": "Day", "display_order": 2 }
  ]
}
```

**No related business rules or open questions.**

---




#### **10.** FR-ACA-10: Close/Reopen Academic Year

| Property | Value |
|----------|-------|
| **ID** | FR-ACA-10 |
| **Description** | School Admin shall close an academic year (read-only) or reopen it |
| **Priority** | P0 |
| **Preconditions** | Academic year exists |
| **Trigger** | POST /api/v1/academic-years/{id}/close, POST /api/v1/academic-years/{id}/reopen |

**Explanation:**

**FR-ACA-10** allows School Admin to close an academic year via `POST /api/v1/academic-years/{id}/close`, making it read-only. Once closed, no attendance, results, or enrollment edits are permitted for that year — only report viewing is allowed (BR-ACA-09). All attendance and results must be finalized before closing (BR-ACA-10). A closed year can be reopened via `POST /api/v1/academic-years/{id}/reopen` if corrections are needed.

**API Endpoint(s):**

##### POST /api/v1/academic-years/{id}/close

Close an academic year.

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "status": "CLOSED"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `HAS_PENDING_DATA` | Attendance or results not finalized |

##### POST /api/v1/academic-years/{id}/reopen

Reopen a closed academic year.

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "status": "ACTIVE"
}
```

**No related business rules or open questions.**

---

#### Open Questions

| # | Question |
|---|----------|
| Q-ACA-01 | Close endpoint or just is_current toggle? |
| Q-ACA-02 | Verify no pending data before closing year? |

---

### 3.4 Settings

**Quick Summary:** School Admin configures their school's branding (logo, address, phone), academic preferences, and general settings. This is a CRUD module — a single `tenant_settings` record per tenant holds all configurable values. No APIs beyond basic GET/PATCH are needed.

> **Data Model:** Single row per tenant in DB Dictionary §Table 3.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-SET-01 | View Settings | School Admin shall view their tenant settings | P0 | Authenticated as School Admin | GET /api/v1/settings |
| 2 | FR-SET-02 | Update Settings | School Admin shall update school information including branding, school code, notification settings, and localization | P0 | Authenticated | PATCH /api/v1/settings |
| 3 | FR-SET-03 | Logo Upload | System shall upload logo to Cloudinary and store URL | P0 | File uploaded | During settings update |

---





#### **1.** FR-SET-01: View Settings

| Property | Value |
|----------|-------|
| **ID** | FR-SET-01 |
| **Description** | School Admin shall view their tenant settings |
| **Priority** | P0 |
| **Preconditions** | Authenticated as School Admin |
| **Trigger** | GET /api/v1/settings |

**Explanation:**

**FR-SET-01** allows School Admin to view their tenant's settings via `GET /api/v1/settings`. The response includes the school logo URL, address, phone number, and email — all of which are used in report branding and communication.

**API Endpoint(s):**

##### GET /api/v1/settings

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-SET-02: Update Settings

| Property | Value |
|----------|-------|
| **ID** | FR-SET-02 |
| **Description** | School Admin shall update school information including branding, school code, notification settings, and localization |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | PATCH /api/v1/settings |

**Explanation:**

**FR-SET-02** allows School Admin to update school information and configuration via `PATCH /api/v1/settings`. Fields include logo URL, school code, address, phone, email, website URL, timezone, date/number format, language preference (for future i18n), sender name/ID overrides for notifications, per-trigger notification type toggles with customizable templates, and an attendance month closure toggle (if enabled, past month attendance is read-only). Only the School Admin role can modify settings (BR-SET-01).

**API Endpoint(s):**

##### PATCH /api/v1/settings

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-SET-01 | Only School Admin can modify settings |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-SET-03: Logo Upload

| Property | Value |
|----------|-------|
| **ID** | FR-SET-03 |
| **Description** | System shall upload logo to Cloudinary and store URL |
| **Priority** | P0 |
| **Preconditions** | File uploaded |
| **Trigger** | During settings update |

**Explanation:**

**FR-SET-03** handles logo upload when the School Admin updates settings. When a logo file is uploaded, the system sends it to Cloudinary for storage and CDN delivery. The returned URL is saved in the `tenant_settings` record. Cloudinary handles image optimization and transformations.

**No related business rules or open questions.**

---

#### Open Questions

| # | Question |
|---|----------|
| Q-SET-01 | Where are notification templates stored? DB Dictionary doesn't define a `notification_templates` table yet |
| Q-SET-02 | Should password policy settings (min length, complexity) be stored in settings or stay hardcoded? — Closed: Hardcoded for MVP — min 8 chars, at least one uppercase, one lowercase, one digit (BR-AUTH-13) |

---

### 3.5 User & Permission Management

**Quick Summary:** School Admin creates Manager accounts and assigns module-level permissions (single toggle per module). Managers can only be assigned modules that the School Admin has (from Platform Admin allocation). School Admin has implicit full access to all assigned modules. This module implements the Tier 2 permission model.

> **Data Model:** Full schema in DB Dictionary §Table 6 (manager_permissions).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-UP-01 | Create Manager | School Admin shall create a Manager with name, email, password | P0 | Authenticated as School Admin | POST /api/v1/managers |
| 2 | FR-UP-02 | Activate/Deactivate Manager | School Admin shall activate/deactivate a Manager | P0 | Manager exists | PATCH /api/v1/managers/{id} |
| 3 | FR-UP-03 | List Managers | School Admin shall view list of all Managers with permission summary | P0 | Authenticated | GET /api/v1/managers |
| 4 | FR-UP-04 | Assign Manager Permissions | School Admin shall assign module-level permissions to a Manager (single toggle per module) | P0 | Manager exists, module is enabled | PUT /api/v1/managers/{id}/permissions |
| 5 | FR-UP-05 | Available Modules for Permissions | School Admin shall only see modules assigned by Platform Admin in the permission UI | P0 | Authenticated | GET /api/v1/permissions/available-modules |
| 6 | FR-UP-06 | Immediate Session Invalidation | System shall deactivate a Manager immediately — all sessions invalidated | P0 | Manager deactivated | Token version incremented |
| 7 | FR-UP-07 | Reset Manager Password | School Admin shall reset a Manager's password without current password | P0 | Manager exists, School Admin is authenticated | PATCH /api/v1/managers/{id}/reset-password |
| 8 | FR-UP-08 | Update Manager Profile | School Admin shall update Manager's name, email, phone | P1 | Manager exists | PATCH /api/v1/managers/{id}/profile |

---





#### **1.** FR-UP-01: Create Manager

| Property | Value |
|----------|-------|
| **ID** | FR-UP-01 |
| **Description** | School Admin shall create a Manager with name, email, password |
| **Priority** | P0 |
| **Preconditions** | Authenticated as School Admin |
| **Trigger** | POST /api/v1/managers |

**Explanation:**

**FR-UP-01** allows School Admin to create Manager accounts via `POST /api/v1/managers`. The School Admin provides the Manager's name, email, password, and phone. The user is created in the `users` table with `role = MANAGER` and `is_active = true`.

**Validation:**
- Email already in use → `409 EMAIL_EXISTS`

**API Endpoint(s):**

##### POST /api/v1/managers

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-UP-01 | Only users with role `MANAGER` can have permission records |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-UP-02: Activate/Deactivate Manager

| Property | Value |
|----------|-------|
| **ID** | FR-UP-02 |
| **Description** | School Admin shall activate/deactivate a Manager |
| **Priority** | P0 |
| **Preconditions** | Manager exists |
| **Trigger** | PATCH /api/v1/managers/{id} |

**Explanation:**

**FR-UP-02** allows School Admin to activate or deactivate a Manager via `PATCH /api/v1/managers/{id}`. Deactivation immediately invalidates all the Manager's active sessions by incrementing `token_version` (BR-UP-06), and the Manager cannot log in until reactivated (FR-AUTH-10).

**API Endpoint(s):**

##### PATCH /api/v1/managers/{id}

Activate/deactivate a Manager.

**Request:**
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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-UP-06 | Deactivating a Manager increments `token_version` to invalidate all sessions |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-UP-03: List Managers

| Property | Value |
|----------|-------|
| **ID** | FR-UP-03 |
| **Description** | School Admin shall view list of all Managers with permission summary |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | GET /api/v1/managers |

**Explanation:**

**FR-UP-03** provides the School Admin a list of all Managers with their active status, last login timestamp, and a summary of their assigned permissions via `GET /api/v1/managers`. This is the main user management view.

**API Endpoint(s):**

##### GET /api/v1/managers

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "full_name": "Jane Smith",
      "email": "jane@school.com",
      "is_active": true,
      "permissions": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT"],
      "last_login_at": "2026-07-09T08:00:00Z"
    }
  ]
}
```

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-UP-04: Assign Manager Permissions

| Property | Value |
|----------|-------|
| **ID** | FR-UP-04 |
| **Description** | School Admin shall assign module-level permissions to a Manager (single toggle per module) |
| **Priority** | P0 |
| **Preconditions** | Manager exists, module is enabled |
| **Trigger** | PUT /api/v1/managers/{id}/permissions |

**Explanation:**

**FR-UP-04** allows School Admin to assign module-level permissions to a Manager via `PUT /api/v1/managers/{id}/permissions`. Each module is a **single toggle** — granting a module gives the Manager a predefined set of actions as defined in the PRD:

| Module | Actions Granted |
|--------|----------------|
| STUDENT_MANAGEMENT | View, Create, Edit (V/C/E) |
| ATTENDANCE_MANAGEMENT | View, Create, Edit (V/C/E) |
| RESULT_MANAGEMENT | View, Create, Edit (V/C/E) |
| NOTICE_BOARD | View, Create, Edit (V/C/E) |
| NOTIFICATIONS (SMS Log) | View, Create (V/C) |
| REPORTS | View only |

- **Delete** is always Admin-only — Managers never receive delete access.
- **Print/Export** are Admin-only actions — not assignable to Managers.
- Modules not listed in the request are denied by default (BR-UP-05).

**API Endpoint(s):**

##### PUT /api/v1/managers/{id}/permissions

**Request:**
```json
{
  "permissions": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT"]
}
```

**Response `200 OK`:**
```json
{
  "user_id": "uuid",
  "permissions": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT"]
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-UP-02 | School Admin automatically has full access to all enabled modules |
| BR-UP-03 | School Admin can only delegate modules that are in `tenant_modules` for their tenant |
| BR-UP-04 | IF a module is disabled in `tenant_modules` THEN all related permissions become inactive |
| BR-UP-05 | IF a module is not in the Manager's granted list THEN access to that module is denied |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-UP-02 | Is `can_print` separate from `can_export` in the UI, or are they combined? — Closed: Print/Export are Admin-only actions. Managers use module-level toggle with predefined action sets. |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-UP-05: Available Modules for Permissions

| Property | Value |
|----------|-------|
| **ID** | FR-UP-05 |
| **Description** | School Admin shall only see modules assigned by Platform Admin in the permission UI |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | GET /api/v1/permissions/available-modules |

**Explanation:**

**FR-UP-05** ensures that the School Admin can only delegate modules that were assigned by the Platform Admin. The `GET /api/v1/permissions/available-modules` endpoint returns only the modules present in the tenant's `tenant_modules` table. This enforces the Tier 2 permission model boundary.

**API Endpoint(s):**

##### GET /api/v1/permissions/available-modules

Returns modules available for permission assignment.

**Response `200 OK`:**
```json
{
  "modules": ["STUDENT_MANAGEMENT", "ATTENDANCE_MANAGEMENT", "RESULT_MANAGEMENT"]
}
```

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-UP-06: Immediate Session Invalidation

| Property | Value |
|----------|-------|
| **ID** | FR-UP-06 |
| **Description** | System shall deactivate a Manager immediately — all sessions invalidated |
| **Priority** | P0 |
| **Preconditions** | Manager deactivated |
| **Trigger** | Token version incremented |

**Explanation:**

**FR-UP-06** ensures immediate session invalidation when a Manager is deactivated. Deactivation increments the `token_version` field on the `users` record. All existing JWT tokens (which embed the token version at issuance) become invalid, forcing the Manager to re-authenticate if reactivated.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-UP-06 | Deactivating a Manager increments `token_version` to invalidate all sessions |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-UP-07: Reset Manager Password

| Property | Value |
|----------|-------|
| **ID** | FR-UP-07 |
| **Description** | School Admin shall reset a Manager's password without current password |
| **Priority** | P0 |
| **Preconditions** | Manager exists, School Admin is authenticated |
| **Trigger** | PATCH /api/v1/managers/{id}/reset-password |

**Explanation:**

**FR-UP-07** allows the School Admin to reset a Manager's password without requiring the current password. This is needed when a Manager forgets their password or is locked out.

**How it works:**
1. School Admin calls `PATCH /api/v1/managers/{id}/reset-password` with `{ "new_password": "NewPass123" }`
2. System updates the Manager's `password_hash` in the `users` table
3. System increments `token_version` to invalidate all existing sessions (BR-AUTH-09)
4. Manager must log in again with the new password

**Validation:**
- Manager not found → `404 MANAGER_NOT_FOUND`
- Only School Admin of the same tenant → `403 FORBIDDEN`

**API Endpoint(s):**

##### PATCH /api/v1/managers/{id}/reset-password

Reset a Manager's password (School Admin only).

**Request:**
```json
{
  "new_password": "NewPass123"
}
```

**Response `200 OK`:**
```json
{
  "message": "Password reset successfully",
  "manager_id": "uuid"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `MANAGER_NOT_FOUND` | Manager not found |
| 403 | `FORBIDDEN` | Only School Admin of the same tenant can reset Manager passwords |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-UP-01 | Should School Admin be able to edit a Manager's email and password? *(Closed: Yes — name, email, phone via FR-UP-08; password reset via FR-UP-07)* |

---

#### **8.** FR-UP-08: Update Manager Profile

| Property | Value |
|----------|-------|
| **ID** | FR-UP-08 |
| **Description** | School Admin shall update Manager's name, email, phone |
| **Priority** | P1 |
| **Preconditions** | Manager exists |
| **Trigger** | PATCH /api/v1/managers/{id}/profile |

**Explanation:**

**FR-UP-08** allows School Admin to update a Manager's profile information (name, email, phone) via `PATCH /api/v1/managers/{id}/profile`. Email uniqueness is enforced across the tenant.

**API Endpoint(s):**

##### PATCH /api/v1/managers/{id}/profile

Update Manager's name, email, phone.

**Request:**
```json
{
  "name": "Updated Name",
  "email": "newemail@school.com",
  "phone": "+8801712345679"
}
```

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "name": "Updated Name",
  "email": "newemail@school.com",
  "phone": "+8801712345679"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `MANAGER_NOT_FOUND` | Manager not found |
| 409 | `EMAIL_EXISTS` | Email already in use by another user in the tenant |

**No related business rules or open questions.**

---

### 3.6 Module Management

**Quick Summary:** School Admin can enable or disable modules from the pool assigned by Platform Admin. Disabled modules are hidden from the sidebar and inaccessible to all users in the tenant. Data is preserved when a module is disabled.

> **Data Model:** Full schema in DB Dictionary §Table 4. Module ENUM values defined in `schema.prisma` `ModuleEnum` and DB Dictionary §Module ENUM.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-MM-01 | View Module Status | School Admin shall view all assigned modules with enable/disable status | P0 | Authenticated | GET /api/v1/tenant-modules |
| 2 | FR-MM-02 | Toggle Module | School Admin shall toggle a module on/off | P0 | Module is assigned by Platform Admin | PATCH /api/v1/tenant-modules/{module} |
| 3 | FR-MM-03 | Disable Module Enforcement | System shall hide disabled modules from sidebar and block API access | P0 | Module toggled off | On any request to that module |

---





#### **1.** FR-MM-01: View Module Status

| Property | Value |
|----------|-------|
| **ID** | FR-MM-01 |
| **Description** | School Admin shall view all assigned modules with enable/disable status |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | GET /api/v1/tenant-modules |

**Explanation:**

**FR-MM-01** allows School Admin to view all modules assigned by Platform Admin along with their current enabled/disabled status via `GET /api/v1/tenant-modules`. This is the module management overview.

**API Endpoint(s):**

##### GET /api/v1/tenant-modules

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-MM-02: Toggle Module

| Property | Value |
|----------|-------|
| **ID** | FR-MM-02 |
| **Description** | School Admin shall toggle a module on/off |
| **Priority** | P0 |
| **Preconditions** | Module is assigned by Platform Admin |
| **Trigger** | PATCH /api/v1/tenant-modules/{module} |

**Explanation:**

**FR-MM-02** allows School Admin to toggle a module on or off via `PATCH /api/v1/tenant-modules/{module}`. The School Admin can only toggle modules that were pre-assigned by the Platform Admin (BR-MM-02). Disabling a module hides it from all users in the tenant.

**API Endpoint(s):**

##### PATCH /api/v1/tenant-modules/{module}

**Request:**
```json
{
  "is_enabled": false
}
```

**Response `200 OK`:**
```json
{
  "module": "ATTENDANCE_MANAGEMENT",
  "is_enabled": false
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-MM-01 | Authentication and Settings modules cannot be disabled |
| BR-MM-02 | School Admin cannot enable a module not assigned by Platform Admin |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-MM-03: Disable Module Enforcement

| Property | Value |
|----------|-------|
| **ID** | FR-MM-03 |
| **Description** | System shall hide disabled modules from sidebar and block API access |
| **Priority** | P0 |
| **Preconditions** | Module toggled off |
| **Trigger** | On any request to that module |

**Explanation:**

**FR-MM-03** enforces module disablement at both the UI and API layers. When a module is toggled off, the sidebar hides its navigation items, and all API endpoints belonging to that module return `403 FORBIDDEN`. Existing data is preserved and restored when the module is re-enabled (BR-MM-04).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-MM-03 | Disabling a module hides it for the entire tenant (all roles) |
| BR-MM-04 | Re-enabling restores access to existing data |

---

### 3.7 Student Management & Admission

**Quick Summary:** Covers the complete student lifecycle: admission (public form → review → accept/reject), student CRUD (with soft-delete), enrollment management, promotion, and bulk import/export. Registration numbers follow `YY + sequence` format. Roll numbers are per-section, per-year. Students have a permanent identity (`students` table) separate from their yearly enrollment (`student_enrollments`). Admission applications (`applications`) go through a PENDING → APPROVED/REJECTED workflow.

> **Data Model:** Full schema in DB Dictionary §Table 12 (applications), §Table 13 (students), §Table 14 (student_enrollments).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-STU-01 | Submit Application | System shall accept admission applications with applicant details | P0 | Tenant is active | POST /api/v1/applications |
| 2 | FR-STU-02 | Application Number Generation | System shall generate a unique application number (`APP-{year}-{seq}`) | P0 | Application submitted | After create |
| 3 | FR-STU-03 | View Pending Applications | Manager with admission permission shall view pending applications | P0 | Authenticated | GET /api/v1/applications?status=PENDING |
| 4 | FR-STU-04 | Approve Application | Manager shall approve an application → creates student + enrollment | P0 | Application is PENDING | POST /api/v1/applications/{id}/approve |
| 5 | FR-STU-05 | Reject Application | Manager shall reject an application with reason | P0 | Application is PENDING | POST /api/v1/applications/{id}/reject |
| 6 | FR-STU-06 | Registration Number Generation | System shall auto-generate registration number (`YY + sequence`) on approval in an atomic transaction | P0 | Application approved | During approval |
| 7 | FR-STU-07 | Roll Number Assignment | System shall auto-assign roll number within section (sequential) | P0 | Enrollment created | During approval |
| 8 | FR-STU-08 | Student CRUD | School Admin/Manager shall CRUD student personal info | P0 | Student exists | /api/v1/students endpoints |
| 9 | FR-STU-09 | Student Search | System shall search students by name, roll number, registration number, class | P0 | Students exist | GET /api/v1/students?search=... |
| 10 | FR-STU-10 | Student Promotion | School Admin shall promote students to next class (bulk or individual) | P0 | Academic year exists, previous enrollment exists | POST /api/v1/students/promote |
| 11 | FR-STU-11 | End-of-Year Outcomes | System shall handle end-of-year outcomes: Promote, Repeat, Graduate, Transfer, Dropout | P0 | Academic year ending | POST /api/v1/students/{id}/outcome |
| 12 | FR-STU-12 | Import Students from Excel | School Admin shall import students from Excel | P1 | Authenticated | POST /api/v1/students/import |
| 13 | FR-STU-13 | Export Student List | School Admin shall export student list to Excel/PDF | P1 | Authenticated | GET /api/v1/students/export |
| 14 | FR-STU-14 | Auto-Expire Pending Applications | System shall auto-delete applications in PENDING state after 30 days | P1 | Application is PENDING for 30+ days | Cron/queue job |

---





#### **1.** FR-STU-01: Submit Application

| Property | Value |
|----------|-------|
| **ID** | FR-STU-01 |
| **Description** | System shall accept admission applications with applicant details |
| **Priority** | P0 |
| **Preconditions** | Tenant is active |
| **Trigger** | POST /api/v1/applications |

**Explanation:**

**FR-STU-01** accepts admission applications from prospective students via `POST /api/v1/applications`. The form collects applicant details including name, gender, date of birth, guardian contact, and address. The application is created with `status = PENDING` and awaits Manager review.

**What happens:**
- A unique application number is generated: `APP-{year}-{seq}` (FR-STU-02)
- The application is linked to the target academic year and applying class

**API Endpoint(s):**

##### POST /api/v1/applications

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-STU-02: Application Number Generation

| Property | Value |
|----------|-------|
| **ID** | FR-STU-02 |
| **Description** | System shall generate a unique application number (`APP-{year}-{seq}`) |
| **Priority** | P0 |
| **Preconditions** | Application submitted |
| **Trigger** | After create |

**Explanation:**

**FR-STU-02** generates a unique application number for each admission application. The format is `APP-{year}-{seq}` (e.g., `APP-2026-000001`), where `year` is the 4-digit admission year and `seq` is a zero-padded sequential counter per tenant. This number is the application's reference identifier throughout the admission process.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-STU-01 | Registration number format: `YY` (admission year) + sequence per tenant (resets each academic year) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-STU-03: View Pending Applications

| Property | Value |
|----------|-------|
| **ID** | FR-STU-03 |
| **Description** | Manager with admission permission shall view pending applications |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | GET /api/v1/applications?status=PENDING |

**Explanation:**

**FR-STU-03** allows Managers with admission permission to view pending applications via `GET /api/v1/applications?status=PENDING`. Results are paginated and filterable. This is the admission review queue where Managers can approve or reject applicants.

**API Endpoint(s):**

##### GET /api/v1/applications

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-STU-04: Approve Application

| Property | Value |
|----------|-------|
| **ID** | FR-STU-04 |
| **Description** | Manager shall approve an application → creates student + enrollment |
| **Priority** | P0 |
| **Preconditions** | Application is PENDING |
| **Trigger** | POST /api/v1/applications/{id}/approve |

**Explanation:**

**FR-STU-04** approves a pending application via `POST /api/v1/applications/{id}/approve`, which creates a student record and an enrollment in a single atomic transaction (BR-STU-06). The Manager specifies the shift and section to enroll the student in. On approval:
1. `students` row created with auto-generated registration number
2. `student_enrollments` row created for the specified section and academic year
3. Application status updated to `APPROVED`

**Validation:**
- Application already reviewed → `409 ALREADY_REVIEWED`
- Invalid section → `422`

**API Endpoint(s):**

##### POST /api/v1/applications/{id}/approve

**Request:**
```json
{
  "shift_id": "uuid",
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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-STU-06 | Approving an application creates student + enrollment in a single transaction |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-STU-05: Reject Application

| Property | Value |
|----------|-------|
| **ID** | FR-STU-05 |
| **Description** | Manager shall reject an application with reason |
| **Priority** | P0 |
| **Preconditions** | Application is PENDING |
| **Trigger** | POST /api/v1/applications/{id}/reject |

**Explanation:**

**FR-STU-05** rejects a pending application via `POST /api/v1/applications/{id}/reject`. The Manager provides a rejection reason. The application status is updated to `REJECTED`. No student record is created.

**Validation:**
- Application already reviewed → `409 ALREADY_REVIEWED`

**API Endpoint(s):**

##### POST /api/v1/applications/{id}/reject

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-STU-06: Registration Number Generation

| Property | Value |
|----------|-------|
| **ID** | FR-STU-06 |
| **Description** | System shall auto-generate registration number (`YY + sequence`) on approval in an atomic transaction |
| **Priority** | P0 |
| **Preconditions** | Application approved |
| **Trigger** | During approval |

**Explanation:**

**FR-STU-06** auto-generates the registration number during application approval. The format is `YY` (2-digit admission year) + sequence per tenant, resetting each academic year (e.g., the first student admitted in 2026 gets `26000001`). The sequence is generated atomically using `SELECT ... FOR UPDATE` to prevent race conditions (BR-STU-02). The registration number is permanent and never changes (BR-STU-03).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-STU-01 | Registration number format: `YY` (admission year) + sequence per tenant (resets each academic year) |
| BR-STU-02 | Registration number sequence is generated atomically using `SELECT ... FOR UPDATE` |
| BR-STU-03 | Registration number is permanent and never changes |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-STU-07: Roll Number Assignment

| Property | Value |
|----------|-------|
| **ID** | FR-STU-07 |
| **Description** | System shall auto-assign roll number within section (sequential) |
| **Priority** | P0 |
| **Preconditions** | Enrollment created |
| **Trigger** | During approval |

**Explanation:**

**FR-STU-07** auto-assigns a roll number when a student is enrolled in a section. Roll numbers are sequential within each (section + academic year) combination (BR-STU-04). The system assigns the next available number (e.g., if the section already has 30 students, the next student gets roll number 31).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-STU-04 | Roll numbers are unique per (section + academic year) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **8.** FR-STU-08: Student CRUD

| Property | Value |
|----------|-------|
| **ID** | FR-STU-08 |
| **Description** | School Admin/Manager shall CRUD student personal info |
| **Priority** | P0 |
| **Preconditions** | Student exists |
| **Trigger** | /api/v1/students endpoints |

**Explanation:**

**FR-STU-08** provides full CRUD operations on student personal information via `/api/v1/students` endpoints. School Admin and authorized Managers can view, create, update, soft-delete, and search student records. Deletion is soft-delete — the student's `deleted_at` timestamp is set and data is preserved for reports and audit. This covers changes to name, address, guardian details, and other personal fields.

**API Endpoint(s):**

##### CRUD /api/v1/students

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/students | List/search students |
| GET | /api/v1/students/{id} | Get student with enrollments |
| POST | /api/v1/students | Create student directly (skip application) |
| PATCH | /api/v1/students/{id} | Update personal info |
| DELETE | /api/v1/students/{id} | Soft-delete a student (sets `deleted_at`, data preserved) |

**GET /api/v1/students?search=alice&class_id=uuid&academic_year_id=uuid**

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
        "shift": "Morning",
        "section": "A",
        "roll_number": 1
      }
    }
  ]
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-STU-05 | A student can have only one enrollment per academic year |
| BR-STU-07 | Student statuses: `ACTIVE`, `DROPPED`, `TRANSFERRED`, `GRADUATED` (see Prisma `StudentStatus` enum) |
| BR-STU-08 | Enrollment statuses: `ACTIVE`, `COMPLETED`, `PROMOTED`, `REPEATED` (see Prisma `EnrollmentStatus` enum) |
| BR-STU-09 | Student deletion is soft-delete — `deleted_at` timestamp is set, data is preserved for reports and audit |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **9.** FR-STU-09: Student Search

| Property | Value |
|----------|-------|
| **ID** | FR-STU-09 |
| **Description** | System shall search students by name, roll number, registration number, class |
| **Priority** | P0 |
| **Preconditions** | Students exist |
| **Trigger** | GET /api/v1/students?search=... |

**Explanation:**

**FR-STU-09** provides student search by name, roll number, registration number, or class via `GET /api/v1/students?search=...`. Additional filters include `class_id` and `academic_year_id`. Results include the student's current enrollment details.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **10.** FR-STU-10: Student Promotion

| Property | Value |
|----------|-------|
| **ID** | FR-STU-10 |
| **Description** | School Admin shall promote students to next class (bulk or individual) |
| **Priority** | P0 |
| **Preconditions** | Academic year exists, previous enrollment exists |
| **Trigger** | POST /api/v1/students/promote |

**Explanation:**

**FR-STU-10** allows School Admin to promote students to the next class, either individually or in bulk via `POST /api/v1/students/promote`. The request specifies the from-class, to-class, shift, section, and list of student IDs. New enrollments are created for the next academic year with fresh roll numbers. The operation is transactional — all-or-nothing.

**API Endpoint(s):**

##### POST /api/v1/students/promote

Bulk promote students to next class.

**Request:**
```json
{
  "academic_year_id": "uuid",
  "from_class_id": "uuid",
  "to_class_id": "uuid",
  "shift_id": "uuid",
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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **11.** FR-STU-11: End-of-Year Outcomes

| Property | Value |
|----------|-------|
| **ID** | FR-STU-11 |
| **Description** | System shall handle end-of-year outcomes: Promote, Repeat, Graduate, Transfer, Dropout |
| **Priority** | P0 |
| **Preconditions** | Academic year ending |
| **Trigger** | POST /api/v1/students/{id}/outcome |

**Explanation:**

**FR-STU-11** handles end-of-year outcomes for students via `POST /api/v1/students/{id}/outcome`. Supported outcomes:
- **Promote** — moves to next class
- **Repeat** — stays in same class
- **Graduate** — marks as graduated
- **Transfer** — marks as transferred out
- **Dropout** — marks as dropped out

Each outcome updates the student's status and enrollment record accordingly.

**API Endpoint(s):**

##### POST /api/v1/students/{id}/outcome

Set end-of-year outcome for a student.

**Request:**
```json
{
  "outcome": "PROMOTED",
  "academic_year_id": "uuid",
  "next_class_id": "uuid",
  "next_shift_id": "uuid",
  "next_section_id": "uuid"
}
```

**Response `200 OK`:**
```json
{
  "student_id": "uuid",
  "outcome": "PROMOTED",
  "enrollment_id": "uuid"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 422 | `VALIDATION_ERROR` | Invalid outcome value |
| 404 | `NOT_FOUND` | Student not found |

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **12.** FR-STU-12: Import Students from Excel

| Property | Value |
|----------|-------|
| **ID** | FR-STU-12 |
| **Description** | School Admin shall import students from Excel |
| **Priority** | P1 |
| **Preconditions** | Authenticated |
| **Trigger** | POST /api/v1/students/import |

**Explanation:**

**FR-STU-12** allows School Admin to import students from an Excel file via `POST /api/v1/students/import`. The system parses the file, validates rows, and creates student + enrollment records. Validation errors are reported per row without rolling back successful imports (partial success model). School Admin can download an Excel template with the required columns before importing.

**API Endpoint(s):**

##### GET /api/v1/students/import/template

Download an Excel template with the required columns for student import.

**Response:** Binary Excel stream (.xlsx) with column headers

##### POST /api/v1/students/import

Import students from Excel file.

**Request:** `multipart/form-data`
- `file`: Excel file (.xlsx)

**Response `200 OK`:**
```json
{
  "imported": 45,
  "errors": [
    { "row": 12, "message": "Invalid email format" },
    { "row": 23, "message": "Duplicate registration number" }
  ]
}
```

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **13.** FR-STU-13: Export Student List

| Property | Value |
|----------|-------|
| **ID** | FR-STU-13 |
| **Description** | School Admin shall export student list to Excel/PDF |
| **Priority** | P1 |
| **Preconditions** | Authenticated |
| **Trigger** | GET /api/v1/students/export |

**Explanation:**

**FR-STU-13** allows School Admin to export student lists to Excel or PDF via `GET /api/v1/students/export`. The export includes all student details and their current enrollment information, filtered by the provided query parameters.

**API Endpoint(s):**

##### GET /api/v1/students/export

Export student list.

**Query params:** `?class_id=uuid&shift_id=uuid&section_id=uuid&academic_year_id=uuid&format=excel|pdf`

**Response:** Binary file stream (Excel or PDF)

**No related business rules or open questions.**

---




#### **14.** FR-STU-14: Auto-Expire Pending Applications

| Property | Value |
|----------|-------|
| **ID** | FR-STU-14 |
| **Description** | System shall auto-delete applications in PENDING state after 30 days |
| **Priority** | P1 |
| **Preconditions** | Application is PENDING for 30+ days |
| **Trigger** | Cron/queue job |

**Explanation:**

**FR-STU-14** automatically deletes applications that have been in `PENDING` status for 30 days or more. This prevents the temporary admission table from accumulating stale records. The cleanup runs via a cron job or queue worker.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-STU-09 | Applications in `PENDING` status for 30+ days are automatically deleted |

---

#### Edge Cases & Error States

| Scenario | Behavior |
|----------|----------|
| Approve already-approved application | 409 ALREADY_REVIEWED |
| Reject already-rejected application | 409 ALREADY_REVIEWED |
| Approve with invalid section_id | 422 section does not belong to shift |
| Approve with invalid shift_id | 422 shift does not belong to class |
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

**Quick Summary:** Manager selects class → shift → section → date, sees all enrolled students (default Present), unchecks absentees, and saves. Attendance has a workflow (DRAFT → SUBMITTED) and supports corrections with audit logging. Absentee SMS can be sent after attendance is saved.

> **Data Model:** Full schema in DB Dictionary §Table 15 (attendance_sessions), §Table 16 (attendance_records).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-ATT-01 | Load Enrolled Students | Manager shall select class → shift → section → date and load enrolled students | P0 | Academic year active, enrollment exists | GET /api/v1/attendance/sessions/init |
| 2 | FR-ATT-02 | Default to Present | System shall default all students to Present | P0 | Session initialized | Before save |
| 3 | FR-ATT-03 | Save Attendance Session | Manager shall save attendance (all students marked) | P0 | Session in DRAFT | POST /api/v1/attendance/sessions |
| 4 | FR-ATT-04 | Submit Attendance | Manager shall submit attendance (DRAFT → SUBMITTED) | P0 | Session saved | POST /api/v1/attendance/sessions/{id}/submit |
| 5 | FR-ATT-05 | Edit Attendance After Submission | Manager shall edit attendance after submission (audit logged) | P1 | Session exists | PATCH /api/v1/attendance/sessions/{id}/records |
| 6 | FR-ATT-06 | Send Absentee SMS | Manager shall send SMS to guardians of absent students | P1 | Session SUBMITTED, students absent | POST /api/v1/attendance/sessions/{id}/send-sms |
| 7 | FR-ATT-07 | Attendance Reports | School Admin shall view attendance reports (per class, per student, date range) | P0 | Attendance exists | GET /api/v1/attendance/reports |

---





#### **1.** FR-ATT-01: Load Enrolled Students

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-01 |
| **Description** | Manager shall select class → shift → section → date and load enrolled students |
| **Priority** | P0 |
| **Preconditions** | Academic year active, enrollment exists |
| **Trigger** | GET /api/v1/attendance/sessions/init |

**Explanation:**

**FR-ATT-01** initializes an attendance session by loading enrolled students for a given class, shift, section, and date via `GET /api/v1/attendance/sessions/init`. Each student row displays Roll, Name, and previous 3 days attendance (P/A badges). The response includes all students enrolled in that section for the current academic year, each defaulted to `PRESENT`. The user can then modify individual attendance statuses before saving.

**API Endpoint(s):**

##### GET /api/v1/attendance/sessions/init

Load enrolled students for attendance marking.

**Query params:** `?class_id=uuid&shift_id=uuid&section_id=uuid&date=2026-07-10`

**Response `200 OK`:**
```json
{
  "session_id": "uuid",
  "class_id": "uuid",
  "shift_id": "uuid",
  "section_id": "uuid",
  "date": "2026-07-10",
  "students": [
    { "student_id": "uuid", "roll_number": 1, "full_name": "Alice", "status": "PRESENT", "previous_attendance": ["P", "P", "A"] },
    { "student_id": "uuid", "roll_number": 2, "full_name": "Bob", "status": "PRESENT", "previous_attendance": ["P", "P", "P"] }
  ]
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `NOT_FOUND` | Class/section not found |

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-ATT-02: Default to Present

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-02 |
| **Description** | System shall default all students to Present |
| **Priority** | P0 |
| **Preconditions** | Session initialized |
| **Trigger** | Before save |

**Explanation:**

**FR-ATT-02** defaults all students to `PRESENT` when initializing an attendance session. This follows the common-sense assumption that most students are present — the Manager only needs to mark exceptions (absent).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ATT-02 | Attendance defaults to `PRESENT` — only absentees are explicitly marked |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-ATT-03: Save Attendance Session

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-03 |
| **Description** | Manager shall save attendance (all students marked) |
| **Priority** | P0 |
| **Preconditions** | Session in DRAFT |
| **Trigger** | POST /api/v1/attendance/sessions |

**Explanation:**

**FR-ATT-03** saves the attendance session with all student records via `POST /api/v1/attendance/sessions`. The session is created in `DRAFT` status, allowing further edits before final submission.

**Validation:**
- Duplicate date for same section → `409` (BR-ATT-01)

**API Endpoint(s):**

##### POST /api/v1/attendance/sessions

Create (or update) an attendance session.

**Request:**
```json
{
  "academic_year_id": "uuid",
  "class_id": "uuid",
  "shift_id": "uuid",
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

**ATTENDANCE_STATUS ENUM:** `PRESENT`, `ABSENT`

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ATT-01 | A section can have only one attendance session per date per academic year |
| BR-ATT-05 | Only today's and future dates can have attendance taken (configurable backdate limit) |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-ATT-01 | Should attendance status include more than PRESENT/ABSENT? *(Closed: only Present and Absent supported per PRD §6.4)* |
| Q-ATT-03 | Backdate limit — 7 days configurable? Or fixed? |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-ATT-04: Submit Attendance

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-04 |
| **Description** | Manager shall submit attendance (DRAFT → SUBMITTED) |
| **Priority** | P0 |
| **Preconditions** | Session saved |
| **Trigger** | POST /api/v1/attendance/sessions/{id}/submit |

**Explanation:**

**FR-ATT-04** submits a draft attendance session, transitioning it from `DRAFT` to `SUBMITTED` via `POST /api/v1/attendance/sessions/{id}/submit`. Once submitted, the session is considered final and corrections require explicit editing (FR-ATT-05) with audit logging.

**Validation:**
- Already submitted → `409 ALREADY_SUBMITTED`

**API Endpoint(s):**

##### POST /api/v1/attendance/sessions/{id}/submit

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-ATT-05: Edit Attendance After Submission

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-05 |
| **Description** | Manager shall edit attendance after submission (audit logged) |
| **Priority** | P1 |
| **Preconditions** | Session exists |
| **Trigger** | PATCH /api/v1/attendance/sessions/{id}/records |

**Explanation:**

**FR-ATT-05** allows editing attendance records after submission. Changes are tracked in `audit_logs` with old and new values. This provides an audit trail for attendance corrections. Edits are only possible for users with the appropriate permission and are restricted to a configurable window (e.g., same day only) per BR-ATT-06.

**API Endpoint(s):**

##### PATCH /api/v1/attendance/sessions/{id}/records

Edit attendance records after submission.

**Request:**
```json
{
  "records": [
    { "student_id": "uuid", "status": "ABSENT" }
  ]
}
```

**Response `200 OK`:**
```json
{
  "updated": 1,
  "session_id": "uuid"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `NOT_FOUND` | Session not found |
| 422 | `VALIDATION_ERROR` | Invalid status value |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ATT-03 | Attendance corrections after submission are logged in `audit_logs` |
| BR-ATT-06 | Attendance editing is restricted to a configurable window (e.g., same day only) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-ATT-06: Send Absentee SMS

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-06 |
| **Description** | Manager shall send SMS to guardians of absent students |
| **Priority** | P1 |
| **Preconditions** | Session SUBMITTED, students absent |
| **Trigger** | POST /api/v1/attendance/sessions/{id}/send-sms |

**Explanation:**

**FR-ATT-06** sends SMS notifications to guardians of absent students after an attendance session is submitted. Each SMS deducts 1 credit from the tenant's `sms_balance`. If balance is zero, SMS sending is blocked but the session status is unaffected (BR-NOT-01).

**API Endpoint(s):**

##### POST /api/v1/attendance/sessions/{id}/send-sms

**Response `200 OK`:**
```json
{
  "sent": 1,
  "failed": 0,
  "sms_sent_at": "2026-07-10T10:30:00Z"
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-ATT-04 | Each SMS sent consumes 1 credit from `tenants.sms_balance` |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-ATT-07: Attendance Reports

| Property | Value |
|----------|-------|
| **ID** | FR-ATT-07 |
| **Description** | School Admin shall view attendance reports (per class, per student, date range) |
| **Priority** | P0 |
| **Preconditions** | Attendance exists |
| **Trigger** | GET /api/v1/attendance/reports |

**Explanation:**

**FR-ATT-07** provides attendance reports for School Admin, filterable by class, shift (optional), student, and date range. The reports aggregate attendance data across sessions and can be used for monitoring student attendance patterns. Reports can be viewed as a monthly breakdown (tabular + summary) or as a full date-range summary.

**API Endpoint(s):**

##### GET /api/v1/attendance/reports

View attendance reports.

**Query params:** `?class_id=uuid&shift_id=uuid&student_id=uuid&from_date=2026-01-01&to_date=2026-12-31&group_by=monthly`

**Response `200 OK` (monthly grouping):**
```json
{
  "class_id": "uuid",
  "group_by": "monthly",
  "months": [
    {
      "month": "2026-01",
      "total_days": 22,
      "records": [
        { "student_id": "uuid", "full_name": "Alice", "present": 21, "absent": 1, "percentage": 95.5 }
      ]
    }
  ],
  "summary": {
    "total_days": 180,
    "total_present": 175,
    "total_absent": 5,
    "overall_percentage": 97.2
  }
}
```

**No related business rules or open questions.**

---

#### Open Questions

| # | Question |
|---|----------|
| Q-ATT-01 | Should attendance status include more than PRESENT/ABSENT? *(Closed: only Present and Absent supported per PRD §6.4)* |
| Q-ATT-02 | Is "month closure" (making attendance read-only) in MVP scope? *(Closed: configurable toggle in Settings — see FR-SET-02)* |
| Q-ATT-03 | Backdate limit — 7 days configurable? Or fixed? |

---

### 3.9 Result Management

**Quick Summary:** School Admin creates exams for a class, adds subjects with full marks, pass marks, and subject weight, enters student marks, and publishes results. Grade scales define letter grade + GPA mapping. Results trigger SMS/Email notifications on publish. Publishing is School Admin-only.

> **Data Model:** Full schema in DB Dictionary §Table 17 (exams), §Table 18 (exam_subjects), §Table 19 (marks), §Table 20 (grade_scales).

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-RES-01 | Create Exam | School Admin shall create exams with name, start/end date, class | P1 | Academic year exists, class exists | POST /api/v1/exams |
| 2 | FR-RES-02 | Configure Exam Subjects | School Admin shall configure subjects for an exam (full marks, pass marks, display order) | P1 | Exam in DRAFT | POST /api/v1/exams/{id}/subjects |
| 3 | FR-RES-03 | Create Grade Scale | School Admin shall create grade scales for the tenant | P1 | Authenticated | POST /api/v1/grade-scales |
| 4 | FR-RES-04 | Enter Marks | Manager shall enter marks per student per subject | P1 | Exam in DRAFT | PUT /api/v1/exams/{id}/marks |
| 5 | FR-RES-05 | Auto-Calculate Results | System shall auto-calculate total, percentage, grade, GPA | P1 | Marks entered | During calculation/display |
| 6 | FR-RES-06 | Publish Results | School Admin shall publish exam results | P1 | All marks entered | POST /api/v1/exams/{id}/publish |
| 7 | FR-RES-07 | Unpublish Results | School Admin shall unpublish results for editing | P1 | Results are published | POST /api/v1/exams/{id}/unpublish |
| 8 | FR-RES-08 | Generate Rank List | School Admin shall generate rank list | P1 | Results published | GET /api/v1/exams/{id}/rank-list |
| 9 | FR-RES-09 | View Result Sheet | School Admin/Manager shall view results in tabular format (student-wise or class-wise) | P1 | Exam exists, marks entered | GET /api/v1/exams/{id}/results |

---





#### **1.** FR-RES-01: Create Exam

| Property | Value |
|----------|-------|
| **ID** | FR-RES-01 |
| **Description** | School Admin shall create exams with name, start/end date, class |
| **Priority** | P1 |
| **Preconditions** | Academic year exists, class exists |
| **Trigger** | POST /api/v1/exams |

**Explanation:**

**FR-RES-01** allows School Admin to create exams for a specific class via `POST /api/v1/exams`. The exam includes a name, start/end dates, and is linked to an academic year and class. The exam starts in `DRAFT` status.

**Validation:**
- Duplicate exam name for the same class and academic year → `409` (BR-RES-01)

**API Endpoint(s):**

##### POST /api/v1/exams

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-RES-01 | A class cannot have two exams with the same name in the same academic year |
| BR-RES-02 | Marks can only be entered for the current academic year |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-RES-02: Configure Exam Subjects

| Property | Value |
|----------|-------|
| **ID** | FR-RES-02 |
| **Description** | School Admin shall configure subjects for an exam (full marks, pass marks, subject weight, display order) |
| **Priority** | P1 |
| **Preconditions** | Exam in DRAFT |
| **Trigger** | POST /api/v1/exams/{id}/subjects |

**Explanation:**

**FR-RES-02** configures the subjects that are part of an exam via `POST /api/v1/exams/{id}/subjects`. For each subject, the School Admin specifies full marks, pass marks, subject weight (optional, default 1), and display order. Subject weight is a multiplier used for weighted percentage and GPA calculations. Subjects are drawn from the class's subject pool. Configuration is only possible while the exam is in `DRAFT` status.

**API Endpoint(s):**

##### POST /api/v1/exams/{id}/subjects

**Request:**
```json
{
  "subjects": [
    {
      "subject_id": "uuid",
      "full_marks": 100,
      "pass_marks": 33,
      "subject_weight": 1,
      "display_order": 1
    }
  ]
}
```

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-RES-03: Create Grade Scale

| Property | Value |
|----------|-------|
| **ID** | FR-RES-03 |
| **Description** | School Admin shall create grade scales for the tenant |
| **Priority** | P1 |
| **Preconditions** | Authenticated |
| **Trigger** | POST /api/v1/grade-scales |

**Explanation:**

**FR-RES-03** allows School Admin to create grade scales via `POST /api/v1/grade-scales`. Grade scales define the mapping from mark ranges to letter grades and GPA (e.g., 80-100 → A+ → 5.00). Grade scales are tenant-wide.

**API Endpoint(s):**

##### POST /api/v1/grade-scales

Create a grade scale.

**Request:**
```json
{
  "name": "Standard Grade Scale",
  "grades": [
    { "min_marks": 80, "max_marks": 100, "grade_letter": "A+", "grade_point": 5.0 },
    { "min_marks": 70, "max_marks": 79, "grade_letter": "A", "grade_point": 4.0 }
  ]
}
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "name": "Standard Grade Scale",
  "grade_count": 2
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 422 | `VALIDATION_ERROR` | Overlapping grade ranges |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-RES-03 | Grade scale — is one per tenant enough, or per academic year? |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-RES-04: Enter Marks

| Property | Value |
|----------|-------|
| **ID** | FR-RES-04 |
| **Description** | Manager shall enter marks per student per subject |
| **Priority** | P1 |
| **Preconditions** | Exam in DRAFT |
| **Trigger** | PUT /api/v1/exams/{id}/marks |

**Explanation:**

**FR-RES-04** allows Manager to enter marks per student per subject via `PUT /api/v1/exams/{id}/marks`. The Manager provides obtained marks for each student-subject pair. If a student was absent, `is_absent = true` is set and marks must be null (BR-RES-06). Marks can only be entered while the exam is in DRAFT status — results must be published separately (FR-RES-06) after all marks are entered.

**Validation:**
- Absent student with marks → validation error

**API Endpoint(s):**

##### PUT /api/v1/exams/{id}/marks

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-RES-06 | IF `is_absent = true` THEN `obtained_marks` must be null |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-RES-01 | Should marks be enterable in a grid view (students × subjects) or one-by-one? |
| Q-RES-02 | Can multiple managers enter marks for different subjects in the same exam? |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-RES-05: Auto-Calculate Results

| Property | Value |
|----------|-------|
| **ID** | FR-RES-05 |
| **Description** | System shall auto-calculate total, percentage, grade, GPA |
| **Priority** | P1 |
| **Preconditions** | Marks entered |
| **Trigger** | During calculation/display |

**Explanation:**

**FR-RES-05** calculates total marks, percentage, letter grade, and GPA at runtime based on the entered marks and the tenant's grade scale. These values are computed on display — they are never stored in the `marks` table (BR-RES-05), ensuring they always reflect the latest grade scale configuration.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-RES-05 | Grades/GPA are calculated at runtime — never stored in `marks` table |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-RES-06: Publish Results

| Property | Value |
|----------|-------|
| **ID** | FR-RES-06 |
| **Description** | School Admin shall publish exam results |
| **Priority** | P1 |
| **Preconditions** | All marks entered |
| **Trigger** | POST /api/v1/exams/{id}/publish |

**Explanation:**

**FR-RES-06** publishes exam results via `POST /api/v1/exams/{id}/publish`. Only School Admin can publish (BR-RES-03). Publishing sets `exams.result_published_at` and automatically triggers SMS/Email notifications to guardians. Publishing is a one-way gate — editing requires unpublishing first.

**Validation:**
- Marks missing for some students → `400 MARKS_INCOMPLETE`
- Non-School Admin → `403 FORBIDDEN`

**API Endpoint(s):**

##### POST /api/v1/exams/{id}/publish

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-RES-03 | Publish is exclusive to School Admin role |
| BR-RES-04 | IF marks are published THEN editing requires unpublish first |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-RES-07: Unpublish Results

| Property | Value |
|----------|-------|
| **ID** | FR-RES-07 |
| **Description** | School Admin shall unpublish results for editing |
| **Priority** | P1 |
| **Preconditions** | Results are published |
| **Trigger** | POST /api/v1/exams/{id}/unpublish |

**Explanation:**

**FR-RES-07** unpublishes published results via `POST /api/v1/exams/{id}/unpublish`, allowing marks to be edited. This returns the exam to a state where marks can be modified. After editing, the exam must be published again for the changes to be visible.

**API Endpoint(s):**

##### POST /api/v1/exams/{id}/unpublish

Unpublish exam results.

**Response `200 OK`:**
```json
{
  "exam_id": "uuid",
  "status": "DRAFT"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `ALREADY_DRAFT` | Exam is not published |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-RES-04 | IF marks are published THEN editing requires unpublish first |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **8.** FR-RES-08: Generate Rank List

| Property | Value |
|----------|-------|
| **ID** | FR-RES-08 |
| **Description** | School Admin shall generate rank list |
| **Priority** | P1 |
| **Preconditions** | Results published |
| **Trigger** | GET /api/v1/exams/{id}/rank-list |

**Explanation:**

**FR-RES-08** generates a rank list for the exam via `GET /api/v1/exams/{id}/rank-list`. Students are ranked by total marks in descending order. Ranking is only available after results are published.

**API Endpoint(s):**

##### GET /api/v1/exams/{id}/rank-list

Generate rank list for exam.

**Response `200 OK`:**
```json
{
  "exam_id": "uuid",
  "ranks": [
    { "rank": 1, "student_id": "uuid", "full_name": "Alice", "total_marks": 95.0 },
    { "rank": 2, "student_id": "uuid", "full_name": "Bob", "total_marks": 88.5 }
  ]
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `NOT_PUBLISHED` | Results not yet published |

**No related business rules or open questions.**

---

#### **9.** FR-RES-09: View Result Sheet

| Property | Value |
|----------|-------|
| **ID** | FR-RES-09 |
| **Description** | School Admin/Manager shall view results in tabular format (student-wise or class-wise) |
| **Priority** | P1 |
| **Preconditions** | Exam exists, marks entered |
| **Trigger** | GET /api/v1/exams/{id}/results |

**Explanation:**

**FR-RES-09** allows viewing a tabular result sheet for an exam via `GET /api/v1/exams/{id}/results`. The view can be toggled between student-wise (all subjects for one student) and class-wise (all students for one subject). Results are visible once marks are entered, regardless of publish status.

**API Endpoint(s):**

##### GET /api/v1/exams/{id}/results

View result sheet in tabular format.

**Query params:** `?view=student-wise|class-wise&student_id=uuid&exam_subject_id=uuid`

**Response `200 OK` (student-wise):**
```json
{
  "exam_id": "uuid",
  "exam_name": "Midterm 2026",
  "view": "student-wise",
  "student": {
    "id": "uuid",
    "registration_no": "26000001",
    "full_name": "Alice Johnson"
  },
  "subjects": [
    {
      "subject": "Mathematics",
      "full_marks": 100,
      "pass_marks": 33,
      "obtained_marks": 85,
      "grade": "A",
      "gpa": 4.0,
      "is_absent": false
    }
  ],
  "total": 85,
  "percentage": 85,
  "grade": "A",
  "gpa": 4.0
}
```

**Response `200 OK` (class-wise):**
```json
{
  "exam_id": "uuid",
  "exam_name": "Midterm 2026",
  "view": "class-wise",
  "subject": "Mathematics",
  "students": [
    {
      "student_id": "uuid",
      "registration_no": "26000001",
      "full_name": "Alice Johnson",
      "obtained_marks": 85,
      "grade": "A",
      "gpa": 4.0,
      "is_absent": false
    }
  ]
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `EXAM_NOT_FOUND` | Exam not found |

**No related business rules or open questions.**

---

---

### 3.10 Notice Board

**Quick Summary:** School Admin or authorized Manager creates notices with a title and body text, optionally attaching files (PDF or Image). Notices can be scheduled (future publish_at), expire automatically, and are visible to all authenticated users within the tenant. Published notices cannot be edited — must archive and recreate.

> **Data Model:** Full schema in DB Dictionary §Table 21.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-NTC-01 | Create Notice | Authorized user shall create a notice with title, body text, and optional file attachments | P1 | Authenticated | POST /api/v1/notices |
| 2 | FR-NTC-02 | Schedule Notice | System shall schedule notice for future publication | P1 | Notice created with future `publish_at` | During create |
| 3 | FR-NTC-03 | Auto-Publish Scheduled Notice | System shall auto-publish scheduled notices when `publish_at` is reached | P1 | Scheduled notice exists | Cron/queue job |
| 4 | FR-NTC-04 | Archive Notice | Authorized user shall archive a published notice | P1 | Notice is published | POST /api/v1/notices/{id}/archive |
| 5 | FR-NTC-05 | View Published Notices | All authenticated users shall view published notices | P1 | Notice is published | GET /api/v1/notices |
| 6 | FR-NTC-06 | Auto-Hide Expired Notice | System shall auto-hide expired notices based on `expires_at` | P1 | `expires_at` is reached | Cron/queue job |
| 7 | FR-NTC-07 | Delete Notice | Authorized user shall soft-delete a notice | P1 | Notice exists | DELETE /api/v1/notices/{id} |

---





#### **1.** FR-NTC-01: Create Notice

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-01 |
| **Description** | Authorized user shall create a notice with title, body text, and optional file attachments |
| **Priority** | P1 |
| **Preconditions** | Authenticated |
| **Trigger** | POST /api/v1/notices |

**Explanation:**

**FR-NTC-01** allows authorized users (School Admin or Manager with permission) to create a notice with a title and body text via `POST /api/v1/notices`, optionally attaching PDF or image files. Notices can be scheduled for future publication (FR-NTC-02).

**API Endpoint(s):**

##### POST /api/v1/notices

**Request (multipart/form-data):**
```
title: "Final Exam Routine"              (required)
body: "The final exam will start from..." (required)
attachments: (pdf or image uploads, optional, multiple)
publish_at: "2026-12-01T08:00:00Z"       (optional, defaults to now)
expires_at: "2026-12-20T00:00:00Z"       (optional)
```

**Response `201 Created`:**
```json
{
  "id": "uuid",
  "title": "Final Exam Routine",
  "body": "The final exam will start from...",
  "attachments": [
    { "id": "uuid", "file_url": "https://res.cloudinary.com/.../notice.pdf", "file_type": "PDF" }
  ],
  "is_published": true,
  "publish_at": "2026-12-01T08:00:00Z",
  "expires_at": "2026-12-20T00:00:00Z"
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NTC-02 | Only PDF and IMAGE file types are supported for notice attachments |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-NTC-02: Schedule Notice

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-02 |
| **Description** | System shall schedule notice for future publication |
| **Priority** | P1 |
| **Preconditions** | Notice created with future `publish_at` |
| **Trigger** | During create |

**Explanation:**

**FR-NTC-02** allows notices to be scheduled for future publication. When a notice is created with a `publish_at` timestamp set in the future, it remains unpublished (`is_published = false`) until the scheduled time is reached.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-NTC-03: Auto-Publish Scheduled Notice

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-03 |
| **Description** | System shall auto-publish scheduled notices when `publish_at` is reached |
| **Priority** | P1 |
| **Preconditions** | Scheduled notice exists |
| **Trigger** | Cron/queue job |

**Explanation:**

**FR-NTC-03** auto-publishes scheduled notices when their `publish_at` timestamp is reached. This is implemented via a cron job or queue worker that periodically checks for due notices and sets `is_published = true`.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-NTC-04: Archive Notice

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-04 |
| **Description** | Authorized user shall archive a published notice |
| **Priority** | P1 |
| **Preconditions** | Notice is published |
| **Trigger** | POST /api/v1/notices/{id}/archive |

**Explanation:**

**FR-NTC-04** archives a published notice via `POST /api/v1/notices/{id}/archive`. Archived notices are hidden from the regular notice list but preserved for record-keeping. Published notices cannot be edited — they must be archived and a new notice created (BR-NTC-01).

**API Endpoint(s):**

##### POST /api/v1/notices/{id}/archive

Archive a published notice.

**Response `200 OK`:**
```json
{
  "notice_id": "uuid",
  "status": "ARCHIVED"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `ALREADY_ARCHIVED` | Notice is already archived |
| 404 | `NOT_FOUND` | Notice not found |

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NTC-01 | Published notices cannot be edited (must archive and recreate) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-NTC-05: View Published Notices

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-05 |
| **Description** | All authenticated users shall view published notices |
| **Priority** | P1 |
| **Preconditions** | Notice is published |
| **Trigger** | GET /api/v1/notices |

**Explanation:**

**FR-NTC-05** allows all authenticated users within the tenant to view published notices via `GET /api/v1/notices`. Only notices with `is_published = true` and `publish_at <= now()` are returned. Expired notices (past `expires_at`) are excluded.

**API Endpoint(s):**

##### GET /api/v1/notices

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Final Exam Routine",
      "body": "The final exam will start from...",
      "attachments": [
        { "id": "uuid", "file_url": "...", "file_type": "PDF" }
      ],
      "published_at": "2026-12-01T08:00:00Z"
    }
  ]
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NTC-03 | All authenticated users in the tenant can view published notices |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-NTC-06: Auto-Hide Expired Notice

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-06 |
| **Description** | System shall auto-hide expired notices based on `expires_at` |
| **Priority** | P1 |
| **Preconditions** | `expires_at` is reached |
| **Trigger** | Cron/queue job |

**Explanation:**

**FR-NTC-06** auto-hides expired notices when their `expires_at` date is reached. Like auto-publishing, this is implemented via a cron job or queue worker. Expired notices are not deleted — they remain in the database but are excluded from query results.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NTC-04 | IF `expires_at` is set and the date has passed THEN notice is hidden |

---




#### **7.** FR-NTC-07: Delete Notice

| Property | Value |
|----------|-------|
| **ID** | FR-NTC-07 |
| **Description** | Authorized user shall soft-delete a notice |
| **Priority** | P1 |
| **Preconditions** | Notice exists |
| **Trigger** | DELETE /api/v1/notices/{id} |

**Explanation:**

**FR-NTC-07** allows authorized users to soft-delete a notice via `DELETE /api/v1/notices/{id}`. The notice's `status` is set to `DELETED`, hiding it from all views. Data is preserved for audit purposes.

**API Endpoint(s):**

##### DELETE /api/v1/notices/{id}

**Response `200 OK`:**
```json
{
  "notice_id": "uuid",
  "status": "DELETED"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `NOT_FOUND` | Notice not found |

**No related business rules or open questions.**

---

---

### 3.11 Notification System

**Quick Summary:** The system sends SMS and Email through a centralized gateway (no per-tenant provider config). SMS quota is managed per tenant by Platform Admin. Notifications are triggered by attendance marking (absentees), result publishing, and manual ad-hoc sends. All notifications are logged in the `notifications` table for delivery tracking and debugging.

> **Data Model:** Full schema in DB Dictionary §Table 22.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-NOT-01 | Send Absentee SMS | System shall send SMS to guardians of absent students when triggered | P1 | Attendance session submitted, absent students exist | POST /api/v1/attendance/sessions/{id}/send-sms |
| 2 | FR-NOT-02 | Send Result Notification | System shall send SMS/Email to guardians when results are published | P1 | Exam results published | Auto-triggered on publish |
| 3 | FR-NOT-04 | Ad-Hoc Notification | Authorized user shall send ad-hoc SMS/Email | P1 | Authenticated | POST /api/v1/notifications/send |
| 4 | FR-NOT-05 | Deduct SMS Balance | System shall deduct SMS balance per SMS sent | P1 | SMS sent successfully | After send |
| 5 | FR-NOT-06 | Block SMS at Zero Balance | System shall block SMS sending when balance is 0 | P1 | `sms_balance` = 0 | Before send |
| 6 | FR-NOT-07 | Log Notifications | System shall log every notification with status, recipient, message | P0 | Notification sent or failed | After attempt |
| 7 | FR-NOT-08 | Retry Failed Notifications | System shall retry failed notifications (configurable) | P1 | Status = FAILED | Cron/queue job |
| 8 | FR-NOT-09 | Configure Notification Settings | School Admin shall enable/disable notification types and customize sender name | P1 | Authenticated as School Admin | PATCH /api/v1/settings/notifications |

---





#### **1.** FR-NOT-01: Send Absentee SMS

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-01 |
| **Description** | System shall send SMS to guardians of absent students when triggered |
| **Priority** | P1 |
| **Preconditions** | Attendance session submitted, absent students exist |
| **Trigger** | POST /api/v1/attendance/sessions/{id}/send-sms |

**Explanation:**

**FR-NOT-01** sends SMS notifications to guardians of absent students when the Manager triggers it via `POST /api/v1/attendance/sessions/{id}/send-sms`. Messages are sent through the centralized provider. Each SMS consumes 1 credit from the tenant's balance.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-NOT-02: Send Result Notification

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-02 |
| **Description** | System shall send SMS/Email to guardians when results are published |
| **Priority** | P1 |
| **Preconditions** | Exam results published |
| **Trigger** | Auto-triggered on publish |

**Explanation:**

**FR-NOT-02** automatically sends SMS/Email notifications to guardians when exam results are published. This is triggered by the result publish action (FR-RES-06). The notification includes the student's grades or a link to view the results.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-NOT-04: Ad-Hoc Notification

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-04 |
| **Description** | Authorized user shall send ad-hoc SMS/Email |
| **Priority** | P1 |
| **Preconditions** | Authenticated |
| **Trigger** | POST /api/v1/notifications/send |

**Explanation:**

**FR-NOT-04** allows authorized users to send ad-hoc SMS or Email messages via `POST /api/v1/notifications/send`. The user specifies the channel (SMS/Email), recipient, and message content. This is useful for emergency communications or custom alerts.

**API Endpoint(s):**

##### POST /api/v1/notifications/send

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

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-NOT-05: Deduct SMS Balance

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-05 |
| **Description** | System shall deduct SMS balance per SMS sent |
| **Priority** | P1 |
| **Preconditions** | SMS sent successfully |
| **Trigger** | After send |

**Explanation:**

**FR-NOT-05** deducts 1 SMS credit from `tenants.sms_balance` each time an SMS is sent successfully. The deduction happens after the provider confirms delivery, ensuring accurate balance tracking.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NOT-02 | Each SMS deducts 1 credit from `tenants.sms_balance` |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-NOT-06: Block SMS at Zero Balance

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-06 |
| **Description** | System shall block SMS sending when balance is 0 |
| **Priority** | P1 |
| **Preconditions** | `sms_balance` = 0 |
| **Trigger** | Before send |

**Explanation:**

**FR-NOT-06** blocks SMS sending when the tenant's SMS balance reaches zero. Email notifications continue to work. The endpoint returns `402 SMS_QUOTA_EXCEEDED`. The Platform Admin can increase the balance via FR-PLT-09.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NOT-03 | IF `sms_balance` = 0 THEN SMS is blocked (email still works) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-NOT-07: Log Notifications

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-07 |
| **Description** | System shall log every notification with status, recipient, message |
| **Priority** | P0 |
| **Preconditions** | Notification sent or failed |
| **Trigger** | After attempt |

**Explanation:**

**FR-NOT-07** logs every notification attempt (SMS and Email) in the `notifications` table, regardless of success or failure. Each log entry includes the channel, recipient, message, status, trigger type, and provider response. Logs are immutable once written (BR-NOT-04).

**API Endpoint(s):**

##### GET /api/v1/notifications

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-NOT-04 | Notification logs are immutable once written |
| BR-NOT-05 | All notifications go through the platform's centralized gateway (no per-tenant provider config) |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-NOT-01 | What SMS/Email provider will be used in MVP? (Twilio, SendGrid, local?) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-NOT-08: Retry Failed Notifications

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-08 |
| **Description** | System shall retry failed notifications (configurable) |
| **Priority** | P1 |
| **Preconditions** | Status = FAILED |
| **Trigger** | Cron/queue job |

**Explanation:**

**FR-NOT-08** retries failed notifications via a cron or queue job. Retries are configurable (max attempts, interval). Each retry increments `retry_count` and updates the log status. After exhausting retries, the notification is marked as permanently failed.

**No related business rules or open questions.**

---




#### **8.** FR-NOT-09: Configure Notification Settings

| Property | Value |
|----------|-------|
| **ID** | FR-NOT-09 |
| **Description** | School Admin shall enable/disable notification types and customize sender name |
| **Priority** | P1 |
| **Preconditions** | Authenticated as School Admin |
| **Trigger** | PATCH /api/v1/settings/notifications |

**Explanation:**

**FR-NOT-09** allows School Admin to configure notification preferences per tenant:
- **Notification types:** Enable/disable SMS and Email independently for each notification trigger (absentee alerts, result publication, ad-hoc sends).
- **Sender name:** Override the default sender name/ID for SMS and Email.
- **Templates:** Customize default notification templates using variables such as `{student_name}`, `{class}`, `{shift}`, `{section}`, `{date}`, `{school_name}`, `{percentage}`, `{subject}`.

Settings are stored in `tenant_settings` and apply to all subsequently triggered notifications. Template changes apply to the next notification, not retroactively.

**No related business rules or open questions.**

---

#### Open Questions

| # | Question |
|---|----------|
| Q-NOT-01 | What SMS/Email provider will be used in MVP? (Twilio, SendGrid, local?) |
| Q-NOT-02 | Are notification templates customizable per tenant? (No template table yet) |

---

### 3.12 Reports

**Quick Summary:** The Reports module provides unified report generation — enter a student's registration number, select a report type (Progress Report, Transfer Certificate, Attendance Report, Result Report), and load the report inline for viewing. Reports include school branding from Settings. Data is read-only at generation time. Reports can be exported to PDF via browser print.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-RPT-01 | Generate Progress Report | System shall generate per-student Progress Report PDF with subject-wise marks, grades, GPA, total, percentage | P1 | Student exists, exam results published | GET /api/v1/reports/students/{id}/progress |
| 2 | FR-RPT-02 | Generate Transfer Certificate | System shall generate Transfer Certificate PDF | P1 | Student status = TRANSFERRED | GET /api/v1/reports/students/{id}/tc |
| 3 | FR-RPT-03 | Generate Attendance Report | System shall generate attendance report PDF | P1 | Attendance records exist | GET /api/v1/reports/attendance |
| 4 | FR-RPT-04 | Generate Result Report | System shall generate result report PDF (per exam) | P1 | Results published | GET /api/v1/reports/results |
| 5 | FR-RPT-05 | Embed School Branding | System shall embed school logo and name from Settings in all reports | P1 | Settings configured | During generation |
| 6 | FR-RPT-06 | Unified Report Load | System shall load report data by registration number + report type for inline viewing | P1 | Student exists, data exists | GET /api/v1/reports/load |

---





#### **1.** FR-RPT-01: Generate Progress Report

| Property | Value |
|----------|-------|
| **ID** | FR-RPT-01 |
| **Description** | System shall generate per-student Progress Report PDF with subject-wise marks, grades, GPA, total, percentage |
| **Priority** | P1 |
| **Preconditions** | Student exists, exam results published |
| **Trigger** | GET /api/v1/reports/students/{id}/progress |

**Explanation:**

**FR-RPT-01** generates a per-student Progress Report PDF via `GET /api/v1/reports/students/{id}/progress`. The report includes the student's name, class, shift, section, roll number, and subject-wise marks with grades, GPA, total, and percentage for the selected academic year. School branding is embedded from Settings.

**API Endpoint(s):**

##### GET /api/v1/reports/students/{id}/progress

**Query params:** `?academic_year_id=uuid`

**Response:** Binary PDF stream

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-RPT-02: Generate Transfer Certificate

| Property | Value |
|----------|-------|
| **ID** | FR-RPT-02 |
| **Description** | System shall generate Transfer Certificate PDF |
| **Priority** | P1 |
| **Preconditions** | Student status = TRANSFERRED |
| **Trigger** | GET /api/v1/reports/students/{id}/tc |

**Explanation:**

**FR-RPT-02** generates a Transfer Certificate (TC) PDF for students with status `TRANSFERRED`. The TC includes student details, academic history, and the school's official seal and branding.

**API Endpoint(s):**

##### GET /api/v1/reports/students/{id}/tc

Generate Transfer Certificate PDF.

**Query params:** `?academic_year_id=uuid`

**Response:** Binary PDF stream

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-RPT-03: Generate Attendance Report

| Property | Value |
|----------|-------|
| **ID** | FR-RPT-03 |
| **Description** | System shall generate attendance report PDF |
| **Priority** | P1 |
| **Preconditions** | Attendance records exist |
| **Trigger** | GET /api/v1/reports/attendance |

**Explanation:**

**FR-RPT-03** generates attendance reports in PDF format, filterable by class, shift (optional), and academic year. Reports include attendance summaries (total days, present, absent, percentage) per student.

**API Endpoint(s):**

##### GET /api/v1/reports/attendance

**Query params:** `?class_id=uuid&shift_id=uuid&academic_year_id=uuid`

**Response:** Binary PDF stream

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-RPT-04: Generate Result Report

| Property | Value |
|----------|-------|
| **ID** | FR-RPT-04 |
| **Description** | System shall generate result report PDF (per exam) |
| **Priority** | P1 |
| **Preconditions** | Results published |
| **Trigger** | GET /api/v1/reports/results |

**Explanation:**

**FR-RPT-04** generates result report PDFs per exam, showing all students' marks for a specific exam. Reports are generated only for published results.

**API Endpoint(s):**

##### GET /api/v1/reports/results

Generate result report PDF.

**Query params:** `?exam_id=uuid&student_id=uuid`

**Response:** Binary PDF stream

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-RPT-05: Embed School Branding

| Property | Value |
|----------|-------|
| **ID** | FR-RPT-05 |
| **Description** | System shall embed school logo and name from Settings in all reports |
| **Priority** | P1 |
| **Preconditions** | Settings configured |
| **Trigger** | During generation |

**Explanation:**

**FR-RPT-05** ensures all generated reports include the school's branding (logo and name) from `tenant_settings`. This guarantees consistent, professional-looking documents across all report types.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-RPT-01 | Reports include school branding from `tenant_settings` |
| BR-RPT-02 | Reports are generated for current academic year by default |
| BR-RPT-03 | Data is read-only — reports never modify data |
| BR-RPT-04 | Reports only include data belonging to the current tenant |

**Related Open Questions:**

| # | Question |
|---|----------|
| Q-RPT-01 | PDF generation library? (DomPDF, Puppeteer, jsPDF?) |
| Q-RPT-03 | Are reports generated sync or async (queued)? |
---

#### **6.** FR-RPT-06: Unified Report Load

| Property | Value |
|----------|-------|
| **ID** | FR-RPT-06 |
| **Description** | System shall load report data by registration number + report type for inline viewing |
| **Priority** | P1 |
| **Preconditions** | Student exists, data exists |
| **Trigger** | GET /api/v1/reports/load |

**Explanation:**

**FR-RPT-06** provides a unified report loading flow as defined in the PRD. The user enters a student's registration number, selects a report type, and the system loads the report data as JSON for inline display (not a PDF download). The PDF export is handled via browser print dialog from the rendered inline view. Existing direct PDF endpoints (FR-RPT-01 through FR-RPT-04) remain available for programmatic access.

**API Endpoint(s):**

##### GET /api/v1/reports/load

Load report data for inline viewing.

**Query params:** `?registration_no=26000001&type=progress|tc|attendance|result&academic_year_id=uuid`

**Response `200 OK` (report data JSON):**
```json
{
  "registration_no": "26000001",
  "student_name": "Alice Johnson",
  "class": "Class 6",
  "shift": "Morning",
  "section": "A",
  "roll_number": 1,
  "report_type": "progress",
  "school_branding": {
    "name": "Sunrise School",
    "address": "123 Main St, Dhaka",
    "logo_url": "https://..."
  },
  "data": {}
}
```

The `data` field content varies by report type — subject-wise marks for progress/result reports, monthly attendance summary for attendance reports, student details and certification for transfer certificate.

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 404 | `STUDENT_NOT_FOUND` | No student found with given registration number |
| 400 | `INVALID_REPORT_TYPE` | Unsupported report type |
| 400 | `NO_DATA` | No data available for the selected report type |

**No related business rules or open questions.**

---

### 3.13 Dashboard

**Quick Summary:** Role-specific dashboards showing at-a-glance metrics. Platform Admin sees cross-tenant overview — tenant count, active/inactive status, total SMS usage, and system health. School Admin sees student count, today's attendance %, recent notices, upcoming exams/events, Manager activity summary, SMS balance/quota, and quick actions. Manager sees a subset based on their permissions.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-DSH-01 | School Admin Dashboard | System shall display School Admin dashboard with all metrics | P0 | Authenticated as School Admin | GET /api/v1/dashboard/school-admin |
| 2 | FR-DSH-02 | Manager Dashboard | System shall display Manager dashboard (permissions-dependent) | P0 | Authenticated as Manager | GET /api/v1/dashboard/manager |
| 3 | FR-DSH-03 | Current Year Scoping | Dashboard metrics shall be scoped to current academic year | P0 | Current academic year exists | During load |
| 4 | FR-DSH-04 | Platform Admin Dashboard | System shall display Platform Admin dashboard with cross-tenant metrics | P1 | Authenticated as Platform Admin | GET /api/v1/admin/dashboard |

---





#### **1.** FR-DSH-01: School Admin Dashboard

| Property | Value |
|----------|-------|
| **ID** | FR-DSH-01 |
| **Description** | System shall display School Admin dashboard with all metrics including SMS balance warning |
| **Priority** | P0 |
| **Preconditions** | Authenticated as School Admin |
| **Trigger** | GET /api/v1/dashboard/school-admin |

**Explanation:**

**FR-DSH-01** displays the School Admin dashboard at `GET /api/v1/dashboard/school-admin` with key metrics: total students, today's attendance percentage, whether today's attendance has been taken, recent notices, upcoming exams/events, Manager activity summary, SMS balance/quota (with a warning banner when balance drops below 10% of the original quota), and quick action links. All metrics are scoped to the current academic year (BR-DSH-01).

**API Endpoint(s):**

##### GET /api/v1/dashboard/school-admin

**Response `200 OK`:**
```json
{
  "total_students": 350,
  "attendance_percentage": 95.2,
  "today_attendance_taken": true,
  "recent_notices": [
    { "id": "uuid", "title": "Exam Routine", "published_at": "..." }
  ],
  "upcoming_exams": [
    { "id": "uuid", "name": "Midterm Exam", "exam_date": "2026-08-15" }
  ],
  "manager_activity_summary": {
    "recent_actions": 12,
    "pending_approvals": 3
  },
  "sms_balance": 420,
  "sms_quota": 500,
  "quick_actions": [
    { "label": "Add Student", "path": "/students/add" },
    { "label": "Take Attendance", "path": "/attendance" },
    { "label": "Create Notice", "path": "/notices/create" }
  ]
}
```

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-DSH-01 | Dashboard data is scoped to the current academic year |
| BR-DSH-03 | Dashboard fetches real-time data (no caching for MVP) |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-DSH-02: Manager Dashboard

| Property | Value |
|----------|-------|
| **ID** | FR-DSH-02 |
| **Description** | System shall display Manager dashboard (permissions-dependent) |
| **Priority** | P0 |
| **Preconditions** | Authenticated as Manager |
| **Trigger** | GET /api/v1/dashboard/manager |

**Explanation:**

**FR-DSH-02** displays the Manager dashboard at `GET /api/v1/dashboard/manager` with a subset of widgets based on the Manager's assigned permissions (BR-DSH-02). If assigned Student Management, the dashboard shows student count. If assigned Attendance, it shows today's attendance percentage and pending attendance. If assigned Results, it shows recent result entries. Quick actions are shown only for assigned modules.

**API Endpoint(s):**

##### GET /api/v1/dashboard/manager

**Query params:** `?modules=STUDENT_MANAGEMENT,ATTENDANCE_MANAGEMENT`

**Response:** Same format as school admin dashboard, but only shows widgets for assigned modules.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-DSH-02 | Managers only see metrics for modules they have View permission on |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-DSH-03: Current Year Scoping

| Property | Value |
|----------|-------|
| **ID** | FR-DSH-03 |
| **Description** | Dashboard metrics shall be scoped to current academic year |
| **Priority** | P0 |
| **Preconditions** | Current academic year exists |
| **Trigger** | During load |

**Explanation:**

**FR-DSH-03** scopes all dashboard metrics to the current academic year. This ensures consistency — counts, percentages, and lists all reflect the active academic year context.

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-DSH-01 | Dashboard data is scoped to the current academic year |

---
#### **4.** FR-DSH-04: Platform Admin Dashboard

| Property | Value |
|----------|-------|
| **ID** | FR-DSH-04 |
| **Description** | System shall display Platform Admin dashboard with cross-tenant metrics |
| **Priority** | P1 |
| **Preconditions** | Authenticated as Platform Admin |
| **Trigger** | GET /api/v1/admin/dashboard |

**Explanation:**

**FR-DSH-04** displays the Platform Admin dashboard at `GET /api/v1/admin/dashboard` with cross-tenant metrics: total tenant count, active/inactive tenant count, total SMS usage across all tenants, and system health indicators. This provides the platform-level overview referenced in FR-AUTH-05.

**API Endpoint(s):**

##### GET /api/v1/admin/dashboard

**Response `200 OK`:**
```json
{
  "total_tenants": 25,
  "active_tenants": 22,
  "inactive_tenants": 3,
  "total_sms_used": 15420,
  "total_sms_quota": 50000,
  "system_health": {
    "status": "healthy",
    "uptime_hours": 720,
    "active_sessions": 340
  }
}
```

**No related business rules or open questions.**

---



---

### 3.14 Audit Logging

**Quick Summary:** Every important business action (create, update, delete, publish, permission change, etc.) is recorded in `audit_logs` with actor, action, old/new values, IP, and timestamp. Logs are append-only and immutable.

> **Data Model:** Full schema in DB Dictionary §Table 23.

#### Functional Requirements


| # | ID | Name | Description | Priority | Preconditions | Trigger |
|---|-----|------|-------------|----------|---------------|---------|
| 1 | FR-AUD-01 | Log Student CUD | System shall log all CUD operations on student data | P0 | Action performed | After successful action |
| 2 | FR-AUD-02 | Log Attendance Corrections | System shall log attendance corrections after submission | P0 | Attendance session modified | After update |
| 3 | FR-AUD-03 | Log Result Publish/Unpublish | System shall log result publish/unpublish | P0 | Result published/unpublished | After action |
| 4 | FR-AUD-04 | Log Permission Changes | System shall log permission changes | P0 | Manager permissions updated | After update |
| 5 | FR-AUD-05 | Log SMS Balance Changes | System shall log SMS balance changes | P0 | Balance adjusted | After adjustment |
| 6 | FR-AUD-06 | Log Tenant Activation | System shall log tenant activation/deactivation | P0 | Tenant status changed | After change |
| 7 | FR-AUD-07 | View Tenant Audit Logs | School Admin shall view audit logs for their tenant | P0 | Authenticated | GET /api/v1/audit-logs |
| 8 | FR-AUD-08 | View Cross-Tenant Audit Logs | Platform Admin shall view audit logs across all tenants | P1 | Authenticated as Platform Admin | GET /api/v1/admin/audit-logs |

---





#### **1.** FR-AUD-01: Log Student CUD

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-01 |
| **Description** | System shall log all CUD operations on student data |
| **Priority** | P0 |
| **Preconditions** | Action performed |
| **Trigger** | After successful action |

**Explanation:**

**FR-AUD-01** logs all Create, Update, and Delete operations on student data in `audit_logs`. Each log entry captures the actor (user), action, entity type, old/new values (as JSONB), IP address, and timestamp. Routine reads and simple queries are NOT logged (BR-AUD-02).

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUD-02 | Routine reads and simple queries are NOT logged |
| BR-AUD-03 | Only business-critical actions are logged (see AUDIT_ACTION enum) |
| BR-AUD-04 | `old_values` and `new_values` use JSONB for flexible schema |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **2.** FR-AUD-02: Log Attendance Corrections

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-02 |
| **Description** | System shall log attendance corrections after submission |
| **Priority** | P0 |
| **Preconditions** | Attendance session modified |
| **Trigger** | After update |

**Explanation:**

**FR-AUD-02** logs attendance corrections made after a session has been submitted. The log captures both the previous and updated attendance statuses, providing an audit trail for any post-submission changes.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **3.** FR-AUD-03: Log Result Publish/Unpublish

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-03 |
| **Description** | System shall log result publish/unpublish |
| **Priority** | P0 |
| **Preconditions** | Result published/unpublished |
| **Trigger** | After action |

**Explanation:**

**FR-AUD-03** logs result publish and unpublish actions. This provides accountability for when results were made available and who performed the action.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **4.** FR-AUD-04: Log Permission Changes

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-04 |
| **Description** | System shall log permission changes |
| **Priority** | P0 |
| **Preconditions** | Manager permissions updated |
| **Trigger** | After update |

**Explanation:**

**FR-AUD-04** logs all Manager permission changes. When a School Admin updates a Manager's permissions, the old and new permission sets are recorded with the actor's details.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **5.** FR-AUD-05: Log SMS Balance Changes

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-05 |
| **Description** | System shall log SMS balance changes |
| **Priority** | P0 |
| **Preconditions** | Balance adjusted |
| **Trigger** | After adjustment |

**Explanation:**

**FR-AUD-05** logs SMS balance adjustments. When the Platform Admin modifies a tenant's SMS balance, the adjustment amount, previous balance, and new balance are recorded.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **6.** FR-AUD-06: Log Tenant Activation

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-06 |
| **Description** | System shall log tenant activation/deactivation |
| **Priority** | P0 |
| **Preconditions** | Tenant status changed |
| **Trigger** | After change |

**Explanation:**

**FR-AUD-06** logs tenant activation and deactivation events. The log captures who performed the action and the timestamp, providing an audit trail for tenant lifecycle changes.

**No related business rules or open questions.**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **7.** FR-AUD-07: View Tenant Audit Logs

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-07 |
| **Description** | School Admin shall view audit logs for their tenant |
| **Priority** | P0 |
| **Preconditions** | Authenticated |
| **Trigger** | GET /api/v1/audit-logs |

**Explanation:**

**FR-AUD-07** allows School Admin to view audit logs for their own tenant via `GET /api/v1/audit-logs`. Logs are filterable by module, action type, and date range. This provides transparency for internal operations within the school.

**API Endpoint(s):**

##### GET /api/v1/audit-logs

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

**Related Business Rules:**

| ID | Rule |
|----|------|
| BR-AUD-01 | Audit logs are append-only — existing records must never be modified or deleted |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━





#### **8.** FR-AUD-08: View Cross-Tenant Audit Logs

| Property | Value |
|----------|-------|
| **ID** | FR-AUD-08 |
| **Description** | Platform Admin shall view audit logs across all tenants |
| **Priority** | P1 |
| **Preconditions** | Authenticated as Platform Admin |
| **Trigger** | GET /api/v1/admin/audit-logs |

**Explanation:**

**FR-AUD-08** allows Platform Admin to view audit logs across all tenants via `GET /api/v1/admin/audit-logs`. This enables centralized monitoring and compliance auditing across the entire platform.

**API Endpoint(s):**

##### GET /api/v1/admin/audit-logs

View audit logs across all tenants.

**Query params:** `?tenant_id=uuid&action=TENANT_ACTIVATED&from=2026-01-01&to=2026-12-31&page=1&limit=50`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "tenant_id": "uuid",
      "actor_id": "uuid",
      "action": "TENANT_ACTIVATED",
      "entity_type": "tenant",
      "old_values": { "is_active": false },
      "new_values": { "is_active": true },
      "ip_address": "192.168.1.1",
      "created_at": "2026-07-10T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "total": 128
  }
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 403 | `FORBIDDEN` | Only Platform Admin can access |

**No related business rules or open questions.**

---

#### **12.** FR-PLT-12: Platform Analytics

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-12 |
| **Description** | Platform Admin shall view usage analytics across all tenants |
| **Priority** | P1 |
| **Preconditions** | Authenticated as Platform Admin |
| **Trigger** | GET /api/v1/admin/analytics |

**Explanation:**

**FR-PLT-12** provides the Platform Admin with cross-tenant usage analytics via `GET /api/v1/admin/analytics`. This includes aggregate metrics such as total active students, total Managers, attendance rates, and result publication counts across all tenants. This fulfills the "Platform Analytics" responsibility listed in the PRD.

**API Endpoint(s):**

##### GET /api/v1/admin/analytics

**Query params:** `?from_date=2026-01-01&to_date=2026-12-31`

**Response `200 OK`:**
```json
{
  "total_students": 8750,
  "active_students": 8200,
  "total_managers": 180,
  "average_attendance_rate": 94.5,
  "total_exams_created": 420,
  "total_results_published": 380,
  "by_tenant": [
    {
      "tenant_id": "uuid",
      "tenant_name": "Sunrise School",
      "students": 350,
      "attendance_rate": 95.2
    }
  ]
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 403 | `FORBIDDEN` | Only Platform Admin can access |

**No related business rules or open questions.**

---

#### **13.** FR-PLT-13: Platform Settings

| Property | Value |
|----------|-------|
| **ID** | FR-PLT-13 |
| **Description** | Platform Admin shall update platform-wide settings |
| **Priority** | P1 |
| **Preconditions** | Authenticated as Platform Admin |
| **Trigger** | PATCH /api/v1/admin/settings |

**Explanation:**

**FR-PLT-13** allows the Platform Admin to update platform-wide settings via `PATCH /api/v1/admin/settings`. Settings include centralized SMS/Email gateway credentials, platform branding (name, logo), default SMS quota for new tenants, and global feature flags. This fulfills the "Platform Settings" responsibility listed in the PRD. Platform settings are stored separately from tenant-level settings (FR-SET-02).

**API Endpoint(s):**

##### PATCH /api/v1/admin/settings

**Request:**
```json
{
  "platform_name": "SiriusSkool",
  "logo_url": "https://...",
  "default_sms_quota": 500,
  "smtp_host": "smtp.example.com",
  "smtp_port": 587,
  "sms_provider_api_key": "encrypted-key"
}
```

**Response `200 OK`:**
```json
{
  "message": "Platform settings updated"
}
```

**Errors:**

| Status | Code | Condition |
|--------|------|-----------|
| 403 | `FORBIDDEN` | Only Platform Admin can modify platform settings |

**No related business rules or open questions.**

---

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

### A. Complete API Endpoint Reference

| Method | Path | Module | Primary FR |
|--------|------|--------|------------|
| POST | /api/v1/admin/tenants | Platform | FR-PLT-01 |
| GET | /api/v1/admin/tenants | Platform | FR-PLT-08 |
| PATCH | /api/v1/admin/tenants/{id} | Platform | FR-PLT-06 |
| PATCH | /api/v1/admin/tenants/{id}/sms-balance | Platform | FR-PLT-09 |
| GET | /api/v1/admin/notifications | Platform | FR-PLT-10 |
| PATCH | /api/v1/admin/users/{id}/reset-password | Platform | FR-PLT-11 |
| GET | /api/v1/admin/analytics | Platform | FR-PLT-12 |
| PATCH | /api/v1/admin/settings | Platform | FR-PLT-13 |
| POST | /api/v1/admin/auth/login | Auth | FR-AUTH-01 |
| POST | /api/v1/auth/login | Auth | FR-AUTH-02 |
| POST | /api/v1/auth/logout | Auth | FR-AUTH-06 |
| POST | /api/v1/auth/refresh | Auth | FR-AUTH-07 |
| POST | /api/v1/auth/change-password | Auth | FR-AUTH-08 |
| GET | /api/v1/auth/me | Auth | FR-AUTH-09 |
| POST | /api/v1/academic-years | Academic | FR-ACA-01 |
| GET | /api/v1/academic-years | Academic | FR-ACA-09 |
| PATCH | /api/v1/academic-years/{id} | Academic | FR-ACA-08 |
| POST | /api/v1/classes | Academic | FR-ACA-04 |
| POST | /api/v1/classes/{classId}/shifts | Academic | FR-ACA-05 |
| GET | /api/v1/classes/{classId}/shifts | Academic | FR-ACA-09 |
| POST | /api/v1/shifts/{shiftId}/sections | Academic | FR-ACA-06 |
| POST | /api/v1/classes/{classId}/subjects | Academic | FR-ACA-07 |
| GET | /api/v1/settings | Settings | FR-SET-01 |
| PATCH | /api/v1/settings | Settings | FR-SET-02 |
| POST | /api/v1/managers | Permissions | FR-UP-01 |
| GET | /api/v1/managers | Permissions | FR-UP-03 |
| PATCH | /api/v1/managers/{id} | Permissions | FR-UP-02 |
| PUT | /api/v1/managers/{id}/permissions | Permissions | FR-UP-04 |
| GET | /api/v1/permissions/available-modules | Permissions | FR-UP-05 |
| PATCH | /api/v1/managers/{id}/reset-password | Permissions | FR-UP-07 |
| PATCH | /api/v1/managers/{id}/profile | Permissions | FR-UP-08 |
| GET | /api/v1/tenant-modules | Module Mgmt | FR-MM-01 |
| PATCH | /api/v1/tenant-modules/{module} | Module Mgmt | FR-MM-02 |
| POST | /api/v1/applications | Student | FR-STU-01 |
| GET | /api/v1/applications | Student | FR-STU-03 |
| POST | /api/v1/applications/{id}/approve | Student | FR-STU-04 |
| POST | /api/v1/applications/{id}/reject | Student | FR-STU-05 |
| GET | /api/v1/students | Student | FR-STU-08/09 |
| GET | /api/v1/students/{id} | Student | FR-STU-08 |
| POST | /api/v1/students | Student | FR-STU-08 |
| PATCH | /api/v1/students/{id} | Student | FR-STU-08 |
| DELETE | /api/v1/students/{id} | Student | FR-STU-08 |
| POST | /api/v1/students/promote | Student | FR-STU-10 |
| POST | /api/v1/students/{id}/outcome | Student | FR-STU-11 |
| POST | /api/v1/students/import | Student | FR-STU-12 |
| GET | /api/v1/students/import/template | Student | FR-STU-12 |
| GET | /api/v1/students/export | Student | FR-STU-13 |
| GET | /api/v1/attendance/sessions/init | Attendance | FR-ATT-01 |
| POST | /api/v1/attendance/sessions | Attendance | FR-ATT-03 |
| POST | /api/v1/attendance/sessions/{id}/submit | Attendance | FR-ATT-04 |
| PATCH | /api/v1/attendance/sessions/{id}/records | Attendance | FR-ATT-05 |
| POST | /api/v1/attendance/sessions/{id}/send-sms | Attendance | FR-ATT-06 |
| GET | /api/v1/attendance/reports | Attendance | FR-ATT-07 |
| POST | /api/v1/exams | Results | FR-RES-01 |
| POST | /api/v1/exams/{id}/subjects | Results | FR-RES-02 |
| PUT | /api/v1/exams/{id}/marks | Results | FR-RES-04 |
| POST | /api/v1/exams/{id}/publish | Results | FR-RES-06 |
| POST | /api/v1/exams/{id}/unpublish | Results | FR-RES-07 |
| GET | /api/v1/exams/{id}/rank-list | Results | FR-RES-08 |
| GET | /api/v1/exams/{id}/results | Results | FR-RES-09 |
| POST | /api/v1/grade-scales | Results | FR-RES-03 |
| POST | /api/v1/notices | Notice Board | FR-NTC-01 |
| GET | /api/v1/notices | Notice Board | FR-NTC-05 |
| POST | /api/v1/notices/{id}/archive | Notice Board | FR-NTC-04 |
| POST | /api/v1/notifications/send | Notifications | FR-NOT-04 |
| GET | /api/v1/notifications | Notifications | FR-NOT-07 |
| GET | /api/v1/reports/students/{id}/progress | Reports | FR-RPT-01 |
| GET | /api/v1/reports/students/{id}/tc | Reports | FR-RPT-02 |
| GET | /api/v1/reports/attendance | Reports | FR-RPT-03 |
| GET | /api/v1/reports/results | Reports | FR-RPT-04 |
| GET | /api/v1/reports/load | Reports | FR-RPT-06 |
| GET | /api/v1/dashboard/school-admin | Dashboard | FR-DSH-01 |
| GET | /api/v1/dashboard/manager | Dashboard | FR-DSH-02 |
| GET | /api/v1/admin/dashboard | Dashboard | FR-DSH-04 |
| GET | /api/v1/audit-logs | Audit | FR-AUD-07 |
| GET | /api/v1/admin/audit-logs | Audit | FR-AUD-08 |
| DELETE | /api/v1/notices/{id} | Notice Board | FR-NTC-07 |

| PATCH | /api/v1/settings/notifications | Settings | FR-NOT-09 |
| POST | /api/v1/academic-years/{id}/close | Academic | FR-ACA-10 |
| POST | /api/v1/academic-years/{id}/reopen | Academic | FR-ACA-10 |
### B. Complete Business Rules

| ID | Rule | Module |
|----|------|--------|
| BR-PLT-01 | IF tenant is created THEN School Admin user, settings record, and module assignments are created in the same transaction | Platform |
| BR-PLT-02 | IF tenant.is_active = false THEN login proceeds but tenant.is_active=false is returned in the login response. All API operations return 403 TENANT_INACTIVE. | Platform |
| BR-PLT-03 | IF subdomain already exists THEN return 409 SUBDOMAIN_TAKEN | Platform |
| BR-PLT-04 | IF SMS balance adjustment would go below 0 THEN return 400 | Platform |
| BR-PLT-05 | Platform Admin cannot directly modify tenant operational data (students, attendance, results) | Platform |
| BR-AUTH-01 | IF user logs in via admin.sirius-skool.com THEN authenticate against platform_admins | Auth |
| BR-AUTH-02 | IF user logs in via {tenant}.sirius-skool.com THEN authenticate against users scoped to that tenant | Auth |
| BR-AUTH-03 | IF user.is_active = false THEN return 403 ACCOUNT_INACTIVE | Auth |
| BR-AUTH-04 | IF tenant.is_active = false THEN login proceeds but returns tenant.is_active=false in the response. All subsequent API requests return 403 TENANT_INACTIVE. | Auth |
| BR-AUTH-05 | IF user role is MANAGER THEN change-password returns 403. IF SCHOOL_ADMIN or PLATFORM_ADMIN THEN current_password must match stored hash | Auth |
| BR-AUTH-06 | IF revoked refresh token is presented THEN revoke ALL refresh tokens for that user (reuse detection) | Auth |
| BR-AUTH-07 | IF login fails >5 times in 1 minute from same IP THEN rate-limit (429) | Auth |
| BR-AUTH-08 | IF Platform Admin calls reset-password THEN password_hash updated and token_version incremented. IF caller is not Platform Admin THEN 403 | Auth |
| BR-AUTH-09 | IF School Admin calls Manager reset-password THEN Manager's password_hash updated and token_version incremented. IF caller != same tenant School Admin THEN 403 | Auth |
| BR-AUTH-10 | IF user calls POST /api/v1/auth/logout THEN the provided refresh token is marked as revoked | Auth |
| BR-AUTH-11 | Access token valid for 15 min. Refresh token valid for 7 days. | Auth |
| BR-AUTH-12 | IF token_version in JWT doesn't match DB THEN reject with 401 | Auth |
| BR-AUTH-13 | IF new password doesn't meet policy (min 8 chars, uppercase, lowercase, digit) THEN 422 | Auth |
| BR-AUTH-14 | IF tenant user exists in Tenant A THEN cannot authenticate via Tenant B's subdomain | Auth |
| BR-ACA-01 | A tenant can have only one current academic year at a time | Academic |
| BR-ACA-02 | Academic year names must be unique per tenant | Academic |
| BR-ACA-03 | Academic year date ranges must not overlap | Academic |
| BR-ACA-04 | Classes, shifts, sections, and subjects use is_active to disable (never delete) | Academic |
| BR-ACA-05 | A section belongs to exactly one shift | Academic |
| BR-ACA-06 | A subject belongs to exactly one class | Academic |
| BR-ACA-07 | Subject codes must be unique within a class | Academic |
| BR-ACA-08 | Setting a new current academic year resets `current_student_sequence` to `starting_sequence` | Academic |
| BR-ACA-09 | A closed academic year is read-only — no data edits allowed, only reports viewing | Academic |
| BR-ACA-10 | Closing an academic year requires all attendance and results for that year to be finalized | Academic |
| BR-ACA-11 | A shift belongs to exactly one class | Academic |
| BR-SET-01 | Only School Admin can modify settings | Settings |
| BR-SET-02 | Every tenant has exactly one settings record | Settings |
| BR-UP-01 | Only users with role MANAGER can have permission records | Permissions |
| BR-UP-02 | School Admin automatically has full access to all enabled modules | Permissions |
| BR-UP-03 | School Admin can only delegate modules that are in tenant_modules for their tenant | Permissions |
| BR-UP-04 | IF a module is disabled in tenant_modules THEN all related permissions become inactive | Permissions |
| BR-UP-05 | IF a module is not in the Manager's granted list THEN access to that module is denied | Permissions |
| BR-UP-06 | Deactivating a Manager increments token_version to invalidate all sessions | Permissions |
| BR-MM-01 | Authentication and Settings modules cannot be disabled | Module Mgmt |
| BR-MM-02 | School Admin cannot enable a module not assigned by Platform Admin | Module Mgmt |
| BR-MM-03 | Disabling a module hides it for the entire tenant (all roles) | Module Mgmt |
| BR-MM-04 | Re-enabling restores access to existing data | Module Mgmt |
| BR-STU-01 | Registration number format: YY + sequence per tenant (resets each academic year) | Student |
| BR-STU-02 | Registration number sequence is generated atomically using SELECT ... FOR UPDATE | Student |
| BR-STU-03 | Registration number is permanent and never changes | Student |
| BR-STU-04 | Roll numbers are unique per (section + academic year) | Student |
| BR-STU-05 | A student can have only one enrollment per academic year | Student |
| BR-STU-06 | Approving an application creates student + enrollment in a single transaction | Student |
| BR-STU-07 | StudentStatus enum: ACTIVE, DROPPED, TRANSFERRED, GRADUATED | Student |
| BR-STU-08 | EnrollmentStatus enum: ACTIVE, COMPLETED, PROMOTED, REPEATED | Student |
| BR-STU-09 | Applications in PENDING status for 30+ days are automatically deleted | Student |
| BR-ATT-01 | A section can have only one attendance session per date per academic year | Attendance |
| BR-ATT-02 | Attendance defaults to PRESENT — only absentees are explicitly marked | Attendance |
| BR-ATT-03 | Attendance corrections after submission are logged in audit_logs | Attendance |
| BR-ATT-04 | Each SMS sent consumes 1 credit from tenants.sms_balance | Attendance |
| BR-ATT-05 | Only today's and future dates can have attendance taken (configurable backdate limit) | Attendance |
| BR-ATT-06 | Attendance editing is restricted to a configurable window (e.g., same day only) | Attendance |
| BR-RES-01 | A class cannot have two exams with the same name in the same academic year | Results |
| BR-RES-02 | Marks can only be entered for the current academic year | Results |
| BR-RES-03 | Publish is exclusive to School Admin role | Results |
| BR-RES-04 | IF marks are published THEN editing requires unpublish first | Results |
| BR-RES-05 | Grades/GPA are calculated at runtime — never stored in marks table | Results |
| BR-RES-06 | IF is_absent = true THEN obtained_marks must be null | Results |
| BR-NTC-01 | Published notices cannot be edited (must archive and recreate) | Notice Board |
| BR-NTC-02 | Only PDF and IMAGE file types are supported for notice attachments | Notice Board |
| BR-NTC-03 | All authenticated users in the tenant can view published notices | Notice Board |
| BR-NTC-04 | IF expires_at is set and the date has passed THEN notice is hidden | Notice Board |
| BR-NOT-01 | Notification failures must NOT roll back the source transaction | Notifications |
| BR-NOT-02 | Each SMS deducts 1 credit from tenants.sms_balance | Notifications |
| BR-NOT-03 | IF sms_balance = 0 THEN SMS is blocked (email still works) | Notifications |
| BR-NOT-04 | Notification logs are immutable once written | Notifications |
| BR-NOT-05 | All notifications go through the platform's centralized gateway | Notifications |
| BR-RPT-01 | Reports include school branding from tenant_settings | Reports |
| BR-RPT-02 | Reports are generated for current academic year by default | Reports |
| BR-RPT-03 | Data is read-only — reports never modify data | Reports |
| BR-RPT-04 | Reports only include data belonging to the current tenant | Reports |
| BR-DSH-01 | Dashboard data is scoped to the current academic year | Dashboard |
| BR-DSH-02 | Managers only see metrics for modules they have View permission on | Dashboard |
| BR-DSH-03 | Dashboard fetches real-time data (no caching for MVP) | Dashboard |
| BR-AUD-01 | Audit logs are append-only — existing records must never be modified or deleted | Audit |
| BR-AUD-02 | Routine reads and simple queries are NOT logged | Audit |
| BR-AUD-03 | Only business-critical actions are logged | Audit |
| BR-AUD-04 | old_values and new_values use JSONB for flexible schema | Audit |

### C. Requirements Traceability Matrix

| Module | FR IDs | PRD § | DB Tables |
|--------|--------|-------|-----------|
| Platform & Multi-Tenant | FR-PLT-01 → FR-PLT-13 | §4, §5, §9 | tenants, tenant_settings, tenant_modules |
| Authentication | FR-AUTH-01 → FR-AUTH-12 | §6.1, §10 | platform_admins, users |
| Academic Structure | FR-ACA-01 → FR-ACA-10 | §6.9, §7.3 | academic_years, classes, shifts, sections, subjects |
| Settings | FR-SET-01 → FR-SET-03 | §6.9 | tenant_settings |
| User & Permission Mgmt | FR-UP-01 → FR-UP-08 | §7.1, §8 | users, manager_permissions |
| Module Management | FR-MM-01 → FR-MM-03 | §7.2 | tenant_modules |
| Student Management | FR-STU-01 → FR-STU-14 | §6.3, §9 | applications, students, student_enrollments |
| Attendance | FR-ATT-01 → FR-ATT-07 | §6.4 | attendance_sessions, attendance_records |
| Results | FR-RES-01 → FR-RES-09 | §6.5 | exams, exam_subjects, marks, grade_scales |
| Notice Board | FR-NTC-01 → FR-NTC-07 | §6.6 | notices |
| Notifications | FR-NOT-01 → FR-NOT-02, FR-NOT-04 → FR-NOT-09 | §6.7 | notifications |
| Reports | FR-RPT-01 → FR-RPT-06 | §6.8 | (read-only) |
| Dashboard | FR-DSH-01 → FR-DSH-04 | §6.2 | (aggregated) |
| Audit Logging | FR-AUD-01 → FR-AUD-08 | §10 | audit_logs |

### D. Error Code Reference

| Code | HTTP Status | Description | Used By |
|------|-------------|-------------|---------|
| `INVALID_CREDENTIALS` | 401 | Wrong email or password | Auth |
| `ACCOUNT_INACTIVE` | 403 | User deactivated | Auth |
| `TENANT_INACTIVE` | 403 | Tenant deactivated — all API operations blocked | Platform |
| `INVALID_REFRESH_TOKEN` | 401 | Refresh token not found or revoked | Auth |
| `TOKEN_EXPIRED` | 401 | Token has expired | Auth |
| `TOKEN_REUSE_DETECTED` | 401 | Revoked token reused — all tokens revoked | Auth |
| `INCORRECT_CURRENT_PASSWORD` | 400 | Wrong current password | Auth |
| `SAME_PASSWORD` | 400 | New password same as current | Auth |
| `MANAGER_CANNOT_CHANGE_PASSWORD` | 403 | Manager role cannot change password | Auth |
| `USER_NOT_FOUND` | 404 | User not found | Platform |
| `MANAGER_NOT_FOUND` | 404 | Manager not found | User Mgmt |
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

### E. Open Questions Log

| ID | Module | Question | Status |
|----|--------|----------|--------|
| Q-PLT-01 | Platform | Should tenant deletion be implemented in MVP? | Closed: Only deactivation, never delete |
| Q-PLT-02 | Platform | What is the default module set for new tenants? | Closed: ALL checked marked while creating tanent |
| Q-AUTH-01 | Auth | Access token expiry configurable? | Closed: Hardcoded 15 min for MVP |
| Q-AUTH-02 | Auth | Refresh token storage — add refresh_tokens model to Prisma, or use signed JWTs without DB lookup? | Closed: Access and Refresh token workflow |
| Q-AUTH-03 | Auth | Single session per user enforced in MVP? | Closed: Optional — token_version mechanism supports future enforcement, not active in MVP |
| Q-ACA-01 | Academic | Close endpoint or just is_current toggle? | OPEN |
| Q-ACA-02 | Academic | Verify no pending data before closing year? | Closed: Saved data as it is in current DB |
| Q-SET-01 | Settings | Where are notification templates stored? | Open |
| Q-SET-02 | Settings | Password policy configurable or hardcoded? | Closed: Hardcoded in validation zod |
| Q-UP-01 | Permissions | Can School Admin edit Manager email/password? | Closed: YES |
| Q-UP-02 | Permissions | can_print separate from can_export in UI? | Closed: Print/Export are Admin-only actions. Managers use module-level toggle with predefined action sets (V/C/E). |
| Q-STU-01 | Student | Admission form public or authenticated? | Closed: A public endpoint for public users |
| Q-STU-02 | Student | Application number format? | Closed: As it is in PRD  |
| Q-STU-03 | Student | CAPTCHA on public admission form? | Closed: NO captcha, We can use rate limitter |
| Q-STU-04 | Student | Reactivate TRANSFERRED or GRADUATED students? | Closed: Can reactive  |
| Q-ATT-01 | Attendance | Attendance status granularity? | Closed: only Present and Absent |
| Q-ATT-02 | Attendance | Month closure in MVP? | Closed: Configurable toggle in Settings (FR-SET-02) — if enabled, past month attendance is read-only |
| Q-ATT-03 | Attendance | Backdate limit value? | OPEN |
| Q-RES-01 | Results | Grid view or one-by-one marks entry? | OPEN |
| Q-RES-02 | Results | Multiple managers per exam subject-wise? | Closed: There is no restriction or defined manager, any manager or multiple manager can entry or edit. |
| Q-RES-03 | Results | Grade scale per tenant or per year? | Closed: Per tenant  |
| Q-NOT-01 | Notifications | SMS/Email provider in MVP? | OPEN |
| Q-NOT-02 | Notifications | Customizable notification templates? | Closed: Yes, from the setting of notification templates |
| Q-RPT-01 | Reports | PDF generation library? | OPEN |
| Q-RPT-03 | Reports | Sync or async report generation? | Open |

---

*End of SRS*

