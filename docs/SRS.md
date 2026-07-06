# Sirius-Skool — Software Requirements Specification

> **Version:** 0.2 (Draft)  
> **Status:** In Progress  
> **Date:** July 6, 2026  
> **Based on:** PRD v1.0 (`docs/PRD.md`)

---

## Table of Contents

1. [Introduction](#1-introduction)
   - 1.1 Purpose
   - 1.2 Scope
   - 1.3 Definitions & Acronyms
   - 1.4 References
   - 1.5 Tech Stack
2. [Overall Description](#2-overall-description)
   - 2.1 User Roles & Hierarchy
   - 2.2 Assumptions & Dependencies
3. [Module: Authentication](#3-module-authentication)
   - 3.1 Overview
   - 3.2 Use Cases
   - 3.3 Functional Requirements
   - 3.4 Data Model
   - 3.5 API Endpoints
   - 3.6 Business Rules
   - 3.7 Error Handling
   - 3.8 Security Considerations
   - 3.9 Acceptance Criteria

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification (SRS) defines the detailed functional and non-functional requirements for the Sirius-Skool School Management System. It is intended to serve as a technical reference for developers, testers, and stakeholders during implementation.

This document supplements the Product Requirements Document (`docs/PRD.md`). Where this SRS is silent, the PRD takes precedence.

### 1.2 Scope

This SRS covers all MVP modules as defined in the PRD. The Authentication module is documented in full detail below. Remaining modules are listed as in-scope and will be detailed in subsequent SRS iterations.

**In-scope for this version:**
- Authentication module (detailed)
- Dashboard, Student Management & Admission, Attendance Management, Result Management, Notice Board, Notification System, Reports, Settings modules (outline — to be detailed in next SRS iterations)
- User & Permission Management, Module Management, Academic Year Management (cross-cutting — to be detailed)
- Overall system data model conventions

**Out-of-scope for this version:**
- UI/UX wireframes (separate design phase)
- Deployment architecture

### 1.3 Definitions & Acronyms

| Term | Definition |
|---|---|
| **Tenant** | A single school/organization with isolated data |
| **Platform Admin** | Super Admin managing all tenants |
| **School Admin** | Primary admin for a single tenant |
| **Manager** | Staff member with action-level permissions |
| **JWT** | JSON Web Token |
| **Access Token** | Short-lived JWT for API authentication |
| **Refresh Token** | Long-lived token to obtain new access tokens |
| **SRS** | Software Requirements Specification |
| **PRD** | Product Requirements Document |

### 1.4 References

| Document | Location |
|---|---|
| Product Requirements Document (PRD) | `docs/PRD.md` |
| PRD Review & Issues | `docs/Issues/PRD-Issues.md` |

### 1.5 Tech Stack

> **Note:** Tech stack decisions are pending and will be documented here once finalized.

---

## 2. Overall Description

### 2.1 User Roles & Hierarchy

As defined in PRD §4:

```
Platform (SaaS Owner)
└── Platform Admin (Super Admin)
      └── Tenant (School)
            └── School Admin
                  └── Manager 1..n
```

| Role | Scope | Access Model |
|---|---|---|
| **Platform Admin** | All tenants | Full platform control |
| **School Admin** | Single tenant | Module-level (full CRUD + Export on assigned modules) |
| **Manager** | Single tenant | Action-level (View/Create/Edit/Delete/Export per module) |

### 2.2 Assumptions & Dependencies

1. Each tenant is identified by a unique subdomain (e.g., `schoolname.sirius-skool.com`)
2. Platform Admin accesses the system via a separate URL (`admin.sirius-skool.com`)
3. Email delivery service is available for password reset and notifications
4. A wildcard SSL certificate covers `*.sirius-skool.com`
5. The system runs in a distributed environment with Redis available for session/token management

---

## 3. Module: Authentication

### 3.1 Overview

The Authentication module provides secure, role-based access to the Sirius-Skool system. It supports tenant-scoped login, JWT-based session management, and password recovery workflows. Platform Admin, School Admin, and Manager all authenticate through this module, with role-based routing after login.

### 3.2 Use Cases

| ID | Use Case | Actor | Description |
|---|---|---|---|
| UC-AUTH-01 | Login | All users | Authenticate with email/username + password |
| UC-AUTH-02 | Logout | All users | End active session |
| UC-AUTH-03 | Refresh Token | All users | Obtain a new access token using refresh token |
| UC-AUTH-04 | Forgot Password | All users | Request a password reset link via email |
| UC-AUTH-05 | Reset Password | All users | Set a new password using a reset token |
| UC-AUTH-06 | Change Password | All authenticated users | Change current password (requires old password) |
| UC-AUTH-07 | View Current User | All authenticated users | Get current user profile and permissions |
| UC-AUTH-08 | Tenant Detection | All users | Identify tenant from subdomain at login |

### 3.3 Functional Requirements

#### FR-AUTH-01: Login

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-01 |
| **Description** | System shall authenticate a user using email/username and password |
| **Trigger** | User submits login form |
| **Preconditions** | Tenant is active; user account is active |
| **Postconditions** | Access token and refresh token are issued; login attempt is logged |
| **Priority** | P0 |

**Input validation:**

| Field | Type | Required | Constraints |
|---|---|---|---|
| `email` or `username` | string | Yes | Must not be empty; trimmed |
| `password` | string | Yes | Must not be empty; minimum 6 characters |

**Success response:**

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<uuid>",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": {
    "id": 1,
    "email": "admin@school.com",
    "name": "John Doe",
    "role": "school_admin",
    "tenant_id": 1
  }
}
```

#### FR-AUTH-02: Tenant Detection

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-02 |
| **Description** | System shall identify the tenant from the request subdomain before authentication |
| **Details** | Login page reads the subdomain from the URL. On `admin.sirius-skool.com`, route to Platform Admin login. On `{tenant}.sirius-skool.com`, resolve tenant and scope authentication to that tenant |
| **Priority** | P0 |

#### FR-AUTH-03: Subdomain Extraction & Fallback

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-03 |
| **Description** | System shall extract tenant subdomain from the `Host` header and provide a fallback mechanism for local development |
| **Details** | In production: parse `Host` header for subdomain. In development: accept a custom header (`X-Tenant-Slug`) or a query parameter (`?tenant=schoolname`) |
| **Priority** | P0 |

#### FR-AUTH-04: Role-Based Redirect

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-04 |
| **Description** | System shall redirect users to their role-specific dashboard after login |
| **Details** | Platform Admin → `/admin/dashboard`; School Admin → `/school/dashboard`; Manager → `/manager/dashboard` |
| **Priority** | P0 |

#### FR-AUTH-05: Logout

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-05 |
| **Description** | System shall invalidate the current session by revoking the refresh token |
| **Details** | Client sends refresh token; system marks it as revoked in the database; access token expiry is relied upon (no server-side access token revocation for MVP) |
| **Priority** | P0 |

#### FR-AUTH-06: Token Refresh

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-06 |
| **Description** | System shall issue a new access token when a valid, non-expired, non-revoked refresh token is presented |
| **Priority** | P0 |

**Token rotation:** The old refresh token is revoked and a new refresh token is issued on each refresh (rotation). If a revoked refresh token is reused, all refresh tokens for that user are revoked (reuse detection).

#### FR-AUTH-07: Forgot Password

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-07 |
| **Description** | System shall send a password reset email with a unique, time-limited token |
| **Priority** | P0 |

**Behavior:**
- Accept email address (must belong to an active account)
- Generate cryptographically secure random token
- Store hashed token in `password_resets` table with 1-hour expiry
- Send email with reset link: `https://{tenant}.sirius-skool.com/auth/reset-password?token={token}`
- Return success response regardless of whether the email exists (prevents email enumeration)

#### FR-AUTH-08: Reset Password

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-08 |
| **Description** | System shall allow password reset using a valid, non-expired, non-used reset token |
| **Priority** | P0 |

**Validation:**
- Token must exist, not expired, not used
- New password must meet password policy (min length, complexity)
- On success: mark token as used, update password hash, revoke all refresh tokens for the user

#### FR-AUTH-09: Change Password

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-09 |
| **Description** | System shall allow authenticated users to change their password by providing current password + new password |
| **Priority** | P0 |

**Validation:**
- Current password must match stored hash
- New password must meet password policy
- New password must differ from current password
- On success: revoke all refresh tokens except current session

#### FR-AUTH-10: Get Current User

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-10 |
| **Description** | System shall return the authenticated user's profile and module permissions |
| **Priority** | P0 |

**Response includes:**
- User details (id, name, email, role)
- Tenant details (id, name, slug)
- Module permissions (list of modules with assigned actions)

#### FR-AUTH-11: Inactive/Deactivated User Block

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-11 |
| **Description** | System shall prevent login for inactive or deactivated user accounts |
| **Priority** | P0 |

#### FR-AUTH-12: Inactive/Deactivated Tenant Block

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-12 |
| **Description** | System shall prevent login for users belonging to an inactive or deactivated tenant |
| **Priority** | P0 |

#### FR-AUTH-13: Rate Limiting

| Field | Detail |
|---|---|
| **ID** | FR-AUTH-13 |
| **Description** | System shall rate-limit login attempts per IP address (5 failed attempts per minute) |
| **Priority** | P1 |

### 3.4 Data Model

#### Table: `users`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGINT UNSIGNED | PK, AUTO_INCREMENT | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK → tenants.id, NOT NULL | Tenant scope (NULL for Platform Admin) |
| `email` | VARCHAR(255) | UNIQUE(tenant_id, email), NOT NULL | Login identifier |
| `username` | VARCHAR(100) | UNIQUE(tenant_id, username), NULLABLE | Alternative login identifier |
| `password_hash` | VARCHAR(255) | NOT NULL | bcrypt/Argon2 hash |
| `name` | VARCHAR(255) | NOT NULL | Display name |
| `role` | ENUM('platform_admin', 'school_admin', 'manager') | NOT NULL | User role |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Account active status |
| `token_version` | INTEGER | NOT NULL, DEFAULT 0 | Incremented on deactivation to invalidate all sessions |
| `last_login_at` | TIMESTAMP | NULLABLE | Last successful login timestamp |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

**Indexes:**
- `UNIQUE(tenant_id, email)` — unique email per tenant
- `UNIQUE(tenant_id, username)` — unique username per tenant
- `INDEX(tenant_id)` — for tenant-scoped queries
- `INDEX(email)` — for forgot-password lookup

#### Table: `refresh_tokens`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGINT UNSIGNED | PK, AUTO_INCREMENT | Primary key |
| `user_id` | BIGINT UNSIGNED | FK → users.id, NOT NULL | Owner |
| `token_hash` | VARCHAR(255) | NOT NULL, UNIQUE | SHA-256 hash of the refresh token |
| `expires_at` | TIMESTAMP | NOT NULL | Expiry timestamp (7 days from issue) |
| `revoked_at` | TIMESTAMP | NULLABLE | When revoked (NULL = active) |
| `replaced_by` | BIGINT UNSIGNED | FK → refresh_tokens.id, NULLABLE | Token that replaced this one (rotation chain) |
| `ip_address` | VARCHAR(45) | NULLABLE | IP address of requester |
| `user_agent` | VARCHAR(500) | NULLABLE | User agent of requester |
| `created_at` | TIMESTAMP | NOT NULL | |

**Indexes:**
- `UNIQUE(token_hash)` — for lookup by token
- `INDEX(user_id)` — to revoke all tokens for a user
- `INDEX(expires_at)` — to clean up expired tokens

#### Table: `password_resets`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | BIGINT UNSIGNED | PK, AUTO_INCREMENT | Primary key |
| `tenant_id` | BIGINT UNSIGNED | FK → tenants.id, NULLABLE | Tenant scope (NULL for Platform Admin) |
| `email` | VARCHAR(255) | NOT NULL | Email requesting reset |
| `token_hash` | VARCHAR(255) | NOT NULL, UNIQUE | SHA-256 hash of reset token |
| `expires_at` | TIMESTAMP | NOT NULL | Expiry (1 hour from creation) |
| `used_at` | TIMESTAMP | NULLABLE | When token was used |
| `created_at` | TIMESTAMP | NOT NULL | |

**Indexes:**
- `UNIQUE(token_hash)` — for lookup by token
- `INDEX(email, tenant_id)` — to find existing resets

### 3.5 API Endpoints

All endpoints return JSON. Authentication-required endpoints use `Authorization: Bearer <access_token>` header.

#### POST /api/auth/login

Authenticate user credentials and issue tokens.

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `login` | string | Yes | Email or username |
| `password` | string | Yes | Min 6 characters |
| `tenant_slug` | string | No | Explicit tenant identifier (fallback) |

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "550e8400-e29b-41d4-a716-446655440000",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": {
    "id": 1,
    "email": "admin@school.com",
    "name": "John Doe",
    "role": "school_admin",
    "tenant_id": 1
  }
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `INVALID_CREDENTIALS` | Email/username or password is incorrect |
| 403 | `ACCOUNT_INACTIVE` | User account is deactivated |
| 403 | `TENANT_INACTIVE` | Tenant is inactive |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many login attempts |

#### POST /api/auth/logout

Revoke the current refresh token.

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `refresh_token` | string | Yes | Must be a valid UUID |

**Response `200 OK`:**
```json
{ "message": "Logged out successfully" }
```

#### POST /api/auth/refresh

Issue a new access token using a refresh token.

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `refresh_token` | string | Yes | Must be a valid UUID |

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "660e8400-e29b-41d4-a716-446655440001",
  "token_type": "Bearer",
  "expires_in": 900
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `INVALID_REFRESH_TOKEN` | Token not found or revoked |
| 401 | `TOKEN_EXPIRED` | Token has expired |
| 401 | `TOKEN_REUSE_DETECTED` | Revoked token was reused — all tokens for user revoked |

#### POST /api/auth/forgot-password

Send a password reset email.

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `email` | string | Yes | Valid email format |

**Response `200 OK`:** Always returns success (prevents email enumeration).
```json
{ "message": "If the email exists, a reset link has been sent" }
```

#### POST /api/auth/reset-password

Reset password using a reset token.

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `token` | string | Yes | Reset token from email |
| `password` | string | Yes | Min 8 characters, must contain uppercase, lowercase, digit |

**Response `200 OK`:**
```json
{ "message": "Password reset successfully" }
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 400 | `INVALID_TOKEN` | Token is invalid |
| 400 | `TOKEN_EXPIRED` | Token has expired |
| 400 | `TOKEN_ALREADY_USED` | Token has already been used |

#### POST /api/auth/change-password

Change password (requires authentication).

**Request headers:** `Authorization: Bearer <access_token>`

**Request body:**

| Field | Type | Required | Validation |
|---|---|---|---|
| `current_password` | string | Yes | Must match current password |
| `new_password` | string | Yes | Min 8 characters, must contain uppercase, lowercase, digit; must differ from current |

**Response `200 OK`:**
```json
{ "message": "Password changed successfully" }
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 400 | `INCORRECT_CURRENT_PASSWORD` | Current password is wrong |
| 400 | `SAME_PASSWORD` | New password is same as current |

#### GET /api/auth/me

Get current authenticated user profile and permissions.

**Request headers:** `Authorization: Bearer <access_token>`

**Response `200 OK`:**
```json
{
  "id": 1,
  "email": "admin@school.com",
  "name": "John Doe",
  "role": "school_admin",
  "tenant": {
    "id": 1,
    "name": "Springfield School",
    "slug": "springfield",
    "is_active": true
  },
  "permissions": {
    "student_management": ["view", "create", "edit", "delete", "export"],
    "attendance": ["view", "take", "edit"]
  }
}
```

### 3.6 Business Rules

| ID | Rule | Source |
|---|---|---|
| BR-AUTH-01 | Platform Admin must log in via `admin.sirius-skool.com` | PRD §6.1 |
| BR-AUTH-02 | School Admin and Managers must log in via their tenant subdomain URL | PRD §6.1 |
| BR-AUTH-03 | A user cannot be logged into multiple tenants simultaneously | PRD §6.1 |
| BR-AUTH-04 | Inactive/deactivated users cannot log in | PRD §6.1, §9 Rule 14 |
| BR-AUTH-05 | Inactive/deactivated tenants block all user access | PRD §9 Rule 3 |
| BR-AUTH-06 | Access tokens expire after 15 minutes (configurable) | PRD §6.1 |
| BR-AUTH-07 | Refresh tokens expire after 7 days (configurable) | — |
| BR-AUTH-08 | Password reset tokens expire after 1 hour | — |
| BR-AUTH-09 | Passwords must be stored using bcrypt or Argon2 | PRD §10 |
| BR-AUTH-10 | Failed login attempts are rate-limited per IP (5/min) | — |
| BR-AUTH-11 | Login attempts are logged for audit (user, IP, timestamp, success/failure) | PRD §7, §10 |
| BR-AUTH-12 | Deactivating a Manager immediately revokes access across all sessions | PRD §9 Rule 14 |

### 3.7 Error Handling

#### Standard Error Response Format

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable description",
    "details": {}
  }
}
```

#### Authentication Error Codes

| Code | HTTP Status | Description |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | Email/username or password is incorrect |
| `ACCOUNT_INACTIVE` | 403 | User account has been deactivated |
| `TENANT_INACTIVE` | 403 | School/tenant has been suspended |
| `INVALID_REFRESH_TOKEN` | 401 | Refresh token not found or already revoked |
| `TOKEN_EXPIRED` | 401 | Token has expired |
| `TOKEN_REUSE_DETECTED` | 401 | Revoked refresh token was reused (security threat) |
| `INVALID_TOKEN` | 400 | Reset token is invalid |
| `TOKEN_ALREADY_USED` | 400 | Reset token has already been used |
| `INCORRECT_CURRENT_PASSWORD` | 400 | Current password provided is incorrect |
| `SAME_PASSWORD` | 400 | New password is identical to current password |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests; retry after the specified time |
| `VALIDATION_ERROR` | 422 | Request body failed validation (details in `details` field) |
| `UNAUTHORIZED` | 401 | Missing or invalid access token |
| `TOKEN_EXPIRED` | 401 | Access token has expired |

#### Validation Error Response (`422`)

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": ["The email field is required"],
      "password": ["The password must be at least 6 characters"]
    }
  }
}
```

#### Rate Limit Response (`429`)

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many login attempts. Please try again in 60 seconds."
  }
}
```

### 3.8 Security Considerations

1. **Password hashing:** bcrypt with cost factor 12 (minimum) or Argon2id
2. **JWT signing:** HS256 with a strong, rotated secret key (minimum 256 bits)
3. **JWT payload:** Must not contain sensitive data (no password hashes, no PII beyond user ID and role)
4. **Refresh token storage:** Store SHA-256 hash in database; never store raw tokens
5. **Token rotation:** Each refresh invalidates the previous refresh token; reuse detection revokes all tokens
6. **HTTPS only:** All endpoints must be served over TLS 1.3
7. **Rate limiting:** Login and forgot-password endpoints are rate-limited per IP
8. **Email enumeration prevention:** Forgot-password returns the same response regardless of whether the email exists
9. **Subdomain validation:** Validate that the subdomain corresponds to an active tenant before authentication
10. **Input sanitization:** All inputs must be sanitized to prevent injection attacks

### 3.9 Acceptance Criteria

| ID | Scenario | Expected Result |
|---|---|---|
| AC-AUTH-01 | User logs in with valid credentials on correct tenant subdomain | 200 with access + refresh tokens; role-based redirect |
| AC-AUTH-02 | User logs in with incorrect password | 401 with `INVALID_CREDENTIALS` |
| AC-AUTH-03 | User logs in with deactivated account | 403 with `ACCOUNT_INACTIVE` |
| AC-AUTH-04 | User logs in on inactive tenant subdomain | 403 with `TENANT_INACTIVE` |
| AC-AUTH-05 | User calls `/me` with valid access token | 200 with user profile + permissions |
| AC-AUTH-06 | User calls `/me` with expired access token | 401 with `TOKEN_EXPIRED` |
| AC-AUTH-07 | User refreshes token with valid refresh token | 200 with new access + refresh tokens |
| AC-AUTH-08 | User refreshes token with revoked refresh token | 401 + all tokens for user revoked |
| AC-AUTH-09 | User requests password reset for existing email | 200; email sent with reset link |
| AC-AUTH-10 | User requests password reset for non-existent email | 200 (same response to prevent enumeration) |
| AC-AUTH-11 | User resets password with valid token | 200; password updated; can login with new password |
| AC-AUTH-12 | User resets password with expired token | 400 with `TOKEN_EXPIRED` |
| AC-AUTH-13 | User changes password with correct current password | 200; password updated; old refresh tokens revoked |
| AC-AUTH-14 | User changes password with incorrect current password | 400 with `INCORRECT_CURRENT_PASSWORD` |
| AC-AUTH-15 | User exceeds 5 failed login attempts in 1 minute | 429 with `RATE_LIMIT_EXCEEDED` |
| AC-AUTH-16 | Platform Admin tries to login via tenant subdomain | 401 (tenant not found for Platform Admin) |


